# Android 架构

## 1.1 架构图

### 1.1.1 系统架构

![architecture](imgs\01\ape_fwk_all.png)

* **应用框架**

  最常被应用开发者使用。作为硬件开发者，应非常了解开发者 API，因为很多此类 API 都可以直接映射到底层 HAL 接口，
  并可提供与实现驱动程序相关的实用信息。

* **Binder IPC**

  Binder 进程间通信 (IPC) 机制允许应用框架跨越进程边界并调用 Android 系统服务代码，这使得高级框架 API 能与 Android 系统服务进行交互。
  在应用框架级别，开发者无法看到此类通信的过程，但一切似乎都在“按部就班地运行”。

* **系统服务**

  系统服务是专注于特定功能的模块化组件，例如窗口管理器、搜索服务或通知管理器。 
  应用框架 API 所提供的功能可与系统服务通信，以访问底层硬件。Android 包含两组服务：“系统”（诸如窗口管理器和通知管理器之类的服务）和“媒体”（与播放和录制媒体相关的服务）。

* **硬件抽象层(HAL)**

  HAL 可定义一个标准接口以供硬件供应商实现，这可让 Android 忽略较低级别的驱动程序实现。
  借助 HAL，您可以顺利实现相关功能，而不会影响或更改更高级别的系统。HAL 实现会被封装成模块，并会由 Android 系统适时地加载。

* **Linux 内核**

  开发设备驱动程序与开发典型的 Linux 设备驱动程序类似。Android 使用的 Linux 内核版本包含几个特殊的补充功能，
  例如：Low Memory Killer（一种内存管理系统，可更主动地保留内存）、唤醒锁定（一种 [`PowerManager`](https://developer.android.com/reference/android/os/PowerManager.html) 系统服务）、Binder IPC 驱动程序以及对移动嵌入式平台来说非常重要的其他功能。这些补充功能主要用于增强系统功能，不会影响驱动程序开发。


### 1.1.1 软件堆栈

![软件堆栈](imgs\01\android-stack_2x.png)

* **Linux 内核**

	Android 平台的基础是 Linux 内核。例如，Android Runtime (ART) 依靠 Linux 内核来执行底层功能，例如线程和低层内存管理。  
	使用 Linux 内核可让 Android 利用主要安全功能，并且允许设备制造商为著名的内核开发硬件驱动程序。

* **硬件抽象层 (HAL)**

	硬件抽象层 (HAL) 提供标准界面，向更高级别的 Java API 框架显示设备硬件功能。HAL 包含多个库模块，
	其中每个模块都为特定类型的硬件组件实现一个界面，例如相机或蓝牙模块。当框架 API 要求访问设备硬件时，Android 系统将为该硬件组件加载库模块。

* **Android Runtime**

	对于运行 Android 5.0（API 级别 21）或更高版本的设备，每个应用都在其自己的进程中运行，
	并且有其自己的 Android Runtime (ART) 实例。ART 编写为通过执行 DEX 文件在低内存设备上运行多个虚拟机，
	DEX 文件是一种专为 Android 设计的字节码格式，经过优化，使用的内存很少。
	编译工具链（例如 Jack）将 Java 源代码编译为 DEX 字节码，使其可在 Android 平台上运行。  
	ART 的部分主要功能包括：
	* 预先 (AOT) 和即时 (JIT) 编译
	* 优化的垃圾回收 (GC)
	* 在 Android 9（API 级别 28）及更高版本的系统中，支持将应用软件包中的 Dalvik Executable 格式 (DEX) 文件转换为更紧凑的机器代码。
	更好的调试支持，包括专用采样分析器、详细的诊断异常和崩溃报告，并且能够设置观察点以监控特定字段
	* 在 Android 版本 5.0（API 级别 21）之前，Dalvik 是 Android Runtime。如果您的应用在 ART 上运行效果很好，
	那么它应该也可在 Dalvik 上运行，但反过来不一定。

	Android 还包含一套核心运行时库，可提供 Java API 框架所使用的 Java 编程语言中的大部分功能，包括一些 Java 8 语言功能。

* **原生 C/C++ 库**

	许多核心 Android 系统组件和服务（例如 ART 和 HAL）构建自原生代码，需要以 C 和 C++ 编写的原生库。
	Android 平台提供 Java 框架 API 以向应用显示其中部分原生库的功能。例如，您可以通过 Android 框架的 Java OpenGL API 访问 OpenGL ES，以支持在应用中绘制和操作 2D 和 3D 图形。

	如果开发的是需要 C 或 C++ 代码的应用，可以使用 Android NDK 直接从原生代码访问某些原生平台库。

* **Java API 框架**

	通过以 Java 语言编写的 API 使用 Android OS 的整个功能集。这些 API 形成创建 Android 应用所需的构建块，
	它们可简化核心模块化系统组件和服务的重复使用，包括以下组件和服务：

	* 丰富、可扩展的视图系统，可用以构建应用的 UI，包括列表、网格、文本框、按钮甚至可嵌入的网络浏览器
	* 资源管理器，用于访问非代码资源，例如本地化的字符串、图形和布局文件
	* 通知管理器，可让所有应用在状态栏中显示自定义提醒
	* Activity 管理器，用于管理应用的生命周期，提供常见的导航返回栈
	* 内容提供程序，可让应用访问其他应用（例如“联系人”应用）中的数据或者共享其自己的数据  
	
开发者可以完全访问 Android 系统应用使用的框架 API。

* **系统应用**

	Android 随附一套用于电子邮件、短信、日历、互联网浏览和联系人等的核心应用。平台随附的应用与用户可以选择安装的应用一样，没有特殊状态。
	因此第三方应用可成为用户的默认网络浏览器、短信 Messenger 甚至默认键盘（有一些例外，例如系统的“设置”应用）。

	系统应用可用作用户的应用，以及提供开发者可从其自己的应用访问的主要功能。例如，如果您的应用要发短信，您无需自己构建该功能，
	可以改为调用已安装的短信应用向您指定的接收者发送消息。

## 1.2 HAL 接口定义语言(HIDL)

Android 8.0 及更高版本提供了一个稳定的新供应商接口，因此设备制造商可以访问 Android 代码中特定于硬件的部分，这样一来，
设备制造商只需更新 Android 操作系统框架，即可跳过芯片制造商直接提供新的 Android 版本:

![Android_Updated](imgs\01\treble_blog_after.png)


## 1.3 相关资源补充
要详细了解 Android 架构，请参阅以下部分：

* [HAL 类型](https://source.android.google.cn/devices/architecture/hal-types)：

	提供了关于绑定式 HAL、直通 HAL、Same-Process (SP) HAL 和旧版 HAL 的说明。

* [HIDL（一般信息）](https://source.android.com/devices/architecture/hidl/index.html)：

	包含与 HAL 和其用户之间的接口有关的一般信息。

* [HIDL (C/C++)](https://developer.android.google.cn/ndk/index.html)：

	包含关于为 HIDL 接口创建 C++ 实现的详情。

* [HIDL (Java)](https://developer.android.google.cn/reference/packages.html)：
	
	包含关于 HIDL 接口的 Java 前端的详情。

* [ConfigStore HAL](https://source.android.com/devices/architecture/configstore/index.html)：
	
	提供了关于可供访问用于配置 Android 框架的只读配置项的 API 的说明。

* [设备树叠加层](https://source.android.com/devices/architecture/dto/index.html)：
	
	提供了关于在 Android 中使用设备树叠加层 (DTO) 的详情。

* [供应商原生开发套件 (VNDK)](https://source.android.com/devices/architecture/vintf/index.html)：
	
	提供了关于一组可供实现供应商 HAL 的供应商专用库的说明。

* [供应商接口对象 (VINTF)](https://source.android.com/devices/architecture/vintf/index.html)：
	
	提供了关于收集设备的相关信息并通过可查询 API 提供这些信息的对象的说明。

