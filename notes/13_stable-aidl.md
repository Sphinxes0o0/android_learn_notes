# 稳定的AIDL

Android 10增加了对稳定的Android接口定义语言（AIDL）的支持，这是一种跟踪AIDL接口提供的应用程序接口（API）/应用程序二进制接口（ABI）的新方法。稳定的AIDL与AIDL有以下主要区别：

- 接口在构建系统中使用定义`aidl_interfaces`。
- 接口只能包含结构化数据。代表所需类型的包裹可根据其AIDL定义自动创建，并自动进行编组和非编组。
- 接口可以声明为稳定的（向后兼容）。发生这种情况时，将在AIDL接口旁边的文件中跟踪其API并对其版本进行版本控制。

## 定义AIDL接口

的定义`aidl_interface`如下所示：

```protobuf
aidl_interface {
    name: "my-module-name",
    local_include_dir: "tests_1",
    srcs: [
        "tests_1/some/package/IFoo.aidl",
        "tests_1/some/package/Thing.aidl",
    ],
    api_dir: "api/test-piece-1",
    versions: ["1"],
}
```

- `name`：模块的名称。在这种情况下，将创建两个使用相应语言的存根库：`my-module-name-java`和 `my-module-name-cpp`。要防止创建C ++库，请使用 `gen_cpp`。这还会创建其他构建系统操作，可用于检查和更新API。
- `local_include_dir`：包开始的路径。
- `srcs`：编译成目标语言的稳定AIDL源文件的列表。
- `api_dir`：转储接口以前版本的API定义的路径。AIDL使用这些转储来确保捕获到接口的API破坏更改（请参见[Versioning interfaces](https://source.android.com/devices/architecture/aidl/stable-aidl#versioning-interfaces)）。Android R弃用该`api_dir`属性。
- `versions`：冻结在的早期版本的界面`api_dir`，从Android R开始，`versions`冻结在。如果没有接口的冻结版本，则不应指定该版本，也不会进行兼容性检查。`aidl_api/name`

## 编写AIDL文件

稳定的AIDL中的接口类似于传统的接口，不同之处在于不允许它们使用非结构化的可组合包裹（因为它们不稳定！）。稳定AIDL的主要区别在于包裹的定义方式。以前，包裹是预先*申报的*；在稳定的AIDL中，可包裹字段和变量是明确定义的。

```java
// in a file like 'some/package/Thing.aidl'
package some.package;

parcelable SubThing {
    String a = "foo";
    int b;
}
```

目前支持的默认（但不是必须）`boolean`，`char`， `float`，`double`，`byte`，`int`，`long`，和`String`。

## 使用存根库

将存根库添加为模块的依赖项之后，可以将它们包含在文件中。以下是构建系统中的存根库示例（`Android.mk`也可用于旧版模块定义）：

```protobuf
cc_... {
    name: ...,
    shared_libs: ["my-module-name-cpp"],
    ...
}
# or
java_... {
    name: ...,
    static_libs: ["my-module-name-java"],
    ...
}
```

C ++中的示例：

```c++
#include "some/package/IFoo.h"
#include "some/package/Thing.h"
...
    // use just like traditional AIDL
```

Java范例：

```java
import some.package.IFoo;import some.package.Thing;...  // use just like traditional AIDL
```

## 版本控制界面

声明名称为**foo**的模块还会在构建系统中创建一个目标，您可以使用该目标来管理模块的API。构建后，**foo-freeze-api** 会在`api_dir`或 下添加一个新的API定义，具体取决于Android版本，以用于接口的下一版本。构建此 属性还将更新属性以反映其他版本。一旦指定的属性，构建系统运行冷冻版本之间，也和在ToT最新的冻结版本之间的兼容性检查。`aidl_api/name``versions``versions`

为了保持接口的稳定性，所有者可以添加新的：

- 接口末尾的方法（或具有明确定义的新序列的方法）
- 地块末尾的元素（要求为每个元素添加默认值）
- 常数值
- 在Android R中，枚举器

不允许执行其他任何操作，其他任何人都不能修改界面（否则，它们可能会与所有者进行的更改发生冲突）。

## 模块命名规则

在R中，生成的模块名称由其模块名称，版本和目标组成。格式为name-version-target。

- 名称：模块名称
- 版
  - `Vversion-number`
  - `unstable`：ToT版本
- 目标：`java`，`cpp`，`ndk`，或`ndk_platform`

对于最新的稳定版本，请省略版本字段（Java目标除外）。换句话说，不会生成。`module-Vlatest-stable-version-(cpp|ndk|ndk_platform)`

假设有一个名为**foo**的模块，其最新版本为**2**，并且支持NDK和C ++。在这种情况下，AIDL生成以下模块：

- 基于版本1
  - `foo-V1-(java|cpp|ndk|ndk_platform)`
- 基于版本2（最新的稳定版本）
  - `foo-(java|cpp|ndk|ndk_platform)`
  - `foo-V2-java` （内容与foo-java相同）
- 基于ToT版本
  - `foo-unstable-(java|cpp|ndk|ndk_platform)`

在大多数情况下，输出文件名与模块名相同。但是，对于C ++或NDK模块的ToT版本，输出文件名与模块名不同。

例如，输出文件名`foo-unstable-cpp`是`foo-V3-cpp.so`，不`foo-unstable-cpp.so`低于如所示的。

- 基于版本1： `foo-V1-(cpp|ndk|ndk_platform).so`
- 基于版本2： `foo-V2-(cpp|ndk|ndk_platform).so`
- 基于ToT版本： `foo-V3-(cpp|ndk|ndk_platform).so`

## 新的元界面方法

Android 10为稳定的AIDL添加了几种元接口方法。

### 查询远程对象的接口版本

客户端可以查询远程对象正在实现的接口版本，并将返回的版本与客户端使用的接口版本进行比较。

C ++中的示例：

```c++
sp<IFoo> foo = ... // the remote object
int32_t my_ver = IFoo::VERSION;
int32_t remote_ver = foo->getInterfaceVersion();
if (remote_ver < my_ver) {
  // the remote side is using an older interface
}
```

Java范例：

```c++
IFoo foo = ... // the remote object
int my_ver = IFoo.VERSION;
int remote_ver = foo.getInterfaceVersion();
if (remote_ver < my_ver) {
  // the remote side is using an older interface
}
```

对于Java语言，远程端必须实现`getInterfaceVersion()`如下：

```java
class MyFoo extends IFoo.Stub {
    @Override
    public final int getInterfaceVersion() { return IFoo.VERSION; }
}
```

这是因为生成的类（`IFoo`，`IFoo.Stub`等）在客户端和服务器之间共享（例如，这些类可以在引导类路径中）。共享类时，即使服务器可能是使用较旧版本的接口构建的，服务器也将与最新版本的类链接。如果此meta接口在共享类中实现，则始终返回最新版本。然而，通过实施该方法如上述，该接口的版本号被嵌入在服务器的代码（因为`IFoo.VERSION`是`static final int`参考当内联），因此该方法可以返回确切版本的服务器，用构建的。

### 处理较旧的界面

客户端可能会使用AIDL接口的较新版本进行更新，但服务器使用的是旧的AIDL接口。在这种情况下，客户端不得调用旧接口中不存在的新方法。在稳定AIDL之前，将静默地忽略调用这种不存在的方法，并且客户端不知道是否调用了该方法。

借助稳定的AIDL，客户可以控制更多。在客户端，您可以将默认实现设置为AIDL接口。仅当未在远程端实现该方法时，才调用默认实现中的方法（因为该方法是使用较旧版本的接口构建的）。

C ++中的示例：

```c++
class MyDefault : public IFooDefault {
  Status anAddedMethod(...) {
   // do something default
  }
};

// once per an interface in a process
IFoo::setDefaultImpl(std::unique_ptr<IFoo>(MyDefault));

foo->anAddedMethod(...); // MyDefault::anAddedMethod() will be called if the
                         // remote side is not implementing it
```

Java范例：

```java
IFoo.Stub.setDefaultImpl(new IFoo.Default() {
    @Override
    public xxx anAddedMethod(...)  throws RemoteException {
        // do something default
    }
}); // once per an interface in a process


foo.anAddedMethod(...);
```

**注意：**在Java中，`setDefaultImpl`是在`Stub`类中，而不是在 `interface`类中。

不需要在AIDL接口中提供所有方法的默认实现。可以在远程端实现的方法（因为您可以确定在AIDL接口说明中介绍了方法时才构建了远程方法），因此不需要在默认`impl` 类中覆盖该方法。

## 将现有的AIDL转换为结构化/稳定的AIDL

如果您有现有的AIDL接口和使用它的代码，请使用以下步骤将接口转换为稳定的AIDL接口。

1. 识别接口的所有依赖性。对于接口依赖的每个包，请确定是否在稳定的AIDL中定义了该包。如果未定义，则必须转换包。
2. 将界面中的所有可拆分项转换为稳定的可拆分项（界面文件本身可以保持不变）。为此，可以直接在AIDL文件中表示其结构。必须重写管理类以使用这些新类型。这可以在创建`aidl_interface`包之前完成 （如下）。
3. 创建一个`aidl_interface`包（如上所述），其中包含模块的名称，其依赖项以及所需的任何其他信息。为了使其稳定（不仅仅是结构化），还需要对其进行版本控制。有关更多信息，请参见[Versioning interfaces](https://source.android.com/devices/architecture/aidl/stable-aidl#versioning-interfaces)。