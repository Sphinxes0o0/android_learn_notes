# 设备树叠加层

设备树 (DT) 是用于描述“不可发现”硬件的命名节点和属性的数据结构。操作系统（例如在 Android 中使用的 Linux 内核）会使用 DT 来支持 Android 设备使用的各种硬件配置。硬件供应商会提供自己的 DT 源文件，接下来 Linux 会将这些文件编译到引导加载程序使用的设备树 Blob (DTB) 文件中。

[设备树叠加层](https://lkml.org/lkml/2012/11/5/615) (DTO) 可让主要的设备树 Blob (DTB) 叠加在设备树上。使用 DTO 的引导加载程序可以维护系统芯片 (SoC) DT，并动态叠加针对特定设备的 DT，从而向树中添加节点并对现有树中的属性进行更改。

本页详细介绍了引导加载程序加载 DT 的典型工作流程，并列出了常见的 DT 术语。本节的其他页面介绍了如何[为 DTO 实现引导加载程序支持](https://source.android.com/devices/architecture/dto/implement)，如何[编译](https://source.android.com/devices/architecture/dto/compile)、验证和[优化 DTO 实现](https://source.android.com/devices/architecture/dto/optimize)以及如何[使用多个 DT](https://source.android.com/devices/architecture/dto/multiple)。您还可以获取关于 [DTO 语法](https://source.android.com/devices/architecture/dto/syntax)和必需的 [DTO/DTBO 分区格式](https://source.android.com/devices/architecture/dto/partitions)的详细信息。

## 概览





## 实现DTO





## DTO语法





## 编译和验证





## 使用多个DT



## DTB/DTBO分区格式



## 优化DTO

