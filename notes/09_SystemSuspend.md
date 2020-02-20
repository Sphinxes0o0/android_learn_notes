# SystemSuspend 服务

在 Android 9 及更低版本中，[libsuspend](https://android.googlesource.com/platform/system/core/+/pie-dev/libsuspend/autosuspend_wakeup_count.cpp#65) 中有一个负责发起系统挂起的线程。Android 10 在 SystemSuspend HIDL 服务中引入了等效功能。此服务位于系统映像中，由 Android 平台提供。`libsuspend` 的逻辑基本保持不变，除了阻止系统挂起的每个用户空间进程都需要与 SystemSuspend 进行通信。