# 引导加载程序

引导加载程序是供应商专有的映像，负责在设备上启动内核。它会监护设备状态，负责初始化[可信执行环境 (TEE)](https://source.android.com/security/trusty) 以及绑定其信任根。

引导加载程序由许多部分组成，包括启动画面。要开始启动，引导加载程序可能会直接将一个新映像刷写到相应的分区中，也可能会使用 `recovery` 开始重新刷写过程，该过程与 OTA 的操作过程一致。一些设备制造商会创建多部分引导加载程序，然后将它们组合到一个 bootloader.img 文件中。在刷写时，引导加载程序会提取各个引导加载程序并刷写所有这些引导加载程序。

最重要的是，引导加载程序会在将执行工作移到内核之前先验证 boot 分区和 recovery 分区的完整性，并显示[启动状态](https://source.android.com/security/verifiedboot/verified-boot#boot_state)部分中指定的警告。

## 启动原因

引导加载程序使用专用的硬件和内存资源来确定设备重新启动的原因，然后将 `androidboot.bootreason=` 添加到用于启动设备的 Android 内核命令行中，以传达这一决定。然后，`init` 会转换此命令行，使其传播到 Android 属性 `bootloader_boot_reason_prop` (`ro.boot.bootreason`) 中。

### 启动原因规范

之前的 Android 版本中指定的启动原因格式如下：不使用空格，全部为小写字母，只有非常少的要求（例如报告 `kernel_panic`、`watchdog`、`cold`/`warm`/`hard`），并且允许其他特殊原因。这种宽松的规范导致出现了成百上千个自定义启动原因字符串（有时毫无意义），进而造成了无法管理的情况。到目前最新的 Android 版本发布之前，引导加载程序提交的近乎无法解析或毫无意义的内容急剧增加已经为 `bootloader_boot_reason_prop` 造成了合规性问题。

在开发 Android 9 版本中，Android 团队发现旧的 `bootloader_boot_reason_prop` 中内容会急剧增加，并且无法在系统运行时重写。因此，要对启动原因规范进行任何改进，都必须与引导加载程序开发者进行互动交流，并对现有系统进行调整。为此，Android 团队采取了以下措施：

- 与引导加载程序开发者互动交流，鼓励他们：
  - 向 `bootloader_boot_reason_prop` 提供规范、可解析且可识别的原因。
  - 向 `system/core/bootstat/bootstat.cpp` `kBootReasonMap` 列表添加内容。
- 添加受控且可在系统运行时重写的 `system_boot_reason_prop` (`sys.boot.reason`) 源代码。只有少量的系统应用（如 `bootstat` 和 `init`）可重写此属性，不过，所有应用都可以通过获得 sepolicy 权限来读取它。
- 将启动原因告知用户，让他们等到用户数据装载完毕后再信任系统启动原因属性 `system_boot_reason_prop` 中的内容。

为什么要等这么久？虽然 `bootloader_boot_reason_prop` 在启动过程的早期阶段就已可用，但 Android 安全政策根据需要对其进行了屏蔽，因为它表示不准确、不可解析且不合规范的信息。大多数情况下，只有对启动系统有深入了解的开发者才需要访问这些信息。只有在用户数据装载完毕**之后**，才可以通过 `system_boot_reason_prop` 准确可靠地提取经过优化、可解析且合乎规范的启动原因 API。具体而言：

- 在用户数据装载完毕**之前**，`system_boot_reason_prop` 将包含 `bootloader_boot_reason_prop` 中的值。
- 在用户数据装载完毕**之后**，可以更新 `system_boot_reason_prop`，以使其符合要求或报告更准确的信息。

出于上述原因，Android 9 延长了可以正式获取启动原因之前需要等待的时间段，将其从启动时立即准确无误的状态（使用 `bootloader_boot_reason_prop`）更改为仅在用户数据装载完毕之后才可用（使用 `system_boot_reason_prop`）。

Bootstat 逻辑依赖于信息更丰富且合规的 `bootloader_boot_reason_prop`。当该属性使用可预测的格式时，能够提高所有受控重新启动和关机情况的准确性，从而优化和扩展 `system_boot_reason_prop` 的准确性和含义。

### 规范化启动原因格式

在 Android 9 中，`bootloader_boot_reason_prop` 的规范化启动原因格式使用以下语法：

```xml
<reason>,<subreason>,<detail>…
```

格式设置规则如下：

- 小写
- 无空格（可使用下划线）
- 全部为可打印字符
- 以英文逗号分隔的`reason`、`subreason`，以及一个或多个`detail`
  - 必需的 `reason`，表示设备为什么必须重新启动或关机且优先级最高的原因。
  - 选用的 `subreason`，表示设备为什么必须重新启动或关机的简短摘要（或重新启动设备/将设备关机的人员）。
  - 一个或多个选用的 `detail` 值。`detail` 可以指向某个子系统，以协助确定是哪个具体系统导致了 `subreason`。您可以指定多个 `detail` 值，这些值通常应按照重要程度排序。不过，也可以报告多个具有同等重要性的 `detail` 值。

如果 `bootloader_boot_reason_prop` 为空值，则会被视为非法（因为这会允许其他代理在事后添加启动原因）。

#### 原因要求

为 `reason`（第一个跨度，位于终止符或英文逗号之前）指定的值必须是以下集合（分为内核原因、强原因和弱原因）之一：

- 内核集：
  - "`watchdog"`
  - `"kernel_panic"`
- 强集：
  - `"recovery"`
  - `"bootloader"`
- 弱集：
  - `"cold"`：通常表示完全重置所有设备，包括内存。
  - `"hard"`：通常表示硬件重置了状态，并且 `ramoops` 应保留持久性内容。
  - `"warm"`：通常表示内存和设备保持某种状态，并且 `ramoops`（请参阅内核中的 `pstore` 驱动程序）后备存储空间包含持久性内容。
  - `"shutdown"`
  - `"reboot"`：通常意味着 `ramoops` 状态和硬件状态未知。该值是与 `cold`、`hard` 和 `warm` 一样的通用值，可提供关于设备重置深度的提示。

引导加载程序必须提供内核集或弱集 `reason`，强烈建议引导加载程序提供 `subreason`（如果可以确定的话）。例如，电源键长按（无论是否有 `ramoops` 备份）的启动原因为 `"reboot,longkey"`。

第一个跨度 `reason` 不能是任何 `subreason` 或 `detail` 的组成部分。不过，由于用户空间无法产生内核集原因，因此可能会在弱集原因之后重复使用 `"watchdog"` 以及源代码的详细信息（例如 `"reboot,watchdog,service_manager_unresponsive"` 或 `"reboot,software,watchdog"`）。

启动原因应该无需专家级内部知识即可解读，并且（或者）应该能让人看懂并提供直观报告。示例：`"shutdown,vbxd"`（糟糕）、`"shutdown,uv"`（较好）、`"shutdown,undervoltage"`（首选）。

#### “原因-子原因”组合

Android 保留了一组 `reason`-`subreason` 组合，在正常使用情况下不应过量使用这些组合；不过，如果组合能准确反映相关状况，则可根据具体情况加以使用。保留组合的示例包括：

- `"reboot,userrequested"`
- `"shutdown,userrequested"`
- `"shutdown,thermal"`（来自 `thermald`）
- `"shutdown,battery"`
- `"shutdown,battery,thermal"`（来自 `BatteryStatsService`）
- `"reboot,adb"`
- `"reboot,shell"`
- `"reboot,bootloader"`
- `"reboot,recovery"`

如需更多详细信息，请参阅 `system/core/bootstat/bootstat.cpp` 中的 `kBootReasonMap` 以及 Android 源代码库中的关联 git 变更日志记录。

### 报告启动原因

所有启动原因（无论是来自引导加载程序还是记录在规范化启动原因中）都必须记录在 `system/core/bootstat/bootstat.cpp` 的 `kBootReasonMap` 部分中。`kBootReasonMap` 列表包含各种合规原因和不合规的旧版原因。引导加载程序开发者应在此处仅登记新的合规原因（除非产品已发货且无法更改，否则不应登记不合规的原因）。

> **注意**：虽然 `system/core/bootstat/bootstat.cpp` 包含一个 `kBootReasonMap` 部分，其中列出了大量旧版原因，但这些原因的存在并不意味着 `reason` 字符串已获准使用。该列表的一个子集内列出了合规原因；随着引导加载程序开发者不断登记更多合规原因并加以说明，这个子集预计将不断增大。

强烈建议使用 `system/core/bootstat/bootstat.cpp` 中现有的合规条目，如果要使用不合规字符串，要先对其加以限制。请参阅以下指导原则：

- **允许**从引导加载程序中报告 `"kernel_panic"`，因为 `bootstat` 或许能检查 `kernel_panic signatures` 的 `ramoops`，以便将子原因优化为规范的 `system_boot_reason_prop`。
- **不允许**从引导加载程序中以 `kBootReasonMap`（如 `"panic")`）的形式报告不合规的字符串，因为这最终将导致无法优化 `reason`。

例如，如果 `kBootReasonMap` 包含 `"wdog_bark"`，则引导加载程序开发者应采取以下措施：

- 更改为 `"watchdog,bark"`，并将其添加到 `kBootReasonMap` 中的列表内。
- 考虑 `"bark"` 对于不熟悉该技术的人来说意味着什么，并确定是否存在更有意义的 `subreason`。

### 验证启动原因合规性

目前，对于引导加载程序可能提供的所有启动原因，Android 没有提供能够准确触发或检查这些原因的主动 CTS 测试；合作伙伴仍然可以尝试运行被动测试来确定兼容性。

因此，要实现引导加载程序合规性，引导加载程序开发者需要自愿遵循上述规则和准则的精神。我们会敦促此类开发者为 AOSP（特别是 `system/core/bootstat/bootstat.cpp`）做贡献，并将这个机会作为一个讨论启动原因问题的论坛。

## 启动映像头文件

从 Android 9 起，启动映像头文件开始包含一个用于指示头文件版本的字段。引导加载程序必须检查该头文件版本字段，并相应地解析头文件。通过对启动映像头文件进行版本编号，可在将来对头文件进行修改，同时保持向后兼容性。

所有搭载 Android 9 的设备都必须使用启动头文件版本 1。

### 启动映像头文件更改



### 实现

### 验证

### 在启动映像中添加 DTB

### 启动映像头文件版本 2





## 分局布局



## 分局和映像



## 产品分区



## ODM分区



## 恢复映像



## 刷写和更新



## 解锁和Trusty



## 用户空间中的fastboot

