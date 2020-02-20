# 配置

Android 10 因 ConfigStore HAL 内存耗用量高且难以使用而将其弃用，并用系统属性替换了这个 HAL。在 Android 10 中：

- ConfigStore 使用编译标记在供应商分区中存储配置值，系统分区中的服务使用 HIDL 访问这些值（在 Android 9 中也是如此）。
- 系统属性使用 `PRODUCT_DEFAULT_PROPERTY_OVERRIDES` 在供应商分区的 `default.prop` 中存储系统属性，而该服务使用 `sysprop` 读取这些属性。

ConfigStore HAL 保留在 AOSP 中以支持旧版供应商分区。在搭载 Android 10 的设备上，`surfaceflinger` 首先读取系统属性；如果没有为 `SurfaceFlingerProperties.sysprop` 中的配置项定义任何系统属性，则 `surfaceflinger` 会回退到 ConfigStore HAL。

## 系统属性API



## 配置文件架构API



## ConfigStore HAL





