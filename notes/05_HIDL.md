# HIDL

HAL 接口定义语言（简称 HIDL，发音为“hide-l”）是用于指定 HAL 和其用户之间的接口的一种接口描述语言 (IDL)。HIDL 允许指定类型和方法调用（会汇集到接口和软件包中）。从更广泛的意义上来说，HIDL 是用于在可以独立编译的代码库之间进行通信的系统。

HIDL 旨在用于进程间通信 (IPC)。进程之间的通信经过 [*Binder*](https://source.android.com/devices/architecture/hidl/binder-ipc) 化。对于必须与进程相关联的代码库，还可以使用[直通模式](https://source.android.com/devices/architecture/hidl#passthrough)（在 Java 中不受支持）。

HIDL 可指定数据结构和方法签名，这些内容会整理归类到接口（与类相似）中，而接口会汇集到软件包中。尽管 HIDL 具有一系列不同的关键字，但 C++ 和 Java 程序员对 HIDL 的语法并不陌生。此外，HIDL 还使用 Java 样式的注释。

## 概览

HAL 接口定义语言（简称 HIDL，发音为“hide-l”）是用于指定 HAL 和其用户之间的接口的一种接口描述语言 (IDL)。HIDL 允许指定类型和方法调用（会汇集到接口和软件包中）。从更广泛的意义上来说，HIDL 是用于在可以独立编译的代码库之间进行通信的系统。

HIDL 旨在用于进程间通信 (IPC)。进程之间的通信经过 [*Binder*](https://source.android.com/devices/architecture/hidl/binder-ipc) 化。对于必须与进程相关联的代码库，还可以使用[直通模式](https://source.android.com/devices/architecture/hidl#passthrough)（在 Java 中不受支持）。

HIDL 可指定数据结构和方法签名，这些内容会整理归类到接口（与类相似）中，而接口会汇集到软件包中。尽管 HIDL 具有一系列不同的关键字，但 C++ 和 Java 程序员对 HIDL 的语法并不陌生。此外，HIDL 还使用 Java 样式的注释。

### HIDL 设计

HIDL 的目标是，框架可以在无需重新构建 HAL 的情况下进行替换。HAL 将由供应商或 SOC 制造商构建，放置在设备的 `/vendor` 分区中，这样一来，框架就可以在其自己的分区中通过 OTA 进行替换，而无需重新编译 HAL。

HIDL 设计在以下方面之间保持了平衡：

- **互操作性**。在可以使用各种架构、工具链和编译配置来编译的进程之间创建可互操作的可靠接口。HIDL 接口是分版本的，发布后不得再进行更改。
- **效率**。HIDL 会尝试尽可能减少复制操作的次数。HIDL 定义的数据以 C++ 标准布局数据结构传递至 C++ 代码，无需解压，可直接使用。此外，HIDL 还提供共享内存接口；由于 RPC 本身有点慢，因此 HIDL 支持两种无需使用 RPC 调用的数据传输方法：共享内存和快速消息队列 (FMQ)。
- **直观**。通过仅针对 RPC 使用 `in` 参数，HIDL 避开了内存所有权这一棘手问题（请参阅 [Android 接口定义语言 (AIDL)](https://developer.android.com/guide/components/aidl.html)）；无法从方法高效返回的值将通过回调函数返回。无论是将数据传递到 HIDL 中以进行传输，还是从 HIDL 接收数据，都不会改变数据的所有权，也就是说，数据所有权始终属于调用函数。数据仅需要在函数被调用期间保留，可在被调用的函数返回数据后立即清除。

### 使用直通模式

要将运行早期版本的 Android 的设备更新为使用 Android O，您可以将惯用的（和旧版）HAL 封装在一个新 HIDL 接口中，该接口将在绑定式模式和同进程（直通）模式提供 HAL。这种封装对于 HAL 和 Android 框架来说都是透明的。

直通模式仅适用于 C++ 客户端和实现。运行早期版本的 Android 的设备没有用 Java 编写的 HAL，因此 Java HAL 自然而然经过 Binder 化。

#### 直通式标头文件

编译 `.hal` 文件时，除了用于 Binder 通信的标头之外，`hidl-gen` 还会生成一个额外的直通标头文件 `BsFoo.h`；此标头定义了会被执行 `dlopen` 操作的函数。由于直通式 HAL 在它们被调用的同一进程中运行，因此在大多数情况下，直通方法由直接函数调用（同一线程）来调用。`oneway` 方法在各自的线程中运行，因为它们不需要等待 HAL 来处理它们（这意味着，在直通模式下使用 `oneway` 方法的所有 HAL 对于线程必须是安全的）。

如果有一个 `IFoo.hal`，`BsFoo.h` 会封装 HIDL 生成的方法，以提供额外的功能（例如使 `oneway` 事务在其他线程中运行）。该文件类似于 `BpFoo.h`，不过，所需函数是直接调用的，并未使用 Binder 传递调用 IPC。未来，HAL 的实现**可能提供**多种实现结果，例如 FooFast HAL 和 FooAccurate HAL。在这种情况下，系统会针对每个额外的实现结果创建一个文件（例如 `PTFooFast.cpp` 和 `PTFooAccurate.cpp`）。

#### Binder 化直通式 HAL

您可以将支持直通模式的 HAL 实现 Binder 化。如果有一个 HAL 接口 `a.b.c.d@M.N::IFoo`，系统会创建两个软件包：

- `a.b.c.d@M.N::IFoo-impl`。包含 HAL 的实现，并暴露函数 `IFoo* HIDL_FETCH_IFoo(const char* name)`。在旧版设备上，此软件包经过 `dlopen` 处理，且实现使用 `HIDL_FETCH_IFoo` 进行了实例化。您可以使用 `hidl-gen` 和 `-Lc++-impl` 以及 `-Landroidbp-impl` 来生成基础代码。
- `a.b.c.d@M.N::IFoo-service`。打开直通式 HAL，并将其自身注册为 Binder 化服务，从而使同一 HAL 实现能够同时以直通模式和 Binder 化模式使用。

如果有一个 `IFoo`，您可以调用 `sp IFoo::getService(string name, bool getStub)`，以获取对 `IFoo` 实例的访问权限。如果 `getStub` 为 True，则 `getService` 会尝试仅在直通模式下打开 HAL。如果 `getStub` 为 False，则 `getService` 会尝试找到 Binder 化服务；如果未找到，则它会尝试找到直通式服务。除了在 `defaultPassthroughServiceImplementation` 中，其余情况一律不得使用 `getStub` 参数。（搭载 Android O 的设备是完全 Binder 化的设备，因此不得在直通模式下打开服务。）

### HIDL 语法

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

### 术语表

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







## 接口和软件包





## 接口哈希





## 服务和数据转移





## 快速消息转移





## 使用Binder IPC





## 使用MemoryBlock



## 网络堆栈配置工具



## 线程模型



## 转换模块



## 数据类型



## Sfae Union



## 版本编号



## 代码样式指南



