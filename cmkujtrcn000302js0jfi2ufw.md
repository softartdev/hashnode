---
title: "The “Stale Capture” Trap in Jetpack Compose Side Effects (and why it can surprise Kotlin developers)"
seoTitle: "Avoiding Stale Captures in Jetpack Compose"
seoDescription: "Learn about the “stale capture” issue in Jetpack Compose, its causes, and how to safely avoid this common pitfall"
datePublished: Mon Jan 26 2026 02:29:03 GMT+0000 (Coordinated Universal Time)
cuid: cmkujtrcn000302js0jfi2ufw
slug: the-stale-capture-trap-in-jetpack-compose-side-effects-and-why-it-can-surprise-kotlin-developers
ogImage: https://cdn.hashnode.com/res/hashnode/image/upload/v1769394522836/d3ae6b55-dc90-4492-bfa5-e50b19d7cfc1.png
tags: kotlin, jetpack-compose

---

Sometimes your UI shows the correct value, but a log (or callback) inside `LaunchedEffect` uses an old one.  
This is usually not a Compose “bug”. It is a **closure capture** issue: a long-running lambda can keep **old references**.

This article explains the problem with a simple example and shows two safe fixes.

---

## What “stale capture” means

In Kotlin, a lambda can capture variables from the outer scope.  
But it does not capture “magic live variables”. It stores **references** to objects.

So if you capture an object, and later a new object is created, the old lambda will still point to the old object.

This is normal Kotlin behavior.

---

## A simple Compose example

Imagine a screen where:

- the ViewModel sometimes provides `text` (for example, after loading from a database)
- the user edits a text field
- after a short delay, we run validation and call `onValidate(text)`

### Code (buggy)

```kotlin
@Composable
fun DemoScreen(
    text: String,        // changes over time (ViewModel update)
    onValidate: (String) -> Unit // can also change on recomposition
) {
    // ⚠️ This creates a NEW state object when text changes.
    val textState = remember(text) {
        mutableStateOf(text)
    }

    // We start ONE coroutine for the whole lifetime of this call site.
    LaunchedEffect(Unit) {
        // Some long-running work (timer, polling, etc.)
        delay(2_000)

        // ⚠️ This lambda can use an old reference to textState and/or old onValidate.
        onValidate(textState.value)
    }

    TextField(
        value = textState.value,
        onValueChange = { textState.value = it }
    )
}
```

### What can happen

1) First composition: `text = ""`  
   Compose creates `textState₁` (object A) and starts the `LaunchedEffect`.

2) Later: `text = "Hello"`  
   Because of `remember(text)`, Compose creates **new** `textState₂` (object B).  
   The UI now reads object B.

3) After 2 seconds the coroutine finishes `delay(...)`  
   The coroutine may still hold object A, so it calls `onValidate("")` even though the UI shows `"Hello"`.

So UI and side effects can disagree, even if the UI is correct.

---

## Why this happens in `LaunchedEffect`

`LaunchedEffect(key1, key2, ...)` starts a coroutine.  
That coroutine keeps running **until the keys change**.

If your effect uses a value but that value is not in the keys, the coroutine may continue running with old references.

That is why Compose has two main patterns:

- restart the effect when inputs change (use keys)
- keep the effect running, but read the **latest** values (`rememberUpdatedState`)

---

## Fix 1: Restart the effect (add the right keys)

If it is OK to restart the coroutine when input changes, add keys:

```kotlin
LaunchedEffect(text, onValidate) {
    delay(2_000)
    onValidate(textState.value)
}
```

Now when `text` changes, the old coroutine is cancelled and a new one starts.

**Use this when restarting is cheap and safe.**

---

## Fix 2: Keep the effect stable, but read the latest value

If you do *not* want to restart the coroutine, use `rememberUpdatedState`.

See the official docs for [`rememberUpdatedState`](https://developer.android.com/develop/ui/compose/side-effects#rememberupdatedstate).

```kotlin
val latestText by rememberUpdatedState(textState.value)
val latestOnValidate by rememberUpdatedState(onValidate)

LaunchedEffect(Unit) {
    delay(2_000)
    latestOnValidate(latestText)
}
```

Idea: `rememberUpdatedState` returns a stable holder, and Compose updates its value on recomposition.  
So your long-running coroutine keeps the same reference, but it reads the latest value.

**Use this when you want one collector, one listener, one timer, etc.**

---

## How to “see” the problem (object identity)

On JVM you can print identity to prove that the state object changed:

```kotlin
SideEffect {
    println("textState identity = ${System.identityHashCode(textState)}")
}
```

When `remember(text)` recreates the state, the identity will change.

---

## Under the hood (optional)

Compose state is implemented in the runtime. If you want to see a real implementation file, check  
[`SnapshotState.kt` in Compose runtime](https://github.com/JetBrains/compose-multiplatform-core/blob/jb-main/compose/runtime/runtime/src/commonMain/kotlin/androidx/compose/runtime/SnapshotState.kt#L340).

You do not need to read it to fix the bug, but it helps to understand that “state” is a real object.

---

## Practical checklist

When you write a long-running side effect (`LaunchedEffect`, `DisposableEffect`, timers, listeners, etc.):

1) List what the effect reads (state objects, values, callbacks).
2) For each item, choose:
   - **Restart the effect** when it changes → put it in the effect keys
   - **Do not restart**, but need the latest value → use `rememberUpdatedState`

This simple rule prevents most stale-capture bugs.

---

## References

- Android Developers — side effects and [`rememberUpdatedState`](https://developer.android.com/develop/ui/compose/side-effects#rememberupdatedstate)  
- Compose runtime source — [`SnapshotState.kt`](https://github.com/JetBrains/compose-multiplatform-core/blob/jb-main/compose/runtime/runtime/src/commonMain/kotlin/androidx/compose/runtime/SnapshotState.kt#L340)