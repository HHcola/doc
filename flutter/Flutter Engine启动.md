# Flutter Engine 启动流程

从Android平台启动Flutter入手，分析是如何启动Engine，以及在启动Engine过程中，涉及到那些线程、以及如何执行到Dart代码。

## Android启动Flutter页面

### 1.1 初始化Flutter engine
Android中启动FlutterActivity，在onStart时会调用`delegate.onStart();`，这里的delegate是FlutterActivityAndFragmentDelegate
具体实现如下：

```
  void onStart() {
    Log.v(TAG, "onStart()");
    ensureAlive();
    doInitialFlutterViewRun();
  }
```

doInitialFlutterViewRun方法里实现了具体的Flutter engine的初始化工作。

[io.flutter.embedding.engine.dart.DartExecutor.java]

```
  public void executeDartEntrypoint(
      @NonNull DartEntrypoint dartEntrypoint, @Nullable List<String> dartEntrypointArgs) {
    if (isApplicationRunning) {
      Log.w(TAG, "Attempted to run a DartExecutor that is already running.");
      return;
    }

    TraceSection.begin("DartExecutor#executeDartEntrypoint");
    try {
      Log.v(TAG, "Executing Dart entrypoint: " + dartEntrypoint);
      flutterJNI.runBundleAndSnapshotFromLibrary(
          dartEntrypoint.pathToBundle,
          dartEntrypoint.dartEntrypointFunctionName,
          dartEntrypoint.dartEntrypointLibrary,
          assetManager,
          dartEntrypointArgs);

      isApplicationRunning = true;
    } finally {
      TraceSection.end();
    }
  }
```

会调用flutterJNI，这个是Android层java到Flutter native的入口类：<br>

```
 @UiThread
  public void runBundleAndSnapshotFromLibrary(
      @NonNull String bundlePath,
      @Nullable String entrypointFunctionName,
      @Nullable String pathToEntrypointFunction,
      @NonNull AssetManager assetManager,
      @Nullable List<String> entrypointArgs) {
    ensureRunningOnMainThread();
    ensureAttachedToNative();
    nativeRunBundleAndSnapshotFromLibrary(
        nativeShellHolderId,
        bundlePath,
        entrypointFunctionName,
        pathToEntrypointFunction,
        assetManager,
        entrypointArgs);
  }
```

FlutterJNI对应的实现在platform_view_android_jni_impl：

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
```

RunBundleAndSnapshotFromLibrary方法会调用ANDROID_SHELL_HOLDER->Launch
其中ANDROID_SHELL_HOLDER的定义如下：
```
#define ANDROID_SHELL_HOLDER \
  (reinterpret_cast<AndroidShellHolder*>(shell_holder))
```
这里的shell_holder和什么对象绑定？是一个全局对象吗？
需要从java层入手：
flutterJNI是和FlutterEngine绑定的：
[io.flutter.embedding.engine.FlutterEngine.java]
```
  public FlutterEngine(
      @NonNull Context context,
      @Nullable FlutterLoader flutterLoader,
      @NonNull FlutterJNI flutterJNI,
      @NonNull PlatformViewsController platformViewsController,
      @Nullable String[] dartVmArgs,
      boolean automaticallyRegisterPlugins,
      boolean waitForRestorationData) {
    AssetManager assetManager;
    try {
      assetManager = context.createPackageContext(context.getPackageName(), 0).getAssets();
    } catch (NameNotFoundException e) {
      assetManager = context.getAssets();
    }

    FlutterInjector injector = FlutterInjector.instance();

    if (flutterJNI == null) {
      flutterJNI = injector.getFlutterJNIFactory().provideFlutterJNI(); // 这里会创建flutterJNI对象
    }
    this.flutterJNI = flutterJNI;

    this.dartExecutor = new DartExecutor(flutterJNI, assetManager);
    this.dartExecutor.onAttachedToJNI();


    if (!flutterJNI.isAttached()) {
      attachToJni(); // 绑定到jni层
    }

```

会调用：`private native long nativeAttach(@NonNull FlutterJNI flutterJNI);`
对应的JNI层：
```
static jlong AttachJNI(JNIEnv* env, jclass clazz, jobject flutterJNI) {
  fml::jni::JavaObjectWeakGlobalRef java_object(env, flutterJNI);
  std::shared_ptr<PlatformViewAndroidJNI> jni_facade =
      std::make_shared<PlatformViewAndroidJNIImpl>(java_object);
  auto shell_holder = std::make_unique<AndroidShellHolder>(
      FlutterMain::Get().GetSettings(), jni_facade);
  if (shell_holder->IsValid()) {
    return reinterpret_cast<jlong>(shell_holder.release());
  } else {
    return 0;
  }
}
```
这里的shell_holder是和Java层的FlutterEngine绑定，是一个Engine对象对应的关系，并不是全局的对象。
继续Engine初始化的流程，调用AndroidShellHolder::Launch。


### 1.2 Flutter Shell
AndroidShellHolder实现在android_shell_holder.cc
[flutter/shell/platform/android/android_shell_holder.cc]

```
void AndroidShellHolder::Launch(
    std::unique_ptr<APKAssetProvider> apk_asset_provider,
    const std::string& entrypoint,
    const std::string& libraryUrl,
    const std::vector<std::string>& entrypoint_args) {
  if (!IsValid()) {
    return;
  }

  apk_asset_provider_ = std::move(apk_asset_provider);
  auto config = BuildRunConfiguration(entrypoint, libraryUrl, entrypoint_args);
  if (!config) {
    return;
  }
  UpdateDisplayMetrics();
  shell_->RunEngine(std::move(config.value()));
}
```

[flutter/shell/common/shell.cc]
```
void Shell::RunEngine(RunConfiguration run_configuration) {
  RunEngine(std::move(run_configuration), nullptr);
}

void Shell::RunEngine(
    RunConfiguration run_configuration,
    const std::function<void(Engine::RunStatus)>& result_callback) {
  auto result = [platform_runner = task_runners_.GetPlatformTaskRunner(),
                 result_callback](Engine::RunStatus run_result) {
    if (!result_callback) {
      return;
    }
    platform_runner->PostTask(
        [result_callback, run_result]() { result_callback(run_result); });
  };
  FML_DCHECK(is_set_up_);
  FML_DCHECK(task_runners_.GetPlatformTaskRunner()->RunsTasksOnCurrentThread());

  fml::TaskRunner::RunNowOrPostTask(
      task_runners_.GetUITaskRunner(),
      fml::MakeCopyable(
          [run_configuration = std::move(run_configuration),
           weak_engine = weak_engine_, result]() mutable {
            if (!weak_engine) {
              FML_LOG(ERROR)
                  << "Could not launch engine with configuration - no engine.";
              result(Engine::RunStatus::Failure);
              return;
            }
            auto run_result = weak_engine->Run(std::move(run_configuration));
            if (run_result == flutter::Engine::RunStatus::Failure) {
              FML_LOG(ERROR) << "Could not launch engine with configuration.";
            }

            result(run_result);
          }));
}
```
这里会把引擎初始化Post到UITask中运行weak_engine->Run(std::move(run_configuration))

