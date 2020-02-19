# HIDL C++

Android O 对 Android 操作系统的架构重新进行了设计，以在独立于设备的 Android 平台与特定于设备和供应商的代码之间定义清晰的接口。Android 已经以 HAL 接口的形式（在 `hardware/libhardware` 中定义为 C 标头）定义了许多此类接口。

HIDL 将这些 HAL 接口替换为稳定的带版本接口，它们可以是采用 C++（如下所述）或 [Java](https://source.android.com/devices/architecture/hidl-java) 的客户端和服务器端 HIDL 接口。

## 概览



### 客户端和服务器实现

HIDL 接口具有客户端和服务器实现：

- HIDL 接口的**客户端**实现是指通过在该接口上调用方法来使用该接口的代码。
- **服务器**实现是指 HIDL 接口的实现，它可接收来自客户端的调用并返回结果（如有必要）。

在从 `libhardware` HAL 转换为 HIDL HAL 的过程中，HAL 实现成为服务器，而调用 HAL 的进程则成为客户端。默认实现可提供直通和绑定式 HAL，并可能会随着时间而发生变化：

![treble_cpp_legacy_hal_progression](imgs/06/treble_cpp_legacy_hal_progression.png)

### 创建 HAL 客户端

首先将 HAL 库添加到 makefile 中：

- Make：`LOCAL_SHARED_LIBRARIES += android.hardware.nfc@1.0`
- Soong：`shared_libs: [ …, android.hardware.nfc@1.0 ]`

接下来，添加 HAL 头文件：

```
#include <android/hardware/nfc/1.0/IFoo.h>
…
// in code:
sp<IFoo> client = IFoo::getService();
client->doThing();
```

### 创建 HAL 服务器

要创建 HAL 实现，必须具有表示 HAL 的 `.hal` 文件并已在 `hidl-gen` 上使用 `-Lmakefile` 或 `-Landroidbp` 为 HAL 生成 makefile（`./hardware/interfaces/update-makefiles.sh` 会为内部 HAL 文件执行这项操作，这是一个很好的参考）。从 `libhardware` 通过 HAL 传输时，可以使用 c2hal 轻松完成许多此类工作。

```
PACKAGE=android.hardware.nfc@1.0
LOC=hardware/interfaces/nfc/1.0/default/
m -j hidl-gen
hidl-gen -o $LOC -Lc++-impl -randroid.hardware:hardware/interfaces \
    -randroid.hidl:system/libhidl/transport $PACKAGE
hidl-gen -o $LOC -Landroidbp-impl -randroid.hardware:hardware/interfaces \
    -randroid.hidl:system/libhidl/transport $PACKAGE
```

`/(system|vendor|...)/lib(64)?/hw/android.hardware.package@3.0-impl($OPTIONAL_IDENTIFIER).so` 下），其中 `$OPTIONAL_IDENTIFIER` 是一个标识直通实现的字符串。直通模式要求会通过上述命令自动满足，这些命令也会创建 `android.hardware.nfc@1.0-impl` 目标，但是可以使用任何扩展。例如，`android.hardware.nfc@1.0-impl-foo` 就是使用 `-foo` 区分自身。

接下来，使用相应功能填写存根并设置守护进程。守护进程代码（支持直通）示例：

```
#include <hidl/LegacySupport.h>

int main(int /* argc */, char* /* argv */ []) {
    return defaultPassthroughServiceImplementation<INfc>("nfc");
}
```

`defaultPassthroughServiceImplementation` 将对提供的 `-impl` 库执行 `dlopen()` 操作，并将其作为绑定式服务提供。守护进程代码（对于纯绑定式服务）示例：

```
int main(int /* argc */, char* /* argv */ []) {
    // This function must be called before you join to ensure the proper
    // number of threads are created. The threadpool will never exceed
    // size one because of this call.
    ::android::hardware::configureRpcThreadpool(1 /*threads*/, true /*willJoin*/);

    sp nfc = new Nfc();
    const status_t status = nfc->registerAsService();
    if (status != ::android::OK) {
        return 1; // or handle error
    }

    // Adds this thread to the threadpool, resulting in one total
    // thread in the threadpool. We could also do other things, but
    // would have to specify 'false' to willJoin in configureRpcThreadpool.
    ::android::hardware::joinRpcThreadpool();
    return 1; // joinRpcThreadpool should never return
}
```

此守护进程通常存在于 `$PACKAGE + "-service-suffix"`（例如 `android.hardware.nfc@1.0-service`）中，但也可以位于任何位置。HAL 的特定类的 [sepolicy](https://source.android.com/security/selinux/device-policy) 是属性 `hal_`（例如 `hal_nfc)`）。必须将此属性应用到运行特定 HAL 的守护进程（如果同一进程提供多个 HAL，则可以将多个属性应用到该进程）。

## 软件包

略



## 接口

略



## 数据类型

略



## 函数

略

