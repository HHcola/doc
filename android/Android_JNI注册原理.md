# <center>Android JNI 注册原理</center>

Android中Java层调用Native层的方法，是如何寻找到对应的方法，Native中的方法是在何时进行注册的，带着这些疑问，一起看看JNI方法的加载。

## JNI原理分析

用FlutterJNI中loadLibrary加载flutter so为例<br>

### 1.1 System.loadLibrary

[io.flutter.embedding.engine.FlutterJNI.java]
```
  public void loadLibrary() {
    if (FlutterJNI.loadLibraryCalled) {
      Log.w(TAG, "FlutterJNI.loadLibrary called more than once");
    }

    System.loadLibrary("flutter");
    FlutterJNI.loadLibraryCalled = true;
  }
```

核心问题是解决System.loadLibrary是如何加载so库，以及在加载过程中如何注册JNI的方法。

我们在Android里使用JNI时也是同样的道理，需要加载so库，然后才能调用native方法，这里加载流程是需要研究的。


[java.lang.System.java]
```
  public static void loadLibrary(String libname) {
        Runtime.getRuntime().loadLibrary0(Reflection.getCallerClass(), libname);
    }
```


[Runtime.java]

```
  private synchronized void loadLibrary0(ClassLoader loader, Class<?> callerClass, String libname) {
        if (libname.indexOf((int)File.separatorChar) != -1) {
            throw new UnsatisfiedLinkError(
    "Directory separator should not appear in library name: " + libname);
        }
        String libraryName = libname;
        // Android-note: BootClassLoader doesn't implement findLibrary(). http://b/111850480
        // Android's class.getClassLoader() can return BootClassLoader where the RI would
        // have returned null; therefore we treat BootClassLoader the same as null here.
        if (loader != null && !(loader instanceof BootClassLoader)) {
            String filename = loader.findLibrary(libraryName);
            if (filename == null &&
                    (loader.getClass() == PathClassLoader.class ||
                     loader.getClass() == DelegateLastClassLoader.class)) {
                // Don't give up even if we failed to find the library in the native lib paths.
                // The underlying dynamic linker might be able to find the lib in one of the linker
                // namespaces associated with the current linker namespace. In order to give the
                // dynamic linker a chance, proceed to load the library with its soname, which
                // is the fileName.
                // Note that we do this only for PathClassLoader  and DelegateLastClassLoader to
                // minimize the scope of this behavioral change as much as possible, which might
                // cause problem like b/143649498. These two class loaders are the only
                // platform-provided class loaders that can load apps. See the classLoader attribute
                // of the application tag in app manifest.
                filename = System.mapLibraryName(libraryName);
            }
            if (filename == null) {
                // It's not necessarily true that the ClassLoader used
                // System.mapLibraryName, but the default setup does, and it's
                // misleading to say we didn't find "libMyLibrary.so" when we
                // actually searched for "liblibMyLibrary.so.so".
                throw new UnsatisfiedLinkError(loader + " couldn't find \"" +
                                               System.mapLibraryName(libraryName) + "\"");
            }
            String error = nativeLoad(filename, loader);
            if (error != null) {
                throw new UnsatisfiedLinkError(error);
            }
            return;
        }

        // We know some apps use mLibPaths directly, potentially assuming it's not null.
        // Initialize it here to make sure apps see a non-null value.
        getLibPaths();
        String filename = System.mapLibraryName(libraryName);
        String error = nativeLoad(filename, loader, callerClass);
        if (error != null) {
            throw new UnsatisfiedLinkError(error);
        }
    }
```

核心加载逻辑在nativeLoad。

### 1.2 native层nativeLoad
davlik和art在native层的对应的类不一样：<br>
**davlik:**runtime/native/java_lang_Runtime.cc
**art:**:libcore/ojluni/src/main/native/Runtime.c


```
JNIEXPORT jstring JNICALL
Runtime_nativeLoad(JNIEnv* env, jclass ignored, jstring javaFilename,
                   jobject javaLoader, jclass caller)
{
    return JVM_NativeLoad(env, javaFilename, javaLoader, caller);
}
```
这里会调用JVM_NativeLoad
[art/openjdkjvm/OpenjdkJvm.cc]

```
JNIEXPORT jstring JVM_NativeLoad(JNIEnv* env,
                                 jstring javaFilename,
                                 jobject javaLoader,
                                 jclass caller) {
  ScopedUtfChars filename(env, javaFilename);
  if (filename.c_str() == nullptr) {
    return nullptr;
  }

  std::string error_msg;
  {
    art::JavaVMExt* vm = art::Runtime::Current()->GetJavaVM();
    bool success = vm->LoadNativeLibrary(env,
                                         filename.c_str(),
                                         javaLoader,
                                         caller,
                                         &error_msg);
    if (success) {
      return nullptr;
    }
  }

  // Don't let a pending exception from JNI_OnLoad cause a CheckJNI issue with NewStringUTF.
  env->ExceptionClear();
  return env->NewStringUTF(error_msg.c_str());
}
```

[art/runtime/jni/java_vm_ext.cc]

终于看到核心加载逻辑，这个方法比好长：
```
bool JavaVMExt::LoadNativeLibrary(JNIEnv* env,
                                  const std::string& path,
                                  jobject class_loader,
                                  jclass caller_class,
                                  std::string* error_msg) {
  
  // 寻找加载so
  // 调用JNI_OnLoad方法
  bool was_successful = false;
  void* sym = library->FindSymbol("JNI_OnLoad", nullptr, android::kJNICallTypeRegular);
  if (sym == nullptr) {
    VLOG(jni) << "[No JNI_OnLoad found in \"" << path << "\"]";
    was_successful = true;
  } else {
    // Call JNI_OnLoad.  We have to override the current class
    // loader, which will always be "null" since the stuff at the
    // top of the stack is around Runtime.loadLibrary().  (See
    // the comments in the JNI FindClass function.)
    ScopedLocalRef<jobject> old_class_loader(env, env->NewLocalRef(self->GetClassLoaderOverride()));
    self->SetClassLoaderOverride(class_loader);

    VLOG(jni) << "[Calling JNI_OnLoad in \"" << path << "\"]";
    using JNI_OnLoadFn = int(*)(JavaVM*, void*);
    JNI_OnLoadFn jni_on_load = reinterpret_cast<JNI_OnLoadFn>(sym);
    int version = (*jni_on_load)(this, nullptr);

    if (IsSdkVersionSetAndAtMost(runtime_->GetTargetSdkVersion(), SdkVersion::kL)) {
      // Make sure that sigchain owns SIGSEGV.
      EnsureFrontOfChain(SIGSEGV);
    }

    self->SetClassLoaderOverride(old_class_loader.get());

    if (version == JNI_ERR) {
      StringAppendF(error_msg, "JNI_ERR returned from JNI_OnLoad in \"%s\"", path.c_str());
    } else if (JavaVMExt::IsBadJniVersion(version)) {
      StringAppendF(error_msg, "Bad JNI version returned from JNI_OnLoad in \"%s\": %d",
                    path.c_str(), version);
      // It's unwise to call dlclose() here, but we can mark it
      // as bad and ensure that future load attempts will fail.
      // We don't know how far JNI_OnLoad got, so there could
      // be some partially-initialized stuff accessible through
      // newly-registered native method calls.  We could try to
      // unregister them, but that doesn't seem worthwhile.
    } else {
      was_successful = true;
    }
    VLOG(jni) << "[Returned " << (was_successful ? "successfully" : "failure")
              << " from JNI_OnLoad in \"" << path << "\"]";
  }

  library->SetResult(was_successful);
  return was_successful;
}
```

这里看到了System.loadLibrary加载so库之后，会调用对应的JNI_OnLoad，在这个方法里会注册JNI的方法。整个注册流程已经情绪，在回来看Flutter的注册。

## Flutter JNI注册
### 2.1 JNI_OnLoad
java层调用System.loadLibrary("flutter");之后，会调用library_loader.cc

[flutter/shell/platform/android/native_library.cc]

```
JNIEXPORT jint JNI_OnLoad(JavaVM* vm, void* reserved) {
  // Initialize the Java VM.
  fml::jni::InitJavaVM(vm);

  JNIEnv* env = fml::jni::AttachCurrentThread();
  bool result = false;

  // Register FlutterMain.
  result = flutter::FlutterMain::Register(env);
  FML_CHECK(result);

  // Register PlatformView
  result = flutter::PlatformViewAndroid::Register(env);
  FML_CHECK(result);

  // Register VSyncWaiter.
  result = flutter::VsyncWaiterAndroid::Register(env);
  FML_CHECK(result);

  // Register AndroidImageDecoder.
  result = flutter::AndroidImageGenerator::Register(env);
  FML_CHECK(result);

  return JNI_VERSION_1_4;
}
```

这里重点看flutter::PlatformViewAndroid::Register(env)

### 2.2 FlutterJNI注册实现
[flutter/shell/platform/android/platform_view_android_jni_impl.cc]
```
bool RegisterApi(JNIEnv* env) {
  static const JNINativeMethod flutter_jni_methods[] = {
      // Start of methods from FlutterJNI
      {
          .name = "nativeAttach",
          .signature = "(Lio/flutter/embedding/engine/FlutterJNI;)J",
          .fnPtr = reinterpret_cast<void*>(&AttachJNI),
      },
      {
          .name = "nativeDestroy",
          .signature = "(J)V",
          .fnPtr = reinterpret_cast<void*>(&DestroyJNI),
      },
      {
          .name = "nativeSpawn",
          .signature = "(JLjava/lang/String;Ljava/lang/String;Ljava/lang/"
                       "String;Ljava/util/List;)Lio/flutter/"
                       "embedding/engine/FlutterJNI;",
          .fnPtr = reinterpret_cast<void*>(&SpawnJNI),
      },
      {
          .name = "nativeRunBundleAndSnapshotFromLibrary",
          .signature = "(JLjava/lang/String;Ljava/lang/String;"
                       "Ljava/lang/String;Landroid/content/res/"
                       "AssetManager;Ljava/util/List;)V",
          .fnPtr = reinterpret_cast<void*>(&RunBundleAndSnapshotFromLibrary),
      },
      {
          .name = "nativeDispatchEmptyPlatformMessage",
          .signature = "(JLjava/lang/String;I)V",
          .fnPtr = reinterpret_cast<void*>(&DispatchEmptyPlatformMessage),
      },
      {
          .name = "nativeCleanupMessageData",
          .signature = "(J)V",
          .fnPtr = reinterpret_cast<void*>(&CleanupMessageData),
      },
```

这里面注册了FlutterJNI的相关native方法。至此分析了JNI注册原理，以及Flutter动态注册实现。


