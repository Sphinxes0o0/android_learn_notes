# 5. HIDL

HAL 接口定义语言（简称 HIDL，发音为“hide-l”）是用于指定 HAL 和其用户之间的接口的一种接口描述语言 (IDL)。HIDL 允许指定类型和方法调用（会汇集到接口和软件包中）。从更广泛的意义上来说，HIDL 是用于在可以独立编译的代码库之间进行通信的系统。

HIDL 旨在用于进程间通信 (IPC)。进程之间的通信经过 [*Binder*](https://source.android.com/devices/architecture/hidl/binder-ipc) 化。对于必须与进程相关联的代码库，还可以使用[直通模式](https://source.android.com/devices/architecture/hidl#passthrough)（在 Java 中不受支持）。

HIDL 可指定数据结构和方法签名，这些内容会整理归类到接口（与类相似）中，而接口会汇集到软件包中。尽管 HIDL 具有一系列不同的关键字，但 C++ 和 Java 程序员对 HIDL 的语法并不陌生。此外，HIDL 还使用 Java 样式的注释。

## 5.1 概览

HAL 接口定义语言（简称 HIDL，发音为“hide-l”）是用于指定 HAL 和其用户之间的接口的一种接口描述语言 (IDL)。HIDL 允许指定类型和方法调用（会汇集到接口和软件包中）。从更广泛的意义上来说，HIDL 是用于在可以独立编译的代码库之间进行通信的系统。

HIDL 旨在用于进程间通信 (IPC)。进程之间的通信经过 [*Binder*](https://source.android.com/devices/architecture/hidl/binder-ipc) 化。对于必须与进程相关联的代码库，还可以使用[直通模式](https://source.android.com/devices/architecture/hidl#passthrough)（在 Java 中不受支持）。

HIDL 可指定数据结构和方法签名，这些内容会整理归类到接口（与类相似）中，而接口会汇集到软件包中。尽管 HIDL 具有一系列不同的关键字，但 C++ 和 Java 程序员对 HIDL 的语法并不陌生。此外，HIDL 还使用 Java 样式的注释。

### 5.1.1 HIDL 设计

HIDL 的目标是，框架可以在无需重新构建 HAL 的情况下进行替换。HAL 将由供应商或 SOC 制造商构建，放置在设备的 `/vendor` 分区中，这样一来，框架就可以在其自己的分区中通过 OTA 进行替换，而无需重新编译 HAL。

HIDL 设计在以下方面之间保持了平衡：

- **互操作性**。在可以使用各种架构、工具链和编译配置来编译的进程之间创建可互操作的可靠接口。HIDL 接口是分版本的，发布后不得再进行更改。
- **效率**。HIDL 会尝试尽可能减少复制操作的次数。HIDL 定义的数据以 C++ 标准布局数据结构传递至 C++ 代码，无需解压，可直接使用。此外，HIDL 还提供共享内存接口；由于 RPC 本身有点慢，因此 HIDL 支持两种无需使用 RPC 调用的数据传输方法：共享内存和快速消息队列 (FMQ)。
- **直观**。通过仅针对 RPC 使用 `in` 参数，HIDL 避开了内存所有权这一棘手问题（请参阅 [Android 接口定义语言 (AIDL)](https://developer.android.com/guide/components/aidl.html)）；无法从方法高效返回的值将通过回调函数返回。无论是将数据传递到 HIDL 中以进行传输，还是从 HIDL 接收数据，都不会改变数据的所有权，也就是说，数据所有权始终属于调用函数。数据仅需要在函数被调用期间保留，可在被调用的函数返回数据后立即清除。

### 5.1.2 使用直通模式

要将运行早期版本的 Android 的设备更新为使用 Android O，您可以将惯用的（和旧版）HAL 封装在一个新 HIDL 接口中，该接口将在绑定式模式和同进程（直通）模式提供 HAL。这种封装对于 HAL 和 Android 框架来说都是透明的。

直通模式仅适用于 C++ 客户端和实现。运行早期版本的 Android 的设备没有用 Java 编写的 HAL，因此 Java HAL 自然而然经过 Binder 化。

#### 5.1.2.1 直通式标头文件

编译 `.hal` 文件时，除了用于 Binder 通信的标头之外，`hidl-gen` 还会生成一个额外的直通标头文件 `BsFoo.h`；此标头定义了会被执行 `dlopen` 操作的函数。由于直通式 HAL 在它们被调用的同一进程中运行，因此在大多数情况下，直通方法由直接函数调用（同一线程）来调用。`oneway` 方法在各自的线程中运行，因为它们不需要等待 HAL 来处理它们（这意味着，在直通模式下使用 `oneway` 方法的所有 HAL 对于线程必须是安全的）。

如果有一个 `IFoo.hal`，`BsFoo.h` 会封装 HIDL 生成的方法，以提供额外的功能（例如使 `oneway` 事务在其他线程中运行）。该文件类似于 `BpFoo.h`，不过，所需函数是直接调用的，并未使用 Binder 传递调用 IPC。未来，HAL 的实现**可能提供**多种实现结果，例如 FooFast HAL 和 FooAccurate HAL。在这种情况下，系统会针对每个额外的实现结果创建一个文件（例如 `PTFooFast.cpp` 和 `PTFooAccurate.cpp`）。

#### 5.1.2.2 Binder 化直通式 HAL

您可以将支持直通模式的 HAL 实现 Binder 化。如果有一个 HAL 接口 `a.b.c.d@M.N::IFoo`，系统会创建两个软件包：

- `a.b.c.d@M.N::IFoo-impl`。包含 HAL 的实现，并暴露函数 `IFoo* HIDL_FETCH_IFoo(const char* name)`。在旧版设备上，此软件包经过 `dlopen` 处理，且实现使用 `HIDL_FETCH_IFoo` 进行了实例化。您可以使用 `hidl-gen` 和 `-Lc++-impl` 以及 `-Landroidbp-impl` 来生成基础代码。
- `a.b.c.d@M.N::IFoo-service`。打开直通式 HAL，并将其自身注册为 Binder 化服务，从而使同一 HAL 实现能够同时以直通模式和 Binder 化模式使用。

如果有一个 `IFoo`，您可以调用 `sp IFoo::getService(string name, bool getStub)`，以获取对 `IFoo` 实例的访问权限。如果 `getStub` 为 True，则 `getService` 会尝试仅在直通模式下打开 HAL。如果 `getStub` 为 False，则 `getService` 会尝试找到 Binder 化服务；如果未找到，则它会尝试找到直通式服务。除了在 `defaultPassthroughServiceImplementation` 中，其余情况一律不得使用 `getStub` 参数。（搭载 Android O 的设备是完全 Binder 化的设备，因此不得在直通模式下打开服务。）

### 5.1.3 HIDL 语法

根据设计，HIDL 语言与 C 语言类似（但前者不使用 C 预处理器）。下面未描述的所有标点符号（用途明显的 `=` 和 `|` 除外）都是语法的一部分。

> **注意**：有关 HIDL 代码样式的详细信息，请参阅[代码样式指南](https://source.android.com/devices/architecture/hidl/code-style.html)。

- `/** */` 表示文档注释。此样式只能应用于类型、方法、字段和枚举值声明。
- `/* */` 表示多行注释。
- `//` 表示注释一直持续到行结束。除了 `//`，换行符与任何其他空白一样。
- 在以下示例语法中，从 `//` 到行结束的文本不是语法的一部分，而是对语法的注释。
- `[empty]` 表示该字词可能为空。
- `?` 跟在文本或字词后，表示它是可选的。
- `...` 表示包含零个或多个项、用指定的分隔符号分隔的序列。HIDL 中不含可变参数。
- 逗号用于分隔序列元素。
- 分号用于终止各个元素，包括最后的元素。
- 大写字母是非终止符。
- `*italics*` 是一个令牌系列，如 `*integer*` 或 `*identifier*`（标准 C 解析规则）。
- `*constexpr* `是 C 样式的常量表达式（例如 `1 + 1` 和 `1L << 3`）。
- `*import_name*` 是软件包或接口名称，按 [HIDL 版本编号](https://source.android.com/devices/architecture/hidl/versioning.html)中所述的方式加以限定。
- 小写 `words` 是文本令牌。

例如：

```HIDL
ROOT =
    PACKAGE IMPORTS PREAMBLE { ITEM ITEM ... }  // not for types.hal
  | PACKAGE IMPORTS ITEM ITEM...  // only for types.hal; no method definitions

ITEM =
    ANNOTATIONS? oneway? identifier(FIELD, FIELD ...) GENERATES?;
  |  safe_union identifier { UFIELD; UFIELD; ...};
  |  struct identifier { SFIELD; SFIELD; ...};  // Note - no forward declarations
  |  union identifier { UFIELD; UFIELD; ...};
  |  enum identifier: TYPE { ENUM_ENTRY, ENUM_ENTRY ... }; // TYPE = enum or scalar
  |  typedef TYPE identifier;

VERSION = integer.integer;

PACKAGE = package android.hardware.identifier[.identifier[...]]@VERSION;

PREAMBLE = interface identifier EXTENDS

EXTENDS = <empty> | extends import_name  // must be interface, not package

GENERATES = generates (FIELD, FIELD ...)

// allows the Binder interface to be used as a type
// (similar to typedef'ing the final identifier)
IMPORTS =
   [empty]
  |  IMPORTS import import_name;

TYPE =
  uint8_t | int8_t | uint16_t | int16_t | uint32_t | int32_t | uint64_t | int64_t |
 float | double | bool | string
|  identifier  // must be defined as a typedef, struct, union, enum or import
               // including those defined later in the file
|  memory
|  pointer
|  vec<TYPE>
|  bitfield<TYPE>  // TYPE is user-defined enum
|  fmq_sync<TYPE>
|  fmq_unsync<TYPE>
|  TYPE[SIZE]

FIELD =
   TYPE identifier

UFIELD =
   TYPE identifier
  |  safe_union identifier { FIELD; FIELD; ...} identifier;
  |  struct identifier { FIELD; FIELD; ...} identifier;
  |  union identifier { FIELD; FIELD; ...} identifier;

SFIELD =
   TYPE identifier
  |  safe_union identifier { FIELD; FIELD; ...};
  |  struct identifier { FIELD; FIELD; ...};
  |  union identifier { FIELD; FIELD; ...};
  |  safe_union identifier { FIELD; FIELD; ...} identifier;
  |  struct identifier { FIELD; FIELD; ...} identifier;
  |  union identifier { FIELD; FIELD; ...} identifier;

SIZE =  // Must be greater than zero
     constexpr

ANNOTATIONS =
     [empty]
  |  ANNOTATIONS ANNOTATION

ANNOTATION =
  |  @identifier
  |  @identifier(VALUE)
  |  @identifier(ANNO_ENTRY, ANNO_ENTRY  ...)

ANNO_ENTRY =
     identifier=VALUE

VALUE =
     "any text including \" and other escapes"
  |  constexpr
  |  {VALUE, VALUE ...}  // only in annotations

ENUM_ENTRY =
     identifier
  |  identifier = constexpr
```

### 5.1.3 术语表

| Binder 化 | 表示 HIDL 用于进程之间的远程过程调用，并通过类似 Binder 的机制来实现。另请参阅“直通式”。 |
| :-------- | ------------------------------------------------------------ |
| 异步回调  | 由 HAL 用户提供、传递给 HAL（通过 HIDL 方法）并由 HAL 调用以随时返回数据的接口。 |
| 同步回调  | 将数据从服务器的 HIDL 方法实现返回到客户端。不用于返回无效值或单个原始值的方法。 |
| 客户端    | 调用特定接口的方法的进程。HAL 进程或框架进程可以是一个接口的客户端和另一个接口的服务器。另请参阅“直通式”。 |
| 扩展      | 表示向另一接口添加方法和/或类型的接口。一个接口只能扩展另一个接口。可用于具有相同软件包名称的 Minor 版本递增，也可用于在旧软件包的基础上构建的新软件包（例如，供应商扩展）。 |
| 生成      | 表示将值返回给客户端的接口方法。要返回一个非原始值或多个值，则会生成同步回调函数。 |
| 接口      | 方法和类型的集合。会转换为 C++ 或 Java 中的类。接口中的所有方法均按同一方向调用：客户端进程会调用由服务器进程实现的方法。 |
| 单向      | 应用到 HIDL 方法时，表示该方法既不返回任何值也不会造成阻塞。 |
| 软件包    | 共用一个版本的接口和数据类型的集合。                         |
| 直通式    | HIDL 的一种模式，使用这种模式时，服务器是共享库，由客户端进行 `dlopen` 处理。在直通模式下，客户端和服务器是相同的进程，但代码库不同。此模式仅用于将旧版代码库并入 HIDL 模型。另请参阅“Binder 化”。 |
| 服务器    | 实现接口的方法的进程。另请参阅“直通式”。                     |
| 传输      | 在服务器和客户端之间移动数据的 HIDL 基础架构。               |
| 版本      | 软件包的版本。由两个整数组成：Major 版本和 Minor 版本。Minor 版本递增可以添加（但不会更改）类型和方法。 |



## 5.2 接口和软件包

HIDL 是围绕接口进行编译的，接口是面向对象的语言使用的一种用来定义行为的抽象类型。每个接口都是软件包的一部分。



### 5.2.1 软件包

软件包名称可以具有子级，例如 `package.subpackage`。已发布的 HIDL 软件包的根目录是 `hardware/interfaces` 或 `vendor/vendorName`（例如 Pixel 设备为 `vendor/google`）。软件包名称在根目录下形成一个或多个子目录；定义软件包的所有文件都位于同一目录下。例如，`package android.hardware.example.extension.light@2.0` 可以在 `hardware/interfaces/example/extension/light/2.0` 下找到。

下表列出了软件包前缀和位置：

| 软件包前缀             | 位置                               | 接口类型         |
| :--------------------- | :--------------------------------- | :--------------- |
| `android.hardware.*`   | `hardware/interfaces/*`            | HAL              |
| `android.frameworks.*` | `frameworks/hardware/interfaces/*` | frameworks/ 相关 |
| `android.system.*`     | `system/hardware/interfaces/*`     | system/ 相关     |
| `android.hidl.*`       | `system/libhidl/transport/*`       | core             |

软件包目录中包含扩展名为 `.hal` 的文件。每个文件均必须包含一个指定文件所属的软件包和版本的 `package` 语句。文件 `types.hal`（如果存在）并不定义接口，而是定义软件包中每个接口可以访问的数据类型。

### 5.2.2 接口定义

除了 `types.hal` 之外，其他 `.hal` 文件均定义一个接口。接口通常定义如下：

```c++
interface IBar extends IFoo { // IFoo is another interface
    // embedded types
    struct MyStruct {/*...*/};

    // interface methods
    create(int32_t id) generates (MyStruct s);
    close();
};
```

不含显式 `extends` 声明的接口会从 `android.hidl.base@1.0::IBase`（类似于 Java 中的 `java.lang.Object`）隐式扩展。隐式导入的 IBase 接口声明了多种不应也不能在用户定义的接口中重新声明或以其他方式使用的预留方法。这些方法包括：

- `ping`
- `interfaceChain`
- `interfaceDescriptor`
- `notifySyspropsChanged`
- `linkToDeath`
- `unlinkToDeath`
- `setHALInstrumentation`
- `getDebugInfo`
- `debug`
- `getHashChain`

### 5.2.3 导入

`import` 语句是用于访问其他软件包中的软件包接口和类型的 HIDL 机制。`import` 语句本身涉及两个实体：

- 导入实体：可以是软件包或接口；以及
- 被导入实体：也可以是软件包或接口。

导入实体由 `import` 语句的位置决定。当该语句位于软件包的 `types.hal` 中时，导入的内容对整个软件包是可见的；这是软件包级导入。当该语句位于接口文件中时，导入实体是接口本身；这是接口级导入。

被导入实体由 `import` 关键字后面的值决定。该值不必是完全限定名称；如果某个组成部分被删除了，系统会自动使用当前软件包中的信息填充该组成部分。 对于完全限定值，支持的导入情形有以下几种：

- **完整软件包导入**。如果该值是一个软件包名称和版本（语法见下文），则系统会将整个软件包导入至导入实体中。

- 部分导入

  如果值为：

  - 一个接口，则系统会将该软件包的 `types.hal` 和该接口导入至导入实体中。
  - 在 `types.hal` 中定义的 UDT，则系统仅会将该 UDT 导入至导入实体中（不导入 `types.hal` 中的其他类型）。

- **仅类型导入**。如果该值将上文所述的“部分导入”的语法与关键字 `types` 而不是接口名称配合使用，则系统仅会导入指定软件包的 `types.hal` 中的 UDT。

导入实体可以访问以下各项的组合：

- `types.hal` 中定义的被导入软件包的常见 UDT；
- 被导入的软件包的接口（完整软件包导入）或指定接口（部分导入），以便调用它们、向其传递句柄和/或从其继承句柄。

导入语句使用完全限定类型名称语法来提供被导入的软件包或接口的名称和版本：

```
import android.hardware.nfc@1.0;            // import a whole package
import android.hardware.example@1.0::IQuux; // import an interface and types.hal
import android.hardware.example@1.0::types; // import just types.hal
```

### 5.2.4 接口继承

接口可以是之前定义的接口的扩展。扩展可以是以下三种类型中的一种：

- 接口可以向其他接口添加功能，并按原样纳入其 API。
- 软件包可以向其他软件包添加功能，并按原样纳入其 API。
- 接口可以从软件包或特定接口导入类型。

接口只能扩展一个其他接口（不支持多重继承）。具有非零 Minor 版本号的软件包中的每个接口必须扩展一个以前版本的软件包中的接口。例如，如果 4.0 版本的软件包 `derivative` 中的接口 `IBar` 是基于（扩展了）1.2 版本的软件包 `original` 中的接口 `IFoo`，并且您又创建了 1.3 版本的软件包 `original`，则 4.1 版本的 `IBar` 不能扩展 1.3 版本的 `IFoo`。相反，4.1 版本的 `IBar` 必须扩展 4.0 版本的 `IBar`，因为后者是与 1.2 版本的 `IFoo` 绑定的。 如果需要，5.0 版本的 `IBar` 可以扩展 1.3 版本的 `IFoo`。

接口扩展并不意味着生成的代码中存在代码库依赖关系或跨 HAL 包含关系，接口扩展只是在 HIDL 级别导入数据结构和方法定义。HAL 中的每个方法必须在相应 HAL 中实现。

### 5.2.5 接口布局总结

总结了如何管理 HIDL 接口软件包（如 `hardware/interfaces`）并整合了整个 HIDL 部分提供的信息。在阅读之前，请务必先熟悉 [HIDL 版本控制](https://source.android.com/devices/architecture/hidl/versioning)、[使用 hidl-gen 添加哈希](https://source.android.com/devices/architecture/hidl/hashing#hidl-gen)中的哈希概念、关于[在一般情况下使用 HIDL](https://source.android.com/devices/architecture/hidl/) 的详细信息以及以下定义：

| 术语                  | 定义                                                         |
| :-------------------- | :----------------------------------------------------------- |
| 应用二进制接口 (ABI)  | 应用编程接口 + 所需的任何二进制链接。                        |
| 完全限定名称 (fqName) | 用于区分 hidl 类型的名称。例如：`android.hardware.foo@1.0::IFoo`。 |
| 软件包                | 包含 HIDL 接口和类型的软件包。例如：`android.hardware.foo@1.0`。 |
| 软件包根目录          | 包含 HIDL 接口的根目录软件包。例如：HIDL 接口 `android.hardware` 在软件包根目录 `android.hardware.foo@1.0` 中。 |
| 软件包根目录路径      | 软件包根目录映射到的 Android 源代码树中的位置。              |

有关更多定义，请参阅 HIDL [术语](https://source.android.com/devices/architecture/hidl/#terms)。

#### 每个文件都可以通过软件包根目录映射及其完全限定名称找到

软件包根目录以参数 `-r android.hardware:hardware/interfaces` 的形式指定给 `hidl-gen`。例如，如果软件包为 `vendor.awesome.foo@1.0::IFoo` 并且向 `hidl-gen` 发送了 `-r vendor.awesome:some/device/independent/path/interfaces`，那么接口文件应该位于 `$ANDROID_BUILD_TOP/some/device/independent/path/interfaces/foo/1.0/IFoo.hal`。

在实践中，建议称为 `awesome` 的供应商或原始设备制造商 (OEM) 将其标准接口放在 `vendor.awesome` 中。在选择了软件包路径之后，不能再更改该路径，因为它已写入接口的 ABI。

#### 软件包路径映射不得重复

例如，如果您有 `-rsome.package:$PATH_A` 和 `-rsome.package:$PATH_B`，则 `$PATH_A` 必须等于 `$PATH_B` 才能实现一致的接口目录（这也能让[接口版本控制起来](https://source.android.com/devices/architecture/hidl/versioning)更简单）。

#### 软件包根目录必须有版本控制文件

如果创建一个软件包路径（如 `-r vendor.awesome:vendor/awesome/interfaces`），则还应创建文件 `$ANDROID_BUILD_TOP/vendor/awesome/interfaces/current.txt`，该文件应包含使用 `hidl-gen`（在[使用 hidl-gen 添加哈希](https://source.android.com/devices/architecture/hidl/hashing#hidl-gen)中广泛进行了讨论）中的 `-Lhash` 选项所创建接口的哈希。



## 5.3 接口哈希

本文档介绍了 HIDL 接口哈希，该哈希是一种旨在防止意外更改接口并确保接口更改经过全面审查的机制。这种机制是必需的，因为 HIDL 接口带有版本编号，也就是说，接口一经发布便不得再更改，但不会影响应用二进制接口 (ABI) 的情况（例如更正备注）除外。

### 5.3.1 布局

每个软件包根目录（即映射到 `hardware/interfaces` 的 `android.hardware` 或映射到 `vendor/foo/hardware/interfaces` 的 `vendor.foo`）都必须包含一个列出所有已发布 HIDL 接口文件的 `current.txt` 文件。

```
# current.txt files support comments starting with a ‘#' character
# this file, for instance, would be vendor/foo/hardware/interfaces/current.txt

# Each line has a SHA-256 hash followed by the name of an interface.
# They have been shortened in this doc for brevity but they are
# 64 characters in length in an actual current.txt file.
d4ed2f0e...995f9ec4 vendor.awesome.foo@1.0::IFoo # comments can also go here

# types.hal files are also noted in types.hal files
c84da9f5...f8ea2648 vendor.awesome.foo@1.0::types

# Multiple hashes can be in the file for the same interface. This can be used
# to note how ABI sustaining changes were made to the interface.
# For instance, here is another hash for IFoo:

# Fixes type where "FooCallback" was misspelled in comment on "FooStruct"
822998d7...74d63b8c vendor.awesome.foo@1.0::IFoo
```

> **注意**：为了便于跟踪各个哈希的来源，Google 会将 HIDL `current.txt` 文件分为不同的部分：第一部分列出在 Android O 中发布的接口文件，第二部分列出在 Android O MR1 中发布的接口文件。我们强烈建议在您的 `current.txt` 文件中使用类似布局。

### 5.3.2 使用 hidl-gen 添加哈希

您可以手动将哈希添加到 `current.txt` 文件中，也可以使用 `hidl-gen` 添加。以下代码段提供了可与 `hidl-gen` 搭配使用来管理 `current.txt` 文件的命令示例（哈希已缩短）：

```
hidl-gen -L hash -r vendor.awesome:vendor/awesome/hardware/interfaces -r android.hardware:hardware/interfaces -r android.hidl:system/libhidl/transport vendor.awesome.nfc@1.0::types
9626fd18...f9d298a6 vendor.awesome.nfc@1.0::types
hidl-gen -L hash -r vendor.awesome:vendor/awesome/hardware/interfaces -r android.hardware:hardware/interfaces -r android.hidl:system/libhidl/transport vendor.awesome.nfc@1.0::INfc
07ac2dc9...11e3cf57 vendor.awesome.nfc@1.0::INfc
hidl-gen -L hash -r vendor.awesome:vendor/awesome/hardware/interfaces -r android.hardware:hardware/interfaces -r android.hidl:system/libhidl/transport vendor.awesome.nfc@1.0
9626fd18...f9d298a6 vendor.awesome.nfc@1.0::types
07ac2dc9...11e3cf57 vendor.awesome.nfc@1.0::INfc
f2fe5442...72655de6 vendor.awesome.nfc@1.0::INfcClientCallback
hidl-gen -L hash -r vendor.awesome:vendor/awesome/hardware/interfaces -r android.hardware:hardware/interfaces -r android.hidl:system/libhidl/transport vendor.awesome.nfc@1.0 >> vendor/awesome/hardware/interfaces/current.txt
```

`hidl-gen` 生成的每个接口定义库都包含哈希，通过调用 `IBase::getHashChain` 可检索这些哈希。`hidl-gen` 编译接口时，会检查 HAL 软件包根目录中的 `current.txt` 文件，以查看 HAL 是否已被更改：

- 如果没有找到 HAL 的哈希，则接口会被视为未发布（处于开发阶段），并且编译会继续进行。
- 如果找到了相应哈希，则会对照当前接口对其进行检查：
  - 如果接口与哈希匹配，则编译会继续进行。
  - 如果接口与哈希不匹配，则编译会暂停，因为这意味着之前发布的接口会被更改。
    - 要在更改的同时不影响 ABI（请参阅 [ABI 稳定性](https://source.android.com/devices/architecture/hidl/hashing#abi-stability)），请务必先修改 `current.txt` 文件，然后编译才能继续进行。
    - 所有其他更改都应在接口的 minor 或 major 版本升级中进行。

### 5.3.3 ABI 稳定性

**要点**：请仔细阅读并理解本部分。

应用二进制接口 (ABI) 包括二进制关联/调用规范/等等。如果 ABI/API 发生更改，则相应接口就不再适用于使用官方接口编译的常规 `system.img`。

确保接口带有版本编号且 ABI 稳定**至关重要**，具体原因有如下几个：

- 可确保您的实现能够通过供应商测试套件 (VTS) 测试，通过该测试后您将能够正常进行仅限框架的 OTA。
- 作为原始设备制造商 (OEM)，您将能够提供简单易用且符合规定的板级支持包 (BSP)。
- 有助于您跟踪哪些接口可以发布。您可以将 `current.txt` 视为接口目录的“地图”，从中了解软件包根目录中提供的所有接口的历史记录和状态。

对于在 `current.txt` 中已有条目的接口，为其添加新的哈希时，请务必仅为可保持 ABI 稳定性的接口添加哈希。请查看以下更改类型：

| 允许的更改   | 更改备注（除非这会更改方法的含义）。更改参数的名称。更改返回参数的名称。更改注释。 |
| :----------- | ------------------------------------------------------------ |
| 不允许的更改 | 重新排列参数、方法等…重命名接口或将其移至新的软件包。重命名软件包。在接口的任意位置添加方法/结构体字段等等…会破坏 C++ vtable 的任何更改。等等… |

## 5.4 服务和数据转移

如何注册和发现服务，以及如何通过调用 `.hal` 文件内的接口中定义的方法将数据发送到服务。

### 5.4.1 注册服务

HIDL 接口服务器（实现接口的对象）可注册为已命名的服务。注册的名称不需要与接口或软件包名称相关。如果没有指定名称，则使用名称“默认”；这应该用于不需要注册同一接口的两个实现的 HAL。例如，在每个接口中定义的服务注册的 C++ 调用是：

```
status_t status = myFoo->registerAsService();
status_t anotherStatus = anotherFoo->registerAsService("another_foo_service");  // if needed
```

HIDL 接口的版本包含在接口本身中。版本自动与服务注册关联，并可通过每个 HIDL 接口上的方法调用 (`android::hardware::IInterface::getInterfaceVersion()`) 进行检索。服务器对象不需要注册，并可通过 HIDL 方法参数传递到其他进程，相应的接收进程会向服务器发送 HIDL 方法调用。

### 5.4.2 发现服务

客户端代码按名称和版本请求指定的接口，并对所需的 HAL 类调用 `getService`：

```
// C++
sp<V1_1::IFooService> service = V1_1::IFooService::getService();
sp<V1_1::IFooService> alternateService = V1_1::IFooService::getService("another_foo_service");
// Java
V1_1.IFooService service = V1_1.IFooService.getService(true /* retry */);
V1_1.IFooService alternateService = V1_1.IFooService.getService("another", true /* retry */);
```

每个版本的 HIDL 接口都会被视为单独的接口。因此，`IFooService` 版本 1.1 和 `IFooService` 版本 2.2 都可以注册为“foo_service”，并且两个接口上的 `getService("foo_service")` 都可获取该接口的已注册服务。因此，在大多数情况下，注册或发现服务均无需提供名称参数（也就是说名称为“默认”）。

供应商接口对象还会影响所返回接口的传输方法。对于软件包 `android.hardware.foo@1.0` 中的接口 `IFoo`，`IFoo::getService` 返回的接口始终使用设备清单中针对 `android.hardware.foo` 声明的传输方法（如果相应条目存在的话）；如果该传输方法不存在，则返回 nullptr。

在某些情况下，即使没有获得相关服务，也可能需要立即继续。例如，当客户端希望自行管理服务通知或者在需要获取所有 hwservice 并检索它们的诊断程序（例如 `atrace`）中时，可能会发生这种情况。在这种情况下，可以使用其他 API，例如 C++ 中的 `tryGetService` 或 Java 中的 `getService("instance-name", false)`。Java 中提供的旧版 API `getService` 也必须与服务通知一起使用。使用此 API 不会避免以下竞态条件：当客户端使用某个非重试 API 请求服务器后，该服务器对自身进行了注册。

### 5.4.3 服务终止通知

想要在服务终止时收到通知的客户端会接收到框架传送的终止通知。要接收通知，客户端必须：

1. 将 HIDL 类/接口 `hidl_death_recipient`（位于 C++ 代码中，而非 HIDL 中）归入子类。
2. 替换其 `serviceDied()` 方法。
3. 实例化 `hidl_death_recipient` 子类的对象。
4. 在要监控的服务上调用 `linkToDeath()` 方法，并传入 `IDeathRecipient` 的接口对象。请注意，此方法并不具备在其上调用它的终止接收方或代理的所有权。

伪代码示例（C++ 和 Java 类似）：

```
class IMyDeathReceiver : hidl_death_recipient {
  virtual void serviceDied(uint64_t cookie,
                           wp<IBase>& service) override {
    log("RIP service %d!", cookie);  // Cookie should be 42
  }
};
....
IMyDeathReceiver deathReceiver = new IMyDeathReceiver();
m_importantService->linkToDeath(deathReceiver, 42);
```

同一终止接收方可能已在多个不同的服务上注册。

### 5.4.4 数据转移

可通过调用 `.hal` 文件内的接口中定义的方法将数据发送到服务。具体方法有两类：

- **阻塞**方法会等到服务器产生结果。
- **单向**方法仅朝一个方向发送数据且不阻塞。如果 RPC 调用中正在传输的数据量超过实现限制，则调用可能会阻塞或返回错误指示（具体行为尚不确定）。

不返回值但未声明为 `oneway` 的方法仍会阻塞。

在 HIDL 接口中声明的所有方法都是单向调用，要么从 HAL 发出，要么到 HAL。该接口没有指定具体调用方向。需要从 HAL 发起调用的架构应该在 HAL 软件包中提供两个（或更多个）接口并从每个进程提供相应的接口。我们根据接口的调用方向来取名“客户端”或“服务器”（即 HAL 可以是一个接口的服务器，也可以是另一个接口的客户端）。

#### 回调

回调”一词可以指代两个不同的概念，可通过“同步回调”和“异步回调”进行区分。

“同步回调”用于返回数据的一些 HIDL 方法。返回多个值（或返回非基元类型的一个值）的 HIDL 方法会通过回调函数返回其结果。如果只返回一个值且该值是基元类型，则不使用回调且该值从方法中返回。服务器实现 HIDL 方法，而客户端实现回调。

“异步回调”允许 HIDL 接口的服务器发起调用。通过第一个接口传递第二个接口的实例即可完成此操作。第一个接口的客户端必须作为第二个接口的服务器。第一个接口的服务器可以在第二个接口对象上调用方法。例如，HAL 实现可以通过在由该进程创建和提供的接口对象上调用方法来将信息异步发送回正在使用它的进程。用于异步回调的接口中的方法可以是阻塞方法（并且可能将值返回到调用程序），也可以是 `oneway` 方法。要查看相关示例，请参阅 [HIDL C++](https://source.android.com/devices/architecture/hidl-cpp/interfaces) 中的“异步回调”。

要简化内存所有权，方法调用和回调只能接受 `in` 参数，并且不支持 `out` 或 `inout` 参数。

#### 每事务限制

每事务限制不会强制限制在 HIDL 方法和回调中发送的数据量。但是，每事务调用 4KB 以上的数据便被视为过度调用。如果发生这种情况，建议重新设计给定 HIDL 接口的架构。另一个限制是可供 HIDL 基础架构处理多个同时进行的事务的资源。由于多个线程或进程向一个进程发送调用或者接收进程未能快速处理多个 `oneway` 调用，因此多个事务可能会同时进行。默认情况下，所有并发事务可用的最大总空间为 1MB。

在设计良好的接口中，不应出现超出这些资源限制的情况；如果超出的话，则超出资源的调用可能会阻塞，直到资源可用或发出传输错误的信号。每当因正在进行的总事务导致出现超出每事务限制或溢出 HIDL 实现资源的情况时，系统都会记录下来以方便调试。

#### 方法实现

HIDL 生成以目标语言（C++ 或 Java）声明必要类型、方法和回调的标头文件。客户端和服务器代码的 HIDL 定义方法和回调的原型是相同的。HIDL 系统提供调用程序端（整理 IPC 传输的数据）的方法**代理**实现，并将代码**存根**到被调用程序端（将数据传递到方法的开发者实现）。

函数的调用程序（HIDL 方法或回调）拥有对传递到该函数的数据结构的所有权，并在调用后保留所有权；被调用程序在所有情况下都无需释放存储。

- 在 C++ 中，数据可能是只读的（尝试写入可能会导致细分错误），并且在调用期间有效。客户端可以深层复制数据，以在调用期间外传播。
- 在 Java 中，代码会接收数据的本地副本（普通 Java 对象），代码可以保留和修改此数据或允许垃圾回收器回收。

#### 非 RPC 数据转移

HIDL 在不使用 RPC 调用的情况下通过两种方法来转移数据：共享内存和快速消息队列 (FMQ)，只有 C++ 同时支持这两种方法。

- **共享内存**。内置 HIDL 类型 `memory` 用于传递表示已分配的共享内存的对象。 可以在接收进程中使用，以映射共享内存。

- 快速消息队列 (FMQ)

  HIDL 提供了一种可实现无等待消息传递的模板化消息队列类型。它在直通式或绑定式模式下不使用内核或调度程序（设备间通信将不具有这些属性）。通常，HAL 会设置其队列的末尾，从而创建可以借助内置 HIDL 类型`MQDescriptorSync`或`MQDescriptorUnsync`的参数通过 RPC 传递的对象。接收进程可使用此对象设置队列的另一端。

  - “已同步”队列不能溢出，且只能有一个读取器。
  - “未同步”队列可以溢出，且可以有多个读取器；每个读取器必须及时读取数据，否则数据就会丢失。

  两种队列都不能下溢（从空队列进行读取将会失败），且都只能有一个写入器。

有关 FMQ 的更多详情，请参阅快速消息队列 .

## 5.5 快速消息转移

HIDL 的远程过程调用 (RPC) 基础架构使用 Binder 机制，这意味着调用涉及开销、需要内核操作，并且可以触发调度程序操作。不过，对于必须在开销较小且无内核参与的进程之间传输数据的情况，则使用快速消息队列 (FMQ) 系统。

FMQ 会创建具有所需属性的消息队列。`MQDescriptorSync` 或 `MQDescriptorUnsync` 对象可通过 HIDL RPC 调用发送，并可供接收进程用于访问消息队列。

### 5.5.1 MessageQueue 类型

#### 未同步

未同步队列只有一个写入器，但可以有任意多个读取器。此类队列有一个写入位置；不过，每个读取器都会跟踪各自的独立读取位置。

对此类队列执行写入操作一定会成功（不会检查是否出现溢出情况），但前提是写入的内容不超出配置的队列容量（如果写入的内容超出队列容量，则操作会立即失败）。由于各个读取器的读取位置可能不同，因此每当新的写入操作需要空间时，系统都允许数据离开队列，而无需等待每个读取器读取每条数据。

读取操作负责在数据离开队列末尾之前对其进行检索。如果读取操作尝试读取的数据超出可用数据量，则该操作要么立即失败（如果非阻塞），要么等到有足够多的可用数据时（如果阻塞）。如果读取操作尝试读取的数据超出队列容量，则读取一定会立即失败。

如果某个读取器的读取速度无法跟上写入器的写入速度，则写入的数据量和该读取器尚未读取的数据量加在一起会超出队列容量，这会导致下一次读取不会返回数据；相反，该读取操作会将读取器的读取位置重置为等于最新的写入位置，然后返回失败。如果在发生溢出后但在下一次读取之前，系统查看可供读取的数据，则会显示可供读取的数据超出了队列容量，这表示发生了溢出。（如果队列溢出发生在系统查看可用数据和尝试读取这些数据之间，则溢出的唯一表征就是读取操作失败。）

#### 已同步

已同步队列有一个写入器和一个读取器，其中写入器有一个写入位置，读取器有一个读取位置。写入的数据量不可能超出队列可提供的空间；读取的数据量不可能超出队列当前存在的数据量。

如果尝试写入的数据量超出可用空间或尝试读取的数据量超出现有数据量，则会立即返回失败，或会阻塞到可以完成所需操作为止，具体取决于调用的是阻塞还是非阻塞写入或读取函数。如果尝试读取或尝试写入的数据量超出队列容量，则读取或写入操作一定会立即失败。

### 5.5.2 设置 FMQ

一个消息队列需要多个 `MessageQueue` 对象：一个对象用作数据写入目标位置，以及一个或多个对象用作数据读取来源。没有关于哪些对象用于写入数据或读取数据的显式配置；用户需负责确保没有对象既用于读取数据又用于写入数据，也就是说最多只有一个写入器，并且对于已同步队列，最多只有一个读取器。

#### 创建第一个 MessageQueue 对象

通过单个调用创建并配置消息队列：

```
#include <fmq/MessageQueue.h>
using android::hardware::kSynchronizedReadWrite;
using android::hardware::kUnsynchronizedWrite;
using android::hardware::MQDescriptorSync;
using android::hardware::MQDescriptorUnsync;
using android::hardware::MessageQueue;
....
// For a synchronized non-blocking FMQ
mFmqSynchronized =
  new (std::nothrow) MessageQueue<uint16_t, kSynchronizedReadWrite>
      (kNumElementsInQueue);
// For an unsynchronized FMQ that supports blocking
mFmqUnsynchronizedBlocking =
  new (std::nothrow) MessageQueue<uint16_t, kUnsynchronizedWrite>
      (kNumElementsInQueue, true /* enable blocking operations */);
```

- `MessageQueue(numElements)` 初始化程序负责创建并初始化支持消息队列功能的对象。
- `MessageQueue(numElements, configureEventFlagWord)` 初始化程序负责创建并初始化支持消息队列功能和阻塞的对象。
- `flavor` 可以是 `kSynchronizedReadWrite`（对于已同步队列）或 `kUnsynchronizedWrite`（对于未同步队列）。
- `uint16_t`（在本示例中）可以是任意不涉及嵌套式缓冲区（无 `string` 或 `vec` 类型）、句柄或接口的 HIDL 定义的类型。
- `kNumElementsInQueue` 表示队列的大小（以条目数表示）；它用于确定将为队列分配的共享内存缓冲区的大小。

#### 创建第二个 MessageQueue 对象

使用从消息队列的第一侧获取的 `MQDescriptor` 对象创建消息队列的第二侧。通过 HIDL RPC 调用将 `MQDescriptor` 对象发送到将容纳消息队列末端的进程。`MQDescriptor` 包含该队列的相关信息，其中包括：

- 用于映射缓冲区和写入指针的信息。
- 用于映射读取指针的信息（如果队列已同步）。
- 用于映射事件标记字词的信息（如果队列是阻塞队列）。
- 对象类型 (``)，其中包含 [HIDL 定义的队列元素类型](https://source.android.com/devices/architecture/hidl-cpp/types)和队列风格（已同步或未同步）。

`MQDescriptor` 对象可用于构建 `MessageQueue` 对象：

```
MessageQueue<T, flavor>::MessageQueue(const MQDescriptor<T, flavor>& Desc, bool resetPointers)
```

`resetPointers` 参数表示是否在创建此 `MessageQueue` 对象时将读取和写入位置重置为 0。在未同步队列中，读取位置（在未同步队列中，是每个 `MessageQueue` 对象的本地位置）在此对象创建过程中始终设为 0。通常，`MQDescriptor` 是在创建第一个消息队列对象过程中初始化的。要对共享内存进行额外的控制，您可以手动设置 `MQDescriptor`（`MQDescriptor` 是在 [`system/libhidl/base/include/hidl/MQDescriptor.h`](https://android.googlesource.com/platform/system/libhidl/+/master/base/include/hidl/MQDescriptor.h) 中定义的），然后按照本部分所述内容创建每个 `MessageQueue` 对象。

#### 阻塞队列和事件标记

默认情况下，队列不支持阻塞读取/写入。有两种类型的阻塞读取/写入调用：

* 短格式：有三个参数（数据指针、项数、超时）。支持阻塞针对单个队列的各个读取/写入操作。在使用这种格式时，队列将在内部处理事件标记和位掩码，并且第一个消息队列对象必须初始化为第二个参数为 `true`。例如：

  ```
  // For an unsynchronized FMQ that supports blocking
  mFmqUnsynchronizedBlocking =
    new (std::nothrow) MessageQueue<uint16_t, kUnsynchronizedWrite>
        (kNumElementsInQueue, true /* enable blocking operations */);
  ```

* 长格式：有六个参数（包括事件标记和位掩码）。支持在多个队列之间使用共享 `EventFlag` 对象，并允许指定要使用的通知位掩码。在这种情况下，必须为每个读取和写入调用提供事件标记和位掩码。

对于长格式，可在每个 `readBlocking()` 和 `writeBlocking()` 调用中显式提供 `EventFlag`。可以将其中一个队列初始化为包含一个内部事件标记，如果是这样，则必须使用 `getEventFlagWord()` 从相应队列的 `MessageQueue` 对象中提取该标记，以用于在每个进程中创建与其他 FMQ 一起使用的 `EventFlag` 对象。或者，可以将 `EventFlag` 对象初始化为具有任何合适的共享内存。

一般来说，每个队列都应只使用以下三项之一：非阻塞、短格式阻塞，或长格式阻塞。混合使用也不算是错误；但要获得理想结果，则需要谨慎地进行编程。

### 5.5.3 使用 MessageQueue

`MessageQueue` 对象的公共 API 是：

```
size_t availableToWrite()  // Space available (number of elements).
size_t availableToRead()  // Number of elements available.
size_t getQuantumSize()  // Size of type T in bytes.
size_t getQuantumCount() // Number of items of type T that fit in the FMQ.
bool isValid() // Whether the FMQ is configured correctly.
const MQDescriptor<T, flavor>* getDesc()  // Return info to send to other process.

bool write(const T* data)  // Write one T to FMQ; true if successful.
bool write(const T* data, size_t count) // Write count T's; no partial writes.

bool read(T* data);  // read one T from FMQ; true if successful.
bool read(T* data, size_t count);  // Read count T's; no partial reads.

bool writeBlocking(const T* data, size_t count, int64_t timeOutNanos = 0);
bool readBlocking(T* data, size_t count, int64_t timeOutNanos = 0);

// Allows multiple queues to share a single event flag word
std::atomic<uint32_t>* getEventFlagWord();

bool writeBlocking(const T* data, size_t count, uint32_t readNotification,
uint32_t writeNotification, int64_t timeOutNanos = 0,
android::hardware::EventFlag* evFlag = nullptr); // Blocking write operation for count Ts.

bool readBlocking(T* data, size_t count, uint32_t readNotification,
uint32_t writeNotification, int64_t timeOutNanos = 0,
android::hardware::EventFlag* evFlag = nullptr) // Blocking read operation for count Ts;

//APIs to allow zero copy read/write operations
bool beginWrite(size_t nMessages, MemTransaction* memTx) const;
bool commitWrite(size_t nMessages);
bool beginRead(size_t nMessages, MemTransaction* memTx) const;
bool commitRead(size_t nMessages);
```

`availableToWrite()` 和 `availableToRead()` 可用于确定在一次操作中可传输的数据量。在未同步队列中：

- `availableToWrite()` 始终返回队列容量。
- 每个读取器都有自己的读取位置，并会针对 `availableToRead()` 进行自己的计算。
- 如果是读取速度缓慢的读取器，队列可以溢出，这可能会导致 `availableToRead()` 返回的值大于队列的大小。发生溢出后进行的第一次读取操作将会失败，并且会导致相应读取器的读取位置被设为等于当前写入指针，无论是否通过 `availableToRead()` 报告了溢出都是如此。

如果所有请求的数据都可以（并已）传输到队列/从队列传出，则 `read()` 和 `write()` 方法会返回 `true`。这些方法不会阻塞；它们要么成功（并返回 `true`），要么立即返回失败 (`false`)。

`readBlocking()` 和 `writeBlocking()` 方法会等到可以完成请求的操作，或等到超时（`timeOutNanos` 值为 0 表示永不超时）。

阻塞操作使用事件标记字词来实现。默认情况下，每个队列都会创建并使用自己的标记字词来支持短格式的 `readBlocking()` 和 `writeBlocking()`。多个队列可以共用一个字词，这样一来，进程就可以等待对任何队列执行写入或读取操作。可以通过调用 `getEventFlagWord()` 获得指向队列事件标记字词的指针，此类指针（或任何指向合适的共享内存位置的指针）可用于创建 `EventFlag` 对象，以传递到其他队列的长格式 `readBlocking()` 和 `writeBlocking()`。`readNotification` 和 `writeNotification` 参数用于指示事件标记中的哪些位应该用于针对相应队列发出读取和写入信号。`readNotification` 和 `writeNotification` 是 32 位的位掩码。

`readBlocking()` 会等待 `writeNotification` 位；如果该参数为 0，则调用一定会失败。如果 `readNotification` 值为 0，则调用不会失败，但成功的读取操作将不会设置任何通知位。在已同步队列中，这意味着相应的 `writeBlocking()` 调用一定不会唤醒，除非已在其他位置对相应的位进行设置。在未同步队列中，`writeBlocking()` 将不会等待（它应仍用于设置写入通知位），而且对于读取操作来说，不适合设置任何通知位。同样，如果 `readNotification` 为 0，`writeblocking()` 将会失败，并且成功的写入操作会设置指定的 `writeNotification` 位。

要一次等待多个队列，请使用 `EventFlag` 对象的 `wait()` 方法来等待通知的位掩码。`wait()` 方法会返回一个状态字词以及导致系统设置唤醒的位。然后，该信息可用于验证相应的队列是否有足够的控件或数据来完成所需的写入/读取操作，并执行非阻塞 `write()`/`read()`。要获取操作后通知，请再次调用 `EventFlag` 的 `wake()` 方法。有关 `EventFlag` 抽象的定义，请参阅 `system/libfmq/include/fmq/EventFlag.h`。

### 5.5.4 零复制操作

`read`/`write`/`readBlocking`/`writeBlocking()` API 会将指向输入/输出缓冲区的指针作为参数，并在内部使用 `memcpy()` 调用，以便在相应缓冲区和 FMQ 环形缓冲区之间复制数据。为了提高性能，Android 8.0 及更高版本包含一组 API，这些 API 可提供对环形缓冲区的直接指针访问，这样便无需使用 `memcpy` 调用。

使用以下公共 API 执行零复制 FMQ 操作：

```
bool beginWrite(size_t nMessages, MemTransaction* memTx) const;
bool commitWrite(size_t nMessages);

bool beginRead(size_t nMessages, MemTransaction* memTx) const;
bool commitRead(size_t nMessages);
```

- `beginWrite` 方法负责提供用于访问 FMQ 环形缓冲区的基址指针。在数据写入之后，使用 `commitWrite()` 提交数据。`beginRead`/`commitRead` 方法的运作方式与之相同。
- `beginRead`/`Write` 方法会将要读取/写入的消息条数视为输入，并会返回一个布尔值来指示是否可以执行读取/写入操作。如果可以执行读取或写入操作，则 `memTx` 结构体中会填入基址指针，这些指针可用于对环形缓冲区共享内存进行直接指针访问。
- `MemRegion` 结构体包含有关内存块的详细信息，其中包括基础指针（内存块的基址）和以 `T` 表示的长度（以 HIDL 定义的消息队列类型表示的内存块长度）。
- `MemTransaction` 结构体包含两个 `MemRegion` 结构体（`first` 和 `second`），因为对环形缓冲区执行读取或写入操作时可能需要绕回到队列开头。这意味着，要对 FMQ 环形缓冲区执行数据读取/写入操作，需要两个基址指针。

从 `MemRegion` 结构体获取基址和长度：

```
T* getAddress(); // gets the base address
size_t getLength(); // gets the length of the memory region in terms of T
size_t getLengthInBytes(); // gets the length of the memory region in bytes
```

获取对 `MemTransaction` 对象内的第一个和第二个 `MemRegion` 的引用：

```
const MemRegion& getFirstRegion(); // get a reference to the first MemRegion
const MemRegion& getSecondRegion(); // get a reference to the second MemRegion
```

使用零复制 API 写入 FMQ 的示例：

```
MessageQueueSync::MemTransaction tx;
if (mQueue->beginRead(dataLen, &tx)) {
    auto first = tx.getFirstRegion();
    auto second = tx.getSecondRegion();

    foo(first.getAddress(), first.getLength()); // method that performs the data write
    foo(second.getAddress(), second.getLength()); // method that performs the data write

    if(commitWrite(dataLen) == false) {
       // report error
    }
} else {
   // report error
}
```

以下辅助方法也是 `MemTransaction` 的一部分：

- `T* getSlot(size_t idx);`
  返回一个指针，该指针指向属于此 `MemTransaction` 对象一部分的 `MemRegions` 内的槽位 `idx`。如果 `MemTransaction` 对象表示要读取/写入 N 个类型为 T 的项目的内存区域，则 `idx` 的有效范围在 0 到 N-1 之间。
- `bool copyTo(const T* data, size_t startIdx, size_t nMessages = 1);`
  将 `nMessages` 个类型为 T 的项目写入到该对象描述的内存区域，从索引 `startIdx` 开始。此方法使用 `memcpy()`，但并非旨在用于零复制操作。如果 `MemTransaction` 对象表示要读取/写入 N 个类型为 T 的项目的内存区域，则 `idx` 的有效范围在 0 到 N-1 之间。
- `bool copyFrom(T* data, size_t startIdx, size_t nMessages = 1);`
  一种辅助方法，用于从该对象描述的内存区域读取 `nMessages` 个类型为 T 的项目，从索引 `startIdx` 开始。此方法使用 `memcpy()`，但并非旨在用于零复制操作。

### 5.5.5 通过 HIDL 发送队列

在创建侧执行的操作：

1. 创建消息队列对象，如上所述。
2. 使用 `isValid()` 验证对象是否有效。
3. 如果您要通过将 `EventFlag` 传递到长格式的 `readBlocking()`/`writeBlocking()` 来等待多个队列，则可以从经过初始化的 `MessageQueue` 对象提取事件标记指针（使用 `getEventFlagWord()`）以创建标记，然后使用该标记创建必需的 `EventFlag` 对象。
4. 使用 `MessageQueue` `getDesc()` 方法获取描述符对象。
5. 在 `.hal` 文件中，为某个方法提供一个类型为 `fmq_sync` 或 `fmq_unsync` 的参数，其中 `T` 是 HIDL 定义的一种合适类型。使用此方法将 `getDesc()` 返回的对象发送到接收进程。

在接收侧执行的操作：

1. 使用描述符对象创建 `MessageQueue` 对象。务必使用相同的队列风格和数据类型，否则将无法编译模板。
2. 如果您已提取事件标记，则在接收进程中从相应的 `MessageQueue` 对象提取该标记。
3. 使用 `MessageQueue` 对象传输数据。

## 5.6 使用Binder IPC

Android 8 中对 Binder 驱动程序进行的更改、提供了有关使用 Binder IPC 的详细信息，并列出了必需的 SELinux 政策。

### 5.6.1 对 Binder 驱动程序进行的更改

从 Android 8 开始，Android 框架和 HAL 现在使用 Binder 互相通信。由于这种通信方式极大地增加了 Binder 流量，因此 Android 8 包含了几项改进，旨在使 Binder IPC 速度很快。SoC 供应商和 OEM 应直接从 android-4.4、android-4.9 和更高版本[内核/通用](https://android.googlesource.com/kernel/common/)项目的相关分支进行合并。

#### 多个 Binder 域（上下文）

*通用 4.4 及更高版本，包括上游*

为了在框架（独立于设备）和供应商（特定于设备）代码之间彻底拆分 Binder 流量，Android 8 引入了“Binder 上下文”的概念。每个 Binder 上下文都有自己的设备节点和上下文（服务）管理器。您只能通过上下文管理器所属的设备节点对其进行访问，并且在通过特定上下文传递 Binder 节点时，只能由另一个进程从相同的上下文访问上下文管理器，从而确保这些域完全互相隔离。如需使用方法的详细信息，请参阅 [vndbinder](https://source.android.com/devices/architecture/hidl/binder-ipc#vndbinder) 和 [vndservicemanager](https://source.android.com/devices/architecture/hidl/binder-ipc#vndservicemanager)。

#### 分散-集中

*通用 4.4 及更高版本，包括上游*

在之前的 Android 版本中，Binder 调用中的每条数据都会被复制 3 次：

- 一次是在调用进程中将数据序列化为 `Parcel`
- 一次是在内核驱动程序中将 `Parcel` 复制到目标进程
- 一次是在目标进程中反序列化 `Parcel`

Android 8 使用分散-集中优化将副本数量从 3 减少到 1。数据保留其原始结构和内存布局，且 Binder 驱动程序会立即将数据复制到目标进程中，而不是先在 `Parcel` 中序列化数据。在目标进程中，这些数据的结构和内存布局保持不变，并且，在无需再次复制的情况下即可读取这些数据。

#### 精细锁定

*通用 4.4 及更高版本，包括上游*

在之前的 Android 版本中，Binder 驱动程序使用全局锁来防范对重要数据结构的并发访问。虽然采用全局锁时出现争用的可能性极低，但主要的问题是，如果低优先级线程获得该锁，然后实现了抢占，则会导致同样需要获得该锁的优先级较高的线程出现严重的延迟。这会导致平台卡顿。

原先尝试解决此问题的方法是在保留全局锁的同时禁止抢占。但是，这更像是一种临时应对手段而非真正的解决方案，最终被上游拒绝并舍弃。后来尝试的解决方法侧重于提升锁定的精细程度，自 2017 年 1 月以来，Pixel 设备上一直采用的是更加精细的锁定。虽然这些更改大部分已公开，但后续版本中还会有一些重大的改进。

在确定了精细锁定实现中的一些小问题后，我们使用不同的锁定架构设计了一种改进的解决方案，并在所有通用内核分支中提交了更改。我们会继续在大量不同的设备上测试这种实现；由于目前我们没有发现这个方案存在什么突出问题，因此建议搭载 Android 8 的设备都使用这种实现。

#### 实时优先级继承

Binder 驱动程序一直支持 nice 优先级继承。随着 Android 中以实时优先级运行的进程日益增加，现在出现以下这种情形也属正常：如果实时线程发出 Binder 调用，则处理该调用的进程中的线程同样会以实时优先级运行。为了支持这些使用情景，Android 8 现在在 Binder 驱动程序中实现了实时优先级继承。

除了事务级优先级继承之外，“节点优先级继承”允许节点（Binder 服务对象）指定执行对该节点的调用所需的最低优先级。之前版本的 Android 已经通过 nice 值支持节点优先级继承，但 Android 8 增加了对实时调度政策节点继承的支持。

>  **注意**：Android 性能团队发现，实时优先级继承会对框架 Binder 域 (`/dev/binder`) 造成负面影响，因此已针对该域**停用**实时优先级继承。

#### userspace 更改

Android 8 包含在通用内核中使用当前 Binder 驱动程序所需的所有 userspace 更改，但有一个例外：针对 `/dev/binder` 停用实时优先级继承的原始实现使用了 [ioctl](https://android.googlesource.com/kernel/msm/+/868f6ee048c6ff51dbd92353dd5c68bea4419c78)。由于后续开发将优先级继承的控制方法改为了更加精细的方法（根据 Binder 模式，而非上下文），因此，ioctl 不存于 Android 通用分支中，而是提交到了我们的通用内核中。

此项更改的影响是，所有节点均默认停用实时优先级继承。Android 性能团队发现，为 `hwbinder` 域中的所有节点启用实时优先级继承会有一定好处。

### 5.6.2 使用 Binder IPC

一直以来，供应商进程都使用 Binder 进程间通信 (IPC) 技术进行通信。在 Android 8 中，`/dev/binder` 设备节点成为框架进程的专有节点，这意味着供应商进程无法再访问此节点。供应商进程可以访问 `/dev/hwbinder`，但必须将其 AIDL 接口转为使用 HIDL。对于想要继续在供应商进程之间使用 AIDL 接口的供应商，Android 会按以下方式支持 Binder IPC。

#### vndbinder

Android 8 支持供供应商服务使用的新 Binder 域，访问此域需要使用 `/dev/vndbinder`（而非 `/dev/binder`）。添加 `/dev/vndbinder` 后，Android 现在拥有以下 3 个 IPC 域：

| IPC 域           | 说明                                                         |
| :--------------- | :----------------------------------------------------------- |
| `/dev/binder`    | 框架/应用进程之间的 IPC，使用 AIDL 接口                      |
| `/dev/hwbinder`  | 框架/供应商进程之间的 IPC，使用 HIDL 接口 供应商进程之间的 IPC，使用 HIDL 接口 |
| `/dev/vndbinder` | 供应商/供应商进程之间的 IPC，使用 AIDL 接口                  |

为了显示 `/dev/vndbinder`，请确保内核配置项 `CONFIG_ANDROID_BINDER_DEVICES` 设为 `"binder,hwbinder,vndbinder"`（这是 Android 通用内核树的默认设置）。

通常，供应商进程不直接打开 Binder 驱动程序，而是链接到打开 Binder 驱动程序的 `libbinder` 用户空间库。为 `::android::ProcessState()` 添加方法可为 `libbinder` 选择 Binder 驱动程序。供应商进程应该在调用 `ProcessState,`、`IPCThreadState` 或发出任何普通 Binder 调用**之前**调用此方法。要使用该方法，请在供应商进程（客户端和服务器）的 `main()` 后放置以下调用：

```
ProcessState::initWithDriver("/dev/vndbinder");
```

#### vndservicemanager

以前，Binder 服务通过 `servicemanager` 注册，其他进程可从中检索这些服务。在 Android 8 中，`servicemanager` 现在专供框架使用，而应用进程和供应商进程无法再对其进行访问。

不过，供应商服务现在可以使用 `vndservicemanager`，这是一个使用 `/dev/vndbinder`（作为编译基础的源代码与框架 `servicemanager` 的相同）而非 `/dev/binder` 的 `servicemanager` 的新实例。供应商进程无需更改即可与 `vndservicemanager` 通信；当供应商进程打开 /`dev/vndbinder` 时，服务查询会自动转至 `vndservicemanager`。

`vndservicemanager` 二进制文件包含在 Android 的默认设备 Makefile 中。



### 5.6.3 SELinux 政策

想要使用 Binder 功能来相互通信的供应商进程需要满足以下要求：

1. 能够访问 `/dev/vndbinder`。
2. 将 Binder `{transfer, call}` 接入 `vndservicemanager`。
3. 针对想要通过供应商 Binder 接口调用供应商域 B 的任何供应商域 A 执行 `binder_call(A, B)` 操作。
4. 有权在 `vndservicemanager` 中对服务执行 `{add, find}` 操作。

要满足要求 1 和 2，请使用 `vndbinder_use()` 宏：

```
vndbinder_use(some_vendor_process_domain);
```

要满足要求 3，需要通过 Binder 通信的供应商进程 A 和 B 的 `binder_call(A, B)` 可以保持不变，且不需要重命名。

要满足要求 4，必须按照处理服务名称、服务标签和规则的方式进行更改。

有关 SELinux 的详细信息，请参阅 [Android 中的安全增强型 Linux](https://source.android.com/security/selinux)。有关 Android 8.0 中 SELinux 的详细信息，请参阅 [SELinux for Android 8.0](https://source.android.com/security/selinux/images/SELinux_Treble.pdf)。

#### 服务名称

以前，供应商进程在 `service_contexts` 文件中注册服务名称并添加用于访问该文件的相应规则。来自 `device/google/marlin/sepolicy` 的 `service_contexts` 文件示例：

```
AtCmdFwd                              u:object_r:atfwd_service:s0
cneservice                            u:object_r:cne_service:s0
qti.ims.connectionmanagerservice      u:object_r:imscm_service:s0
rcs                                   u:object_r:radio_service:s0
uce                                   u:object_r:uce_service:s0
vendor.qcom.PeripheralManager         u:object_r:per_mgr_service:s0
```

在 Android 8 中，`vndservicemanager` 会改为加载 `vndservice_contexts` 文件。迁移到 `vndservicemanager`（且已经在旧的 `service_contexts` 文件中）的供应商服务应该添加到新的 `vndservice_contexts` 文件中。

#### 服务标签

以前，服务标签（例如 `u:object_r:atfwd_service:s0`）在 `service.te` 文件中定义。例如：

```
type atfwd_service,      service_manager_type;
```

在 Android 8 中，您必须将类型更改为 `vndservice_manager_type`，并将规则移至 `vndservice.te` 文件。例如：

```
type atfwd_service,    vndservice_manager_type;
```

#### Servicemanager 规则

以前，规则会授予域访问权限，以向 `servicemanager` 添加服务或在其中查找服务。例如：

```
allow atfwd atfwd_service:service_manager find;
allow some_vendor_app atfwd_service:service_manager add;
```

在 Android 8 中，这样的规则可继续存在并使用相同的类。示例：

```
allow atfwd atfwd_service:service_manager find;allow some_vendor_app atfwd_service:service_manager add;
```

## 5.7 使用MemoryBlock

HIDL MemoryBlock 是构建在 `hidl_memory`、`HIDL @1.0::IAllocator` 和 `HIDL @1.0::IMapper` 之上的抽象层，专为有多个内存块共用单个内存堆的 HIDL 服务而设计。

### 5.7.1 性能提升



### 5.7.2 架构



### 5.7.3 常规用法



### 5.7.4 扩展用法





## 5.8 网络堆栈配置工具

### 5.8.1 Netutils 封装容器

### 5.8.2 Netutils 封装容器过滤器





## 5.9 线程模型

### 5.9.1 直通模式下的线程

### 5.9.2 绑定式 HAL 中的线程

### 5.9.3 服务器线程模型





## 5.10 转换模块

### 5.10.1 使用 c2hal

### 5.10.2 c2hal 操作

### 5.10.3 手动操作

### 5.10.4 实现 HAL



## 5.11 数据类型

### 5.11.1 数据表示法

### 5.11.2 注释

### 5.11.2 前向声明

### 5.11.3 嵌套式声明

### 5.11.4 原始指针语法

### 5.11.5 接口

### 5.11.6 MQDescriptorSync 和 MQDescriptorUnsync

### 5.11.7 memory 类型

### 5.11.8 pointer 类型

### 5.11.9 bitfield <T> 类型模板

### 5.11.10 句柄基本类型

### 5.11.11 有大小的数组

### 5.11.12 字符串

### 5.11.13 vec<T> 类型模板

### 5.11.14 用户定义的类型

## 5.12 Sfae Union

### 5.12.1 语法

### 5.12.2 用法

### 5.12.3 Monostate





## 5.13 版本编号

### 5.13.1 HIDL 代码结构

### 5.13.2 版本编号理念

### 5.13.3 构建接口

### 5.13.4 HIDL 代码布局

### 5.13.5 用户定义的类型的完全限定名称

### 5.13.6 完全限定的枚举值

### 5.13.7 自动推理规则

### 5.13.8 types.hal

### 5.13.9 接口级版本编号

### 5.13.10 软件包级版本编号





## 5.14 代码样式指南

### 5.14.1 命名规范

### 5.14.2 备注

### 5.14.3 格式





