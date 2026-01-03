---
title: "Kotlin Coroutines on wasmJs: why there is no `runBlocking` (and why “hacks” are risky)"
seoTitle: "Kotlin WasmJS: No `runBlocking` Needed"
seoDescription: "Explains why Kotlin's `runBlocking` is absent for wasmJs, highlighting risks and limitations of implementing blocking in JS/WASM environments"
datePublished: Sat Jan 03 2026 15:02:23 GMT+0000 (Coordinated Universal Time)
cuid: cmjyflyy8000002ldb59qby1i
slug: kotlin-coroutines-on-wasmjs-why-there-is-no-runblocking-and-why-hacks-are-risky
ogImage: https://cdn.hashnode.com/res/hashnode/image/upload/v1767452512315/a51fa2ef-c31c-456b-8bab-1bb848a623b5.png
tags: js, kotlin, wasm, kotlin-multiplatform, kotlin-coroutines

---

## Cheat sheet

- `runBlocking` is a **thread-blocking** bridge.
- It exists only for targets that are built from the **`concurrent`** source set.
- `js/wasm*` are built from **`jsAndWasm*`** source sets, so they **don’t get** `runBlocking`.
- On JS (and wasmJs hosted by JS) you cannot block the single event loop and still let async callbacks run.
- Background reading: [Kt. Academy — Coroutines in other languages](https://kt.academy/article/cc-other-languages)

---

## 1) The build layout makes it explicit

In `kotlinx.coroutines`, the source set graph is documented directly in
[`kotlinx-coroutines-core/build.gradle.kts`](https://github.com/Kotlin/kotlinx.coroutines/blob/master/kotlinx-coroutines-core/build.gradle.kts):

```text
/* ==========================================================================
  Configure source sets structure for kotlinx-coroutines-core:

     TARGETS                            SOURCE SETS
     ------------------------------------------------------------
     wasmJs \------> jsAndWasmJsShared ----+
     js     /                              |
                                           V
     wasmWasi --------------------> jsAndWasmShared ----------+
                                                              |
                                                              V
     jvm ----------------------------> concurrent -------> common
                                        ^
     ios     \                          |
     macos   | ---> nativeDarwin ---> native ---+
     tvos    |                         ^
     watchos /                         |
                                       |
     linux  \  ---> nativeOther -------+
     mingw  /
 ========================================================================== */
```

**Key point:** Targets that support `runBlocking` share the **`concurrent`** source set (`jvm`, `native*`).
Targets like `js`, `wasmJs`, and `wasmWasi` share **`jsAndWasm*`**, where blocking builders are intentionally absent.

---

## 2) Where `runBlocking` actually lives

In [`concurrent/src/Builders.concurrent.kt`](https://github.com/Kotlin/kotlinx.coroutines/blob/master/kotlinx-coroutines-core/concurrent/src/Builders.concurrent.kt)
it is declared as an `expect` function:

```kotlin
public expect fun <T> runBlocking(
    context: CoroutineContext = EmptyCoroutineContext,
    block: suspend CoroutineScope.() -> T
): T
```

Then each “blocking-capable” platform provides its own `actual` implementation.

### Native `runBlocking`: the essential mechanics

See [`native/src/Builders.kt`](https://github.com/Kotlin/kotlinx.coroutines/blob/master/kotlinx-coroutines-core/native/src/Builders.kt) (simplified):

```kotlin
val eventLoop: EventLoop? = ...
val newContext: CoroutineContext = ...
val coroutine = BlockingCoroutine<T>(newContext, eventLoop)

coroutine.start(CoroutineStart.DEFAULT, coroutine, block)
return coroutine.joinBlocking()
```

**What matters:**

1. It selects/creates an **EventLoop** for the blocked thread.
2. It starts the coroutine.
3. It **blocks** the thread in `joinBlocking()` until completion, while pumping the event loop manually.

---

## 3) Minimum about EventLoop (only what we need here)

Blocking builders depend on the ability to **process queued work while the thread is blocked**.

In [`common/src/EventLoop.common.kt`](https://github.com/Kotlin/kotlinx.coroutines/blob/master/kotlinx-coroutines-core/common/src/EventLoop.common.kt):

```kotlin
open fun processNextEvent(): Long {
    if (!processUnconfinedEvent()) return Long.MAX_VALUE
    return 0
}
```

So `runBlocking` is not just “wait”; it is “wait **and** keep progressing coroutine work”.

---

## 4) Why “blocking” is still possible on Android (even with one UI thread)

Even if you think “my Android app is single-threaded”, it still runs on a real **OS thread**.
That thread can be **parked / blocked**, and other threads (timers, I/O, Binder, etc.) can still run.

Also, Android’s main thread is built around a **message loop** (an event loop). The core loop is literally an
infinite loop in `Looper.loop()`.

Sources:
- [`Looper.java` on Android Code Search](https://cs.android.com/android/platform/superproject/main/+/main:frameworks/base/core/java/android/os/Looper.java)
- [`Handler.java` on Android Code Search](https://cs.android.com/android/platform/superproject/main/+/main:frameworks/base/core/java/android/os/Handler.java)

Key fragment from `Looper.loop()`:

```java
for (;;) {
    if (!loopOnce(me, ident, thresholdOverride)) {
        return;
    }
}
```

On Android, “one thread” does **not** mean “no event loop”. It means: “one thread running an event loop”.

**Important:** Calling `runBlocking` on the **main/UI thread** is still a bad idea.
It freezes the Looper, stops message processing, and can lead to ANR (Application Not Responding).

---

## 5) Why this cannot exist on wasmJs

On wasmJs (and Kotlin/JS), coroutine resumptions often come from **JS callbacks**
(Promise microtasks, timers, I/O). These callbacks run only when control returns to the **JS event loop**.

If you block the main thread, you stop the loop, so the coroutine **cannot resume**.

Minimal JS deadlock example:

```javascript
function fakeRunBlocking(promise) {
  let done = false;
  promise.then(() => done = true);

  while (!done) {} // blocks the event loop -> then() never runs
}
```

This is why JS frameworks “wait” by **returning a Promise**, not by blocking a thread.

---

## 6) About “runBlocking for all targets” libraries

There are small community libraries that try to provide `runBlocking` for JS/WASM.
Example: [`com.javiersc.kotlinx:coroutines-run-blocking-all`](https://mvnrepository.com/artifact/com.javiersc.kotlinx/coroutines-run-blocking-all).

One JS/WASM implementation is found in
[`RunBlocking.kt` in run-blocking-kmp](https://github.com/JavierSegoviaCordoba/run-blocking-kmp/blob/main/coroutines-run-blocking-all/js/main/kotlin/com/javiersc/kotlinx/coroutines/run/blocking/RunBlocking.kt) (simplified):

```kotlin
public actual fun <T> runBlocking(
    context: CoroutineContext,
    block: suspend CoroutineScope.() -> T,
): T = GlobalScope.promise(context) { block() } as T
```

### Why this is dangerous

1. **It lies about the return type:** `Promise<T>` is cast to `T`. This breaks type safety.
2. **It returns immediately:** It does not actually wait.
3. **It uses `GlobalScope`:** This breaks *structured concurrency* (harder cancellation, easier leaks).

---

## 7) Your `runBlockingAll`: better idea, but still not a solution

A simplified version of a common attempt to "fix" this:

```kotlin
actual fun <T> runBlockingAll(
    context: CoroutineContext,
    block: suspend CoroutineScope.() -> T
): T {
    var out: Result<T>? = null
    val continuation: Continuation<T> = Continuation(context) { out = it }
    block.startCoroutine(receiver = GlobalScope, completion = continuation)
    return out!!.getOrThrow()
}
```

### Why it is a bit better

- It does **not** return a Promise disguised as `T`.
- If `block` completes **immediately** (no suspension), it really returns `T`.

### Why it is not a solution

- If the coroutine **suspends**, `out` is still `null`, so it crashes on `out!!`.
- It still uses `GlobalScope`.
- You still can’t “block until resume” on JS/WASM without freezing the event loop.

**The verdict:**

> It can “unwrap” only *non-suspending* coroutines.  
> It cannot turn async work into sync work on JS/WASM.

---

## 8) Conclusion: don’t overuse `runBlocking` (even where it exists)

Even on platforms where `runBlocking` exists (JVM/Android/Native), it should be used **rarely**.

- It **blocks a thread**.
- On Android, using it on the **main/UI thread** can freeze the app (no Looper messages → ANR risk).
- It makes code harder to maintain and easier to deadlock.

In app code, prefer launching coroutines from a proper scope:

- **Android UI layer:** use lifecycle scopes like `viewModelScope`.
- **Lower layers / non-Android:** create your own scope and control its lifetime. For example:

```kotlin
val scope = CoroutineScope(
    SupervisorJob() + Dispatchers.Default + coroutineExceptionHandler
)
```

The best long-term strategy is to **avoid `runBlocking` in your business logic**.
If you later port your app/library to more platforms (especially web / wasmJs), you will have much less work,
because you won’t need to migrate blocking calls to `suspend` and async flows.