# Uno + Skia + Android + NativeAOT, oh my!

# .NET 9

Build, Install, Run:

```sh
% time dotnet build -c Release -f net9.0-android */*.csproj -t:Install,StartAndroidActivity -r android-x64 -bl ; echo exit=$?; notify complete
dotnet build -c Release -f net9.0-android */*.csproj  -r android-x64 -bl  351.23s user 93.02s system 244% cpu 3:01.39 total
```

Resulting `.apk` is 30MB in size.

Launch times:

```
I ActivityManager: Displayed Scratch.UnoSkiaAndroidNativeAOT/crc64780f121fa62707e1.MainActivity: +9s887ms
I ActivityManager: Displayed Scratch.UnoSkiaAndroidNativeAOT/crc64780f121fa62707e1.MainActivity: +5s560ms
I ActivityManager: Displayed Scratch.UnoSkiaAndroidNativeAOT/crc64780f121fa62707e1.MainActivity: +5s518ms
```


# .NET 10

```sh
% export PATH=$HOME/Downloads/dotnet-sdk-10.0.100-preview.6.25358.103-osx-x64:$PATH
% time dotnet build -c Release -f net10.0-android */*.csproj -t:Install,StartAndroidActivity -r android-x64 -bl ; echo exit=$?; notify complete
…
dotnet build -c Release -f net10.0-android */*.csproj  -r android-x64 -bl  352.53s user 84.20s system 298% cpu 2:26.49 total
```

Resulting `.apk` is 30MB in size.

Launch times:

```
I ActivityManager: Displayed Scratch.UnoSkiaAndroidNativeAOT/crc64780f121fa62707e1.MainActivity: +5s234ms
I ActivityManager: Displayed Scratch.UnoSkiaAndroidNativeAOT/crc64780f121fa62707e1.MainActivity: +5s488ms
I ActivityManager: Displayed Scratch.UnoSkiaAndroidNativeAOT/crc64780f121fa62707e1.MainActivity: +5s455ms
```

## Discussion

NativeAOT is enabled by setting `$(PublishAOT)`=true, which is done by default in
`Scratch.UnoSkiaAndroidNativeAOT.csproj` for Release builds when using .NET 10.

Initially, it would crash on startup:

```
E AndroidRuntime: net.dot.jni.internal.JavaProxyThrowable: System.InvalidOperationException: Methods such as __export__ are not implemented!
E AndroidRuntime:    at Microsoft.Android.Runtime.ManagedTypeManager.RegisterNativeMembers(JniType, Type, ReadOnlySpan`1) + 0x386
E AndroidRuntime:    at Java.Interop.ManagedPeer.RegisterNativeMembers(IntPtr jnienv, IntPtr klass, IntPtr n_nativeClass, IntPtr n_methods) + 0x195
E AndroidRuntime:        at net.dot.jni.ManagedPeer.registerNativeMembers(Native Method)
…
F DOTNET  : FATAL UNHANDLED EXCEPTION: System.InvalidOperationException: Methods such as __export__ are not implemented!
F DOTNET  :    at Microsoft.Android.Runtime.ManagedTypeManager.RegisterNativeMembers(JniType, Type, ReadOnlySpan`1) + 0x386
F DOTNET  :    at Java.Interop.ManagedPeer.RegisterNativeMembers(IntPtr jnienv, IntPtr klass, IntPtr n_nativeClass, IntPtr n_methods) + 0x195
```

From the types:

```
rc64d0916be76e7aa092/ApplicationActivity.java
crc645c0cc48a56ad0c27/UnoWebViewHandler.java
```

The problem was the use of `[Export]`, which is removed via the unoplatform/uno patch, below.

After that change, it still crashes, because some of the referenced assemblies hit a method registration codepath which requires System.Reflection.Emit:

```
E AndroidRuntime: net.dot.jni.internal.JavaProxyThrowable: System.PlatformNotSupportedException: PlatformNotSupported_ReflectionEmit
E AndroidRuntime:    at System.Reflection.Emit.ReflectionEmitThrower.ThrowPlatformNotSupportedException() + 0x2f
E AndroidRuntime:    at Android.Runtime.JNINativeWrapper.CreateDelegate(Delegate dlg) + 0x1fc
E AndroidRuntime:    at AndroidX.CustomView.Widget.ExploreByTouchHelper.GetGetVirtualViewAt_FFHandler() + 0x58
E AndroidRuntime:    at Microsoft.Android.Runtime.ManagedTypeManager.RegisterNativeMembers(JniType, Type, ReadOnlySpan`1) + 0x1e7
E AndroidRuntime:    at Java.Interop.ManagedPeer.RegisterNativeMembers(IntPtr jnienv, IntPtr klass, IntPtr n_nativeClass, IntPtr n_methods) + 0x195
E AndroidRuntime: --- End of stack trace from previous location ---
E AndroidRuntime:    at Java.Interop.JniEnvironment.Object.AllocObject(JniObjectReference) + 0x11b
E AndroidRuntime:    at Java.Interop.JniPeerMembers.JniInstanceMethods.StartCreateInstance(String, Type, JniArgumentValue*) + 0x2e
E AndroidRuntime:    at AndroidX.CustomView.Widget.ExploreByTouchHelper..ctor(View host) + 0x122
E AndroidRuntime:    at Uno.UI.Runtime.Skia.Android.UnoExploreByTouchHelper..ctor(UnoSKCanvasView) + 0x8c
E AndroidRuntime:    at Uno.UI.Runtime.Skia.Android.UnoSKCanvasView..ctor(Context) + 0xe0
E AndroidRuntime:    at Microsoft.UI.Xaml.ApplicationActivity.OnStart() + 0x9e
E AndroidRuntime:    at Android.App.Activity.n_OnStart(IntPtr jnienv, IntPtr native__this) + 0x95
E AndroidRuntime:        at crc64d0916be76e7aa092.ApplicationActivity.n_onStart(Native Method)
E AndroidRuntime:        at crc64d0916be76e7aa092.ApplicationActivity.onStart(ApplicationActivity.java:84)
E AndroidRuntime:        at android.app.Instrumentation.callActivityOnStart(Instrumentation.java:1391)
E AndroidRuntime:        at android.app.Activity.performStart(Activity.java:7157)
E AndroidRuntime:        at android.app.ActivityThread.handleStartActivity(ActivityThread.java:2937)
E AndroidRuntime:        at android.app.servertransaction.TransactionExecutor.performLifecycleSequence(TransactionExecutor.java:180)
E AndroidRuntime:        at android.app.servertransaction.TransactionExecutor.cycleToPath(TransactionExecutor.java:165)
E AndroidRuntime:        at android.app.servertransaction.TransactionExecutor.executeLifecycleState(TransactionExecutor.java:142)
E AndroidRuntime:        at android.app.servertransaction.TransactionExecutor.execute(TransactionExecutor.java:70)
E AndroidRuntime:        at android.app.ActivityThread$H.handleMessage(ActivityThread.java:1808)
E AndroidRuntime:        at android.os.Handler.dispatchMessage(Handler.java:106)
E AndroidRuntime:        at android.os.Looper.loop(Looper.java:193)
E AndroidRuntime:        at android.app.ActivityThread.main(ActivityThread.java:6669)
E AndroidRuntime:        at java.lang.reflect.Method.invoke(Native Method)
E AndroidRuntime:        at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:493)
E AndroidRuntime:        at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:858)
F DOTNET  : FATAL UNHANDLED EXCEPTION: System.PlatformNotSupportedException: PlatformNotSupported_ReflectionEmit
F DOTNET  :    at System.Reflection.Emit.ReflectionEmitThrower.ThrowPlatformNotSupportedException() + 0x2f
F DOTNET  :    at Android.Runtime.JNINativeWrapper.CreateDelegate(Delegate dlg) + 0x1fc
F DOTNET  :    at AndroidX.CustomView.Widget.ExploreByTouchHelper.GetGetVirtualViewAt_FFHandler() + 0x58
F DOTNET  :    at Microsoft.Android.Runtime.ManagedTypeManager.RegisterNativeMembers(JniType, Type, ReadOnlySpan`1) + 0x1e7
F DOTNET  :    at Java.Interop.ManagedPeer.RegisterNativeMembers(IntPtr jnienv, IntPtr klass, IntPtr n_nativeClass, IntPtr n_methods) + 0x195
F DOTNET  : --- End of stack trace from previous location ---
F DOTNET  :    at Java.Interop.JniEnvironment.Object.AllocObject(JniObjectReference) + 0x11b
F DOTNET  :    at Java.Interop.JniPeerMembers.JniInstanceMethods.StartCreateInstance(String, Type, JniArgumentValue*) + 0x2e
F DOTNET  :    at AndroidX.CustomView.Widget.ExploreByTouchHelper..ctor(View host) + 0x122
F DOTNET  :    at Uno.UI.Runtime.Skia.Android.UnoExploreByTouchHelper..ctor(UnoSKCanvasView) + 0x8c
```

This is fixed by using newer AndroidX packages which no longer require System.Reflection.Emit.

# unoplatform/uno changes

```diff
diff --git a/src/Uno.UI.Runtime.Skia.Android/ApplicationActivity.cs b/src/Uno.UI.Runtime.Skia.Android/ApplicationActivity.cs
index 7b21ca8457..c2e7aba63b 100644
--- a/src/Uno.UI.Runtime.Skia.Android/ApplicationActivity.cs
+++ b/src/Uno.UI.Runtime.Skia.Android/ApplicationActivity.cs
@@ -429,7 +429,6 @@ namespace Microsoft.UI.Xaml
 		/// </summary>
 		/// <param name="type">A type full name</param>
 		/// <returns>The assembly that contains the specified type</returns>
-		[Java.Interop.Export]
 		[global::System.ComponentModel.EditorBrowsable(global::System.ComponentModel.EditorBrowsableState.Never)]
 		public static string GetTypeAssemblyFullName(string type) => Type.GetType(type)?.Assembly.FullName!;
 
diff --git a/src/Uno.UI/UI/Xaml/ApplicationActivity.Android.cs b/src/Uno.UI/UI/Xaml/ApplicationActivity.Android.cs
index 83df02dd34..73403fcd83 100644
--- a/src/Uno.UI/UI/Xaml/ApplicationActivity.Android.cs
+++ b/src/Uno.UI/UI/Xaml/ApplicationActivity.Android.cs
@@ -427,7 +427,6 @@ namespace Microsoft.UI.Xaml
 		/// </summary>
 		/// <param name="type">A type full name</param>
 		/// <returns>The assembly that contains the specified type</returns>
-		[Java.Interop.Export(nameof(GetTypeAssemblyFullName))]
 		[global::System.ComponentModel.EditorBrowsable(global::System.ComponentModel.EditorBrowsableState.Never)]
 		public static string GetTypeAssemblyFullName(string type) => Type.GetType(type)?.Assembly.FullName;
 
diff --git a/src/Uno.UI/UI/Xaml/Controls/WebView/Native/Android/UnoWebViewHandler.Android.cs b/src/Uno.UI/UI/Xaml/Controls/WebView/Native/Android/UnoWebViewHandler.Android.cs
index c1f1a24487..c52a828b92 100644
--- a/src/Uno.UI/UI/Xaml/Controls/WebView/Native/Android/UnoWebViewHandler.Android.cs
+++ b/src/Uno.UI/UI/Xaml/Controls/WebView/Native/Android/UnoWebViewHandler.Android.cs
@@ -12,7 +12,7 @@ internal class UnoWebViewHandler : Java.Lang.Object
 		_nativeWebView = wrapper;
 	}
 
-	[Export]
+	// [Export]
 	[JavascriptInterface]
 	public void postMessage(string message) => _nativeWebView?.OnWebMessageReceived(message);
 }
diff --git a/src/Uno.UI/UI/Xaml/NativeApplication.cs b/src/Uno.UI/UI/Xaml/NativeApplication.cs
index 843f437308..79ca7eac32 100644
--- a/src/Uno.UI/UI/Xaml/NativeApplication.cs
+++ b/src/Uno.UI/UI/Xaml/NativeApplication.cs
@@ -155,7 +155,7 @@ namespace Microsoft.UI.Xaml
 		/// </summary>
 		/// <param name="type">A type full name</param>
 		/// <returns>The assembly that contains the specified type</returns>
-		[Export(nameof(GetTypeAssemblyFullName))]
+		// [Export(nameof(GetTypeAssemblyFullName))]
 		[global::System.ComponentModel.EditorBrowsable(global::System.ComponentModel.EditorBrowsableState.Never)]
 		public static string GetTypeAssemblyFullName(string type) => Type.GetType(type)?.Assembly.FullName;
 
```