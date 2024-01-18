# Flutter Platform Channel通信流程

Flutter如何和原生端通信，通信过程中是否涉及到数据拷贝、线程切换以及Dart VM和Native，Native和ART通信等等诸多问题。这篇文章就从源码角度分析解决这几个问题。<br>
<br>

## 一、Dart 发送消息

Dart 通过Platform Channel发送消息：<br>
`await Platform.platform.invokeMethod(_getAppInfo)`

这里的Platform.platform是Dart和原生端定义的通信协议：`static const platform = MethodChannel("xxx");`<br>

运行在isolate的UI线程中，具体的源码流程如下：

### 1.1 Dart Send Msg

platform_channel.dart<br>

```
 @optionalTypeArgs
  Future<T?> invokeMethod<T>(String method, [ dynamic arguments ]) {
    return _invokeMethod<T>(method, missingOk: false, arguments: arguments);
  }
```

### 1.2 Dart Call Native

通过dispatcher调用__sendPlatformMessage，绑定到Native：PlatformConfigurationNativeApi::SendPlatformMessage
platform_dispatcher.dart<br>

```
 String? _sendPlatformMessage(String name, PlatformMessageResponseCallback? callback, ByteData? data) =>
      __sendPlatformMessage(name, callback, data);

  @FfiNative<Handle Function(Handle, Handle, Handle)>('PlatformConfigurationNativeApi::SendPlatformMessage')
  external static String? __sendPlatformMessage(String name, PlatformMessageResponseCallback? callback, ByteData? data);
```


### 1.3 Native层SendPlatformMessage

platform_configuration.cc
```
Dart_Handle PlatformConfigurationNativeApi::SendPlatformMessage(
    const std::string& name,
    Dart_Handle callback,
    Dart_Handle data_handle) {
  UIDartState* dart_state = UIDartState::Current();

  if (!dart_state->platform_configuration()) {
    return tonic::ToDart(
        "SendPlatformMessage only works on the root isolate, see "
        "SendPortPlatformMessage.");
  }

  fml::RefPtr<PlatformMessageResponse> response;
  if (!Dart_IsNull(callback)) {
    response = fml::MakeRefCounted<PlatformMessageResponseDart>(
        tonic::DartPersistentValue(dart_state, callback),
        dart_state->GetTaskRunners().GetUITaskRunner(), name);
  }

  return HandlePlatformMessage(dart_state, name, data_handle, response);
}
```

从Dart层到Native有一次数据的Copy，Dart层是ByteData? data，这个是isolate里创建的对象，而Native层属于非虚拟机的内存对象，这里会有一次数据拷贝。
具体源码：
```
Dart_Handle HandlePlatformMessage(
    UIDartState* dart_state,
    const std::string& name,
    Dart_Handle data_handle,
    const fml::RefPtr<PlatformMessageResponse>& response) {
  if (Dart_IsNull(data_handle)) {
    return dart_state->HandlePlatformMessage(
        std::make_unique<PlatformMessage>(name, response));
  } else {
    tonic::DartByteData data(data_handle);
    const uint8_t* buffer = static_cast<const uint8_t*>(data.data());
    return dart_state->HandlePlatformMessage(std::make_unique<PlatformMessage>(
        name, fml::MallocMapping::Copy(buffer, data.length_in_bytes()),
        response));
  }
}
```

而Dart_Handle是跨越Dart和Native的桥梁，具体定义在dart_api.h<br>
`typedef struct _Dart_Handle* Dart_Handle;`

_Dart_Handle的具体定义，这里有一个回答：<br>
**It does not have a definition - it's an opaque pointer. In reality it points to dart::LocalHandle. See Api::UnwrapHandle in dart_api_impl.cc.**


在调用到ui_dart_state.cc的HandlePlatformMessage<br>
```
Dart_Handle UIDartState::HandlePlatformMessage(
    std::unique_ptr<PlatformMessage> message) {
  if (platform_configuration_) {
    platform_configuration_->client()->HandlePlatformMessage(
        std::move(message));
  } else {
    std::shared_ptr<PlatformMessageHandler> handler =
        platform_message_handler_.lock();
    if (handler) {
      handler->HandlePlatformMessage(std::move(message));
    } else {
      return tonic::ToDart(
          "No platform channel handler registered for background isolate.");
    }
  }

  return Dart_Null();
}
```

## 二、Dart VM