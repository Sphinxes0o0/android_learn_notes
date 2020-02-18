# HAL 

## 3.0 旧版HAL 

省略



## 3.1 HAL 类型

在 Android 8.0 及更高版本中，较低级别的层已重新编写以采用更加模块化的新架构。运行 Android 8.0 或更高版本的设备必须支持使用 HIDL 语言编写的 HAL，下面列出了一些例外情况。这些 HAL 可以是绑定式 HAL 也可以是直通式 HAL:

* **绑定式 HAL。** 以 HAL 接口定义语言 (HIDL) 表示的 HAL。这些 HAL 取代了早期 Android 版本中使用的传统 HAL 和旧版 HAL。在绑定式 HAL 中，Android 框架和 HAL 之间通过 Binder 进程间通信 (IPC) 调用进行通信。所有在推出时即搭载了 Android 8.0 或更高版本的设备都必须只支持绑定式 HAL。
* **直通式 HAL。** 以 HIDL 封装的传统 HAL 或旧版 HAL。这些 HAL 封装了现有的 HAL，可在绑定模式和 Same-Process（直通）模式下使用。升级到 Android 8.0 的设备可以使用直通式 HAL。



### 3.1.1 模型要求

|           设备            |                            直通式                            |                            绑定式                            |
| :-----------------------: | :----------------------------------------------------------: | :----------------------------------------------------------: |
|  搭载 Android 8.0 的设备  | [直通式 HAL](https://source.android.com/devices/architecture/hal-types/#passthrough) 中列出的 HAL 必须为直通式。 |  所有其他 HAL 均为绑定式（包括作为供应商扩展程序的 HAL）。   |
| 升级到 Android 8.0 的设备 | [直通式 HAL](https://source.android.com/devices/architecture/hal-types/#passthrough) 中列出的 HAL 必须为直通式（供应商映像提供的所有其他 HAL 既可以在直通模式下使用，也可以在绑定模式下使用。在完全符合 Treble 标准的设备中，所有 HAL 都必须为绑定式 HAL。） | [绑定式 HAL](https://source.android.com/devices/architecture/hal-types/#binderized) 中列出的 HAL 必须为绑定式。 |



### 3.1.2 绑定式HAL

Android 要求所有 Android 设备（无论是搭载 Android O 的设备还是升级到 Android O 的设备）上的下列 HAL 均为绑定式：

- `android.hardware.biometrics.fingerprint@2.1`。取代 Android 8.0 中已不存在的 `fingerprintd`。
- `android.hardware.configstore@1.0`。Android 8.0 中的新 HAL。
- `android.hardware.dumpstate@1.0`。此 HAL 提供的原始接口可能无法继续使用，并且已更改。因此，`dumpstate_board` 必须在指定的设备上重新实现（这是一个可选的 HAL）。
- `android.hardware.graphics.allocator@2.0`。在 Android 8.0 中，此 HAL 必须为绑定式，因此无需在可信进程和不可信进程之间分享文件描述符。
- `android.hardware.radio@1.0`。取代由存活于自身进程中的 `rild` 提供的接口。
- `android.hardware.usb@1.0`。Android 8.0 中的新 HAL。
- `android.hardware.wifi@1.0`。Android 8.0 中的新 HAL，可取代此前加载到 `system_server` 的旧版 WLAN HAL 库。
- `android.hardware.wifi.supplicant@1.0`。在现有 `wpa_supplicant` 进程之上的 HIDL 接口

> ★**注意**：Android 提供的以下 HIDL 接口将一律在绑定模式下使用：`android.frameworks.*`、`android.system.*` 和 `android.hidl.*`（不包括下文所述的 `android.hidl.memory@1.0`）。



### 3.1.3 直通式HAL

Android 要求所有 Android 设备（无论是搭载 Android O 的设备还是升级到 Android O 的设备）上的下列 HAL 均在直通模式下使用：

- `android.hardware.graphics.mapper@1.0`。将内存映射到其所属的进程中。
- `android.hardware.renderscript@1.0`。在同一进程中传递项（等同于 `openGL`）。

上方未列出的所有 HAL 在搭载 Android O 的设备上都必须为绑定式。





### 3.1.4 Same-Process HAL

Same-Process HAL (SP-HAL) 一律在使用它们的进程中打开，其中包括未以 HIDL 表示的所有 HAL，以及那些**非**绑定式的 HAL。SP-HAL 集的成员只能由 Google 控制，这一点没有例外。

SP-HAL 包括以下 HAL：

- `openGL`
- `Vulkan`
- `android.hidl.memory@1.0`（由 Android 系统提供，一律为直通式）
- `android.hardware.graphics.mapper@1.0`。
- `android.hardware.renderscript@1.0`





### 3.1.5 传统 HAL 和旧版 HAL

传统 HAL（在 Android 8.0 中已弃用）是指与具有特定名称及版本号的应用二进制接口 (ABI) 标准相符的接口。大部分 Android 系统接口（[相机](https://android.googlesource.com/platform/hardware/libhardware/+/master/include/hardware/camera3.h)、[音频](https://android.googlesource.com/platform/hardware/libhardware/+/master/include/hardware/audio.h)和[传感器](https://android.googlesource.com/platform/hardware/libhardware/+/master/include/hardware/sensors.h)等）都采用传统 HAL 形式（已在 [hardware/libhardware/include/hardware](https://android.googlesource.com/platform/hardware/libhardware/+/master/include/hardware) 下进行定义）。

旧版 HAL（也已在 Android 8.0 中弃用）是指早于传统 HAL 的接口。一些重要的子系统（WLAN、无线接口层和蓝牙）采用的就是旧版 HAL。虽然没有统一或标准化的方式来指明是否为旧版 HAL，但如果 HAL 早于 Android 8.0 而出现，那么这种 HAL 如果不是传统 HAL，就是旧版 HAL。有些旧版 HAL 的一部分包含在 [libhardware_legacy](https://android.googlesource.com/platform/hardware/libhardware_legacy/+/master) 中，而其他部分则分散在整个代码库中。



## 3.2 框架测试

省略



## 3.3 动态生命周期

Android 9 支持在不使用或不需要 Android 硬件子系统时动态关停这些子系统。例如，如果用户未使用 WLAN，WLAN 子系统就不应占用内存、耗用电量或使用其他系统资源。早期版本的 Android 中，在 Android 手机启动的整个期间，Android 设备上的 HAL/驱动程序都会保持开启状态。

实现动态关停涉及连接数据流以及执行动态进程，下文对此进行了详细介绍。



### 3.3.1 对HAL定义所做的更改

要实现动态关停，需要有关于哪些进程为哪些 HAL 接口提供服务的信息（此类信息之后在其他情况中也可能很有用），还需要确保在设备启动时不启动进程，而且在进程退出后，直到系统再次请求启动它们之前，都不重新启动它们。

```bash
# some init.rc script associated with the HAL
service vendor.some-service-name /vendor/bin/hw/some-binary-service
    # init language extension, provides information of what service is served
    # if multiple interfaces are served, they can be specified one on each line
    interface android.hardware.light@2.0::ILight default
    # restarted if hwservicemanager dies
    # would also cause the hal to start early during boot if oneshot wasn't set
    class hal
    # will not be restarted if it exits until it is requested to be restarted
    oneshot
    # will only be started when requested
    disabled
    # ... other properties
```


### 3.3.2 对 init 和 hwservicemanager 所做的更改

要实现动态关停，还需要让 `hwservicemanager` 告知 `init` 启动所请求的服务。在 Android 9 中，`init` 包含三个额外的控制消息（例如，`ctl.start`）：`ctl.interface_start`、`ctl.interface_stop` 和 `ctl.interface_restart`。这些消息可用于指示 `init` 打开或关闭特定硬件接口。如果系统请求使用某个服务但该服务未注册，则 `hwservicemanager` 会请求启动该服务。不过，动态 HAL 不需要使用以上任何消息。


### 3.3.3 确定 HAL 退出

在 Android 9 中，必须手动确定 HAL 退出。对于 Android 10 及更高版本，还可以使用[自动生命周期](https://source.android.com/devices/architecture/hal/dynamic-lifecycle#automatic-lifecycles)确定 HAL 退出。

要实现动态关停，需要多个政策来决定何时启动和关停 HAL。如果 HAL 出于任何原因而决定退出，则当系统再次需要用到它时，它将使用以下信息和基础架构自动重新启动：HAL 定义中提供的信息，以及更改后的 `init` 和 `hwservicemanager` 提供的基础架构。 这可能会涉及多个不同的政策，包括：

- 如果有人对 HAL 调用关闭命令或类似的 API，则 HAL 可能会选择自行调用退出命令。此行为必须在相应的 HAL 接口中指定。
- HAL 可在任务完成后关停（记录在 HAL 文件中）。


### 3.3.4 自动生命周期

Android 10 为内核和 `hwservicemanager` 添加了更多支持，可让 HAL 在没有任何客户端时自动关闭。要使用此功能，请完成[对 HAL 定义所做的更改](https://source.android.com/devices/architecture/hal/dynamic-lifecycle#changes-HAL-definitions)部分的所有步骤，并执行以下操作：

- 使用`LazyServiceRegistrar`而不是成员函数`registerAsService`在 C++ 中注册服务，例如：

  ```
  // only one instance of LazyServiceRegistrar per process
  LazyServiceRegistrar registrar();
  registrar.registerAsService(myHidlService /* , "default" */);
  ```

- 验证 HAL 客户端是否仅在使用时保留对顶级 HAL（通过 `hwservicemanager` 注册的接口）的引用。为了避免出现延迟，如果该引用在继续执行的 hwbinder 线程上被丢弃，则客户端还应该在丢弃引用后调用 `IPCThreadState::self()->flushCommands()`，以确保 Binder 驱动程序在相关引用计数发生变化时收到通知。