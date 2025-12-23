---
title: "Under the Hood: How Compose Multiplatform Opens a URL"
seoTitle: "Composable URL Navigation in Multiplatform Compose"
seoDescription: "Explore how Compose Multiplatform powers cross-platform URL opening, unraveling the journey from Kotlin code to OS-specific browser execution"
datePublished: Tue Dec 23 2025 09:47:30 GMT+0000 (Coordinated Universal Time)
cuid: cmjiein8f000u02l82znga8as
slug: under-the-hood-how-compose-multiplatform-opens-a-url
tags: kotlin-multiplatform, compose-multiplatform

---

As mobile developers moving into the Multiplatform space, we often take high-level APIs for granted. A perfect example is opening a hyperlink. In Jetpack Compose, we simply grab the `LocalUriHandler` and call `openUri()`. But what happens after that call? How does a single line of Kotlin code trigger the default browser on Android, iOS, Windows, macOS, and Linux?

Today, we are diving deep into the call stack—from the Compose API down to the C++ JNI calls in the JDK.

For example, `UriHandler` is used like this (Kotlin Multiplatform RSS Reader sample):

* [https://github.com/Kotlin/kmp-production-sample/blob/master/shared/src/commonMain/kotlin/com/github/jetbrains/rssreader/ui/Screen.kt#L81](https://github.com/Kotlin/kmp-production-sample/blob/master/shared/src/commonMain/kotlin/com/github/jetbrains/rssreader/ui/Screen.kt#L81)
    

```kotlin
val uriHandler = LocalUriHandler.current
// ... some code omitted
uriHandler.openUri(url)
```

This allows opening a link in an external browser on Android, iOS, Desktop (Java/AWT) and Web (wasmJs).

## Compose API

`UriHandler` interface (AndroidX source code):

* AOSP: [https://android.googlesource.com/platform/frameworks/support/+/refs/heads/androidx-main/compose/ui/ui/src/commonMain/kotlin/androidx/compose/ui/platform/UriHandler.kt](https://android.googlesource.com/platform/frameworks/support/+/refs/heads/androidx-main/compose/ui/ui/src/commonMain/kotlin/androidx/compose/ui/platform/UriHandler.kt)
    
* JetBrains fork: [https://github.com/JetBrains/compose-multiplatform-core/blob/jb-main/compose/ui/ui/src/commonMain/kotlin/androidx/compose/ui/platform/UriHandler.kt](https://github.com/JetBrains/compose-multiplatform-core/blob/jb-main/compose/ui/ui/src/commonMain/kotlin/androidx/compose/ui/platform/UriHandler.kt)
    

```kotlin
interface UriHandler {
    fun openUri(uri: String)
}
```

Android implementation (`AndroidUriHandler`):

* AOSP: [https://android.googlesource.com/platform/frameworks/support/+/refs/heads/androidx-main/compose/ui/ui/src/androidMain/kotlin/androidx/compose/ui/platform/AndroidUriHandler.android.kt](https://android.googlesource.com/platform/frameworks/support/+/refs/heads/androidx-main/compose/ui/ui/src/androidMain/kotlin/androidx/compose/ui/platform/AndroidUriHandler.android.kt)
    
* JetBrains fork: [https://github.com/JetBrains/compose-multiplatform-core/blob/jb-main/compose/ui/ui/src/androidMain/kotlin/androidx/compose/ui/platform/AndroidUriHandler.android.kt](https://github.com/JetBrains/compose-multiplatform-core/blob/jb-main/compose/ui/ui/src/androidMain/kotlin/androidx/compose/ui/platform/AndroidUriHandler.android.kt)
    

```kotlin
class AndroidUriHandler(private val context: Context) : UriHandler {
    override fun openUri(uri: String) {
        try {
            context.startActivity(Intent(Intent.ACTION_VIEW, Uri.parse(uri)))
        } catch (e: ActivityNotFoundException) {
            throw IllegalArgumentException("Can't open $uri.", e)
        }
    }
}
```

## Desktop / iOS / Web (Skiko-based)

Next, Compose delegates to a Skiko-based implementation (`PlatformUriHandler`):

* [https://github.com/JetBrains/compose-multiplatform-core/blob/jb-main/compose/ui/ui/src/skikoMain/kotlin/androidx/compose/ui/platform/PlatformUriHandler.skiko.kt](https://github.com/JetBrains/compose-multiplatform-core/blob/jb-main/compose/ui/ui/src/skikoMain/kotlin/androidx/compose/ui/platform/PlatformUriHandler.skiko.kt)
    

```kotlin
internal class PlatformUriHandler : UriHandler, URIManager()
```

`URIManager` and `URIHandler_openUri` are defined in Skiko:

* [https://github.com/JetBrains/skiko/blob/master/skiko/src/commonMain/kotlin/org/jetbrains/skiko/Platform.kt](https://github.com/JetBrains/skiko/blob/master/skiko/src/commonMain/kotlin/org/jetbrains/skiko/Platform.kt)
    

```kotlin
open class URIManager {
    open fun openUri(uri: String) = URIHandler_openUri(uri)
}
internal expect fun URIHandler_openUri(uri: String)
```

### Skiko platform actuals

**Desktop (AWT / Java):**

* [https://github.com/JetBrains/skiko/blob/master/skiko/src/awtMain/kotlin/org/jetbrains/skiko/Actuals.awt.kt](https://github.com/JetBrains/skiko/blob/master/skiko/src/awtMain/kotlin/org/jetbrains/skiko/Actuals.awt.kt)
    

```kotlin
// excerpt
val desktop = Desktop.getDesktop()
if (desktop.isSupported(Desktop.Action.BROWSE)) {
    desktop.browse(URI(uri))
    return
}
// Linux fallback: Runtime.getRuntime().exec(arrayOf("xdg-open", URI(uri).toString()))
```

**macOS (Kotlin/Native):**

* [https://github.com/JetBrains/skiko/blob/master/skiko/src/macosMain/kotlin/org/jetbrains/skiko/Actuals.macos.kt](https://github.com/JetBrains/skiko/blob/master/skiko/src/macosMain/kotlin/org/jetbrains/skiko/Actuals.macos.kt)
    

```kotlin
internal actual fun URIHandler_openUri(uri: String) {
    NSWorkspace.sharedWorkspace.openURL(NSURL.URLWithString(uri)!!)
}
```

**Web (wasmJs):**

* [https://github.com/JetBrains/skiko/blob/master/skiko/src/wasmJsMain/kotlin/org/jetbrains/skiko/Actuals.wasmJs.kt](https://github.com/JetBrains/skiko/blob/master/skiko/src/wasmJsMain/kotlin/org/jetbrains/skiko/Actuals.wasmJs.kt)
    

```kotlin
internal actual fun URIHandler_openUri(uri: String) {
    window.open(uri, target = "_blank")
}
```

**iOS (UIKit):**

* [https://github.com/JetBrains/skiko/blob/master/skiko/src/uikitMain/kotlin/org/jetbrains/skiko/Actuals.uikit.kt](https://github.com/JetBrains/skiko/blob/master/skiko/src/uikitMain/kotlin/org/jetbrains/skiko/Actuals.uikit.kt)
    

```kotlin
internal actual fun URIHandler_openUri(uri: String) {
    UIApplication.sharedApplication.openURL(
        url = URLWithString(uri)!!,
        options = emptyMap<Any?, Any>(),
        completionHandler = null
    )
}
```

# What `java.awt.Desktop.browse()` actually does (OpenJDK)

Next: what `java.awt.Desktop.browse()` (AWT, not Compose) actually does, and how it opens the browser on different OSes.

## High-level flow

In `java.awt.Desktop`, `browse()` does a few checks and delegates to the platform peer:

```java
public void browse(URI uri) throws IOException {
    checkAWTPermission();
    checkExec();
    checkActionSupport(Action.BROWSE);
    Objects.requireNonNull(uri);
    peer.browse(uri);
}
```

The peer comes from `SunToolkit.createDesktopPeer(this)` inside the private constructor:

```java
private Desktop() {
    Toolkit defaultToolkit = Toolkit.getDefaultToolkit();
    // same cast as in isDesktopSupported()
    if (defaultToolkit instanceof SunToolkit) {
        peer = ((SunToolkit) defaultToolkit).createDesktopPeer(this);
    }
}
```

Sources:

* Tag: [https://github.com/openjdk/jdk17u/tree/jdk-17.0.2-ga](https://github.com/openjdk/jdk17u/tree/jdk-17.0.2-ga)
    
* Tarball: [https://openjdk-sources.osci.io/openjdk17/openjdk-17.0.2-ga.tar.xz](https://openjdk-sources.osci.io/openjdk17/openjdk-17.0.2-ga.tar.xz)
    
* Release announcement: [https://mail.openjdk.org/pipermail/jdk-updates-dev/2022-January/010793.html](https://mail.openjdk.org/pipermail/jdk-updates-dev/2022-January/010793.html)
    

Files:

* Desktop API: [https://github.com/openjdk/jdk17u/blob/jdk-17.0.2-ga/src/java.desktop/share/classes/java/awt/Desktop.java](https://github.com/openjdk/jdk17u/blob/jdk-17.0.2-ga/src/java.desktop/share/classes/java/awt/Desktop.java)
    

## Windows

On Windows, the peer is [`sun.awt.windows`](http://sun.awt.windows)`.WDesktopPeer`. It calls a native `ShellExecute(...)` wrapper:

```java
@Override
public void browse(URI uri) throws IOException {
    this.ShellExecute(uri, ACTION_OPEN_VERB);
}

private void ShellExecute(URI uri, String verb) throws IOException {
    String errmsg = ShellExecute(uri.toString(), verb);

    if (errmsg != null) {
        throw new IOException("Failed to " + verb + " " + uri
                + ". Error message: " + errmsg);
    }
}

private static native String ShellExecute(String fileOrUri, String verb);
```

Native side uses Windows ShellExecute (after COM init):

```cpp
JNIEXPORT jstring JNICALL Java_sun_awt_windows_WDesktopPeer_ShellExecute
  (JNIEnv *env, jclass cls, jstring fileOrUri_j, jstring verb_j)
{
    ...
    HRESULT hr = ::CoInitializeEx(NULL, COINIT_APARTMENTTHREADED |
                                        COINIT_DISABLE_OLE1DDE);
    HINSTANCE retval;
    DWORD error;
    if (SUCCEEDED(hr)) {
        retval = ::ShellExecute(NULL, verb_c, fileOrUri_c, NULL, NULL,
                                SW_SHOWNORMAL);
        error = ::GetLastError();
        ::CoUninitialize();
    }
    ...
}
```

Files:

* [https://github.com/openjdk/jdk17u/blob/jdk-17.0.2-ga/src/java.desktop/windows/classes/sun/awt/windows/WDesktopPeer.java](https://github.com/openjdk/jdk17u/blob/jdk-17.0.2-ga/src/java.desktop/windows/classes/sun/awt/windows/WDesktopPeer.java)
    
* [https://github.com/openjdk/jdk17u/blob/jdk-17.0.2-ga/src/java.desktop/windows/native/libawt/windows/awt\_Desktop.cpp](https://github.com/openjdk/jdk17u/blob/jdk-17.0.2-ga/src/java.desktop/windows/native/libawt/windows/awt_Desktop.cpp)
    

## macOS

On macOS, the peer is `sun.lwawt.macosx.CDesktopPeer`. `browse()` delegates to `lsOpen(uri)` which calls native `_lsOpenURI(...)`:

```java
@Override
public void browse(URI uri) throws IOException {
    this.lsOpen(uri);
}

private void lsOpen(URI uri) throws IOException {
    int status = _lsOpenURI(uri.toString());

    if (status != 0 /* noErr */) {
        throw new IOException("Failed to mail or browse " + uri + ". Error code: " + status);
    }
}

private static native int _lsOpenURI(String uri);
```

Native side uses LaunchServices (`LSOpenURLsWithRole`):

```plaintext
JNIEXPORT jint JNICALL Java_sun_lwawt_macosx_CDesktopPeer__1lsOpenURI
(JNIEnv *env, jclass clz, jstring uri)
{
    OSStatus status = noErr;
JNI_COCOA_ENTER(env);

    // So we use LaunchServices directly.
    NSURL *url = [NSURL URLWithString:JavaStringToNSString(env, uri)];

    LSLaunchFlags flags = kLSLaunchDefaults;
    LSApplicationParameters params = {0, flags, NULL, NULL, NULL, NULL, NULL};
    status = LSOpenURLsWithRole((CFArrayRef)[NSArray arrayWithObject:url],
                                kLSRolesAll, NULL, &params, NULL, 0);

JNI_COCOA_EXIT(env);
    return status;
}
```

Files:

* [https://github.com/openjdk/jdk17u/blob/jdk-17.0.2-ga/src/java.desktop/macosx/classes/sun/lwawt/macosx/CDesktopPeer.java](https://github.com/openjdk/jdk17u/blob/jdk-17.0.2-ga/src/java.desktop/macosx/classes/sun/lwawt/macosx/CDesktopPeer.java)
    
* [https://github.com/openjdk/jdk17u/blob/jdk-17.0.2-ga/src/java.desktop/macosx/native/libawt\_lwawt/awt/CDesktopPeer.m](https://github.com/openjdk/jdk17u/blob/jdk-17.0.2-ga/src/java.desktop/macosx/native/libawt_lwawt/awt/CDesktopPeer.m)
    

## Linux / X11 (Unix)

On Linux/X11, the peer is `sun.awt.X11.XDesktopPeer`. Java delegates to a native `gnome_url_show(...)`:

```java
public void browse(URI uri) throws IOException {
    launch(uri);
}

private void launch(URI uri) throws IOException {
    byte[] uriByteArray = ( uri.toString() + '\0' ).getBytes();
    boolean result = false;
    XToolkit.awtLock();
    try {
        if (!nativeLibraryLoaded) {
            throw new IOException("Failed to load native libraries.");
        }
        result = gnome_url_show(uriByteArray);
    } finally {
        XToolkit.awtUnlock();
    }
    if (!result) {
        throw new IOException("Failed to show URI:" + uri);
    }
}

private native boolean gnome_url_show(byte[] url);
private static native boolean init(int gtkVersion, boolean verbose);
```

Native side tries GTK first (`gtk_show_uri`), then GNOME (`gnome_url_show`):

```c
JNIEXPORT jboolean JNICALL Java_sun_awt_X11_XDesktopPeer_init
  (JNIEnv *env, jclass cls, jint version, jboolean verbose)
{
    if (gtk_has_been_loaded || gnome_has_been_loaded) {
        return JNI_TRUE;
    }

    if (gtk_load(env, version, verbose) && gtk->show_uri_load(env)) {
        gtk_has_been_loaded = TRUE;
        return JNI_TRUE;
    } else if (gnome_load()) {
        gnome_has_been_loaded = TRUE;
        return JNI_TRUE;
    }

...

JNIEXPORT jboolean JNICALL Java_sun_awt_X11_XDesktopPeer_gnome_1url_1show
  (JNIEnv *env, jobject obj, jbyteArray url_j)
{
    ...
    if (gtk_has_been_loaded) {
        gtk->gdk_threads_enter();
        success = gtk->gtk_show_uri(NULL, url_c, GDK_CURRENT_TIME, NULL);
        gtk->gdk_threads_leave();
    } else if (gnome_has_been_loaded) {
        success = (*gnome_url_show)(url_c, NULL);
    }
    ...
}
```

Files:

* [https://github.com/openjdk/jdk17u/blob/jdk-17.0.2-ga/src/java.desktop/unix/classes/sun/awt/X11/XDesktopPeer.java](https://github.com/openjdk/jdk17u/blob/jdk-17.0.2-ga/src/java.desktop/unix/classes/sun/awt/X11/XDesktopPeer.java)
    
* [https://github.com/openjdk/jdk17u/blob/jdk-17.0.2-ga/src/java.desktop/unix/native/libawt\_xawt/xawt/awt\_Desktop.c](https://github.com/openjdk/jdk17u/blob/jdk-17.0.2-ga/src/java.desktop/unix/native/libawt_xawt/xawt/awt_Desktop.c)
    

## Other platforms

In OpenJDK, desktop support is primarily implemented for Windows, macOS, and Unix/X11. On headless/minimal builds (or when native libraries are missing), `Desktop` may report that `BROWSE` is unsupported and throw `UnsupportedOperationException` / `IOException` instead of opening a browser.

## Summary

When you write `uriHandler.openUri("`[https://hashnode.com/](https://hashnode.com/)`")` in your common Compose code, you are triggering a massive chain of abstractions:

1. **Compose:** `LocalUriHandler` delegates to platform implementation.
    
2. **Skiko:** Delegates to `java.awt.Desktop` (on JVM).
    
3. **AWT:** Delegates to OS-specific Peers (`WDesktopPeer`, `CDesktopPeer`, `XDesktopPeer`).
    
4. **JNI:** Crosses the boundary into C/C++/Objective-C.
    
5. **OS API:** Finally calls `ShellExecute`, `LSOpenURLsWithRole`, or `gtk_show_uri`.
    

It is a long journey for a simple click, but it highlights the power of Kotlin Multiplatform: writing once, and letting the framework handle the system-level complexity.