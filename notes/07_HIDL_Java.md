# HIDL Java

Android 8.0 对 Android 操作系统的架构重新进行了设计，以在独立于设备的 Android 平台与特定于设备和供应商的代码之间定义清晰的接口。
Android 已经以 HAL 接口的形式（在 hardware/libhardware 中定义为 C 头文件）定义了许多此类接口。
HIDL 将这些 HAL 接口替换为带版本编号的稳定接口，可以采用 Java（如下所述），也可以采用 C++ 的客户端和服务器端 HIDL 接口。

HIDL 接口主要通过原生代码使用，因此 HIDL 专注于自动生成高效的 C++ 代码。
不过，HIDL 接口也必须能够直接通过 Java 使用，因为有些 Android 子系统（如 Telephony）很可能具有 Java HIDL 接口。

本部分介绍了 HIDL 接口的 Java 前端，详细说明了如何创建、注册和使用服务，
以及使用 Java 编写的 HAL 和 HAL 客户端如何与 HIDL RPC 系统进行交互。

## 概览

### 作为客户端

这是软件包 `android.hardware.foo@1.0` 中的接口 IFoo 的一个客户端示例，
该软件包的服务名称注册为 default，另外还有一个自定义服务名称为 second_impl 的附加服务。

#### 添加库
需要添加相应 HIDL 存根库的依赖项（如果您需要使用的话）。通常，这是一个静态库：
```
// in Android.bp
static_libs: [ "android.hardware.foo-V1.0-java", ],
// in Android.mk
LOCAL_STATIC_JAVA_LIBRARIES += android.hardware.foo-V1.0-java
```

如果已经获得这些库的依赖项，则还可以使用共享关联：
```
// in Android.bp
libs: [ "android.hardware.foo-V1.0-java", ],
// in Android.mk
LOCAL_JAVA_LIBRARIES += android.hardware.foo-V1.0-java
```

#### 在 Android 10 中添加库的额外注意事项

如果系统/供应商应用以 Android 10 及更高版本为目标平台，那么必须以静态方式包含这些库。
对于旧版应用，旧式行为会保留下来。或者，也可以仅使用设备上安装的自定义 JAR 中的 HIDL 类，
以及使用通过系统应用的现有 uses-library 机制提供的稳定 Java API。为了节省设备上的空间，建议采用这种方法。
如需了解详情，请参阅实现 Java SDK 库。

从 Android 10 开始，还会提供这些库的“浅层”版本。这些版本包含相关类，但不包含任何从属类。
例如，`android.hardware.foo-V1.0-java-shallow` 包含 foo 软件包中的类，
但不包含 `android.hidl.base-V1.0-java`（其中包含所有 HIDL 接口的基类）中的类。
如果要创建的库已经将优先接口的基类作为依赖项，则可以使用以下命令：

```
// in Android.bp
static_libs: [ "android.hardware.foo-V1.0-java-shallow", ],
// in Android.mk
LOCAL_STATIC_JAVA_LIBRARIES += android.hardware.foo-V1.0-java-shallow
```

启动类路径中也不再提供 HIDL 基类和管理器库。不过，这些内容已移动到具有 jarjar 的新命名空间中。
使用 HIDL 的启动类路径中的模块需要使用这些库的浅层变体以及将 jarjar_rules: ":framework-jarjar-rules" 添加到其 Android.bp 中，
以避免重复的代码，并让系统/供应商应用使用隐藏 API。

修改 Java 源代码
此服务只有一个版本 (@1.0)，因此该代码仅检索该版本。如需了解如何处理此服务的多个不同版本，请参阅接口扩展。

```
import android.hardware.foo.V1_0.IFoo;
...
// retry to wait until the service starts up if it is in the manifest
IFoo server = IFoo.getService(true /* retry */); // throws NoSuchElementException if not available
IFoo anotherServer = IFoo.getService("second_impl", true /* retry */);
server.doSomething(…);
```

### 提供服务

Java 中的框架代码可能需要提供接口才能接收来自 HAL 的异步回调。

对于 1.0 版软件包 `android.hardware.foo` 中的接口 IFooCallback，可以按照以下步骤用 Java 实现接口：

1. 用 HIDL 定义您的接口。
2. 打开 `/tmp/android/hardware/foo/IFooCallback.java` 作为参考。
3. 为您的 Java 实现创建一个新模块。
4. 检查抽象类 `android.hardware.foo.V1_0.IFooCallback.Stub`，然后编写一个新类以对其进行扩展，并实现抽象方法。

#### 查看自动生成的文件
```
hidl-gen -o /tmp -Ljava \
  -randroid.hardware:hardware/interfaces \
  -randroid.hidl:system/libhidl/transport android.hardware.foo@1.0
```
这些命令会生成目录 `/tmp/android/hardware/foo/1.0`。对于文件 `hardware/interfaces/foo/1.0/IFooCallback.hal`，
这会生成文件 `/tmp/android/hardware/foo/1.0/IFooCallback.java`，其中包含 Java 接口、代理代码和存根（代理和存根均与接口吻合）。

-Lmakefile 会生成在构建时运行此命令的规则，并允许您包含 `android.hardware.foo-V1.0-java` 并链接到相应的文件。
可以在 `hardware/interfaces/update-makefiles.sh` 中找到自动为充满接口的项目执行此操作的脚本。 
本示例中的路径是相对路径；硬件/接口可能是代码树下的一个临时目录，让您能够先开发 HAL 然后再进行发布。

#### 运行服务
HAL 提供 IFoo 接口，它必须通过接口 `IFooCallback` 对框架进行异步回调。
`IFooCallback` 接口不按名称注册为可检测到的服务；相反，IFoo 必须包含一个诸如 `setFooCallback(IFooCallback x)` 的方法。

要在 1.0 版软件包 `android.hardware.foo` 中设置 `IFooCallback`，请将 `android.hardware.foo-V1.0-java `添加到 Android.mk。运行服务的代码为：
```java
import android.hardware.foo.V1_0.IFoo;
import android.hardware.foo.V1_0.IFooCallback.Stub;
....
class FooCallback extends IFooCallback.Stub {
    // implement methods
}
....
// Get the service you will be receiving callbacks from.
// This also starts the threadpool for your callback service.
IFoo server = IFoo.getService(true /* retry */); // throws NoSuchElementException if not available
....
// This must be a persistent instance variable, not local,
//   to avoid premature garbage collection.
FooCallback mFooCallback = new FooCallback();
....
// Do this once to create the callback service and tell the "foo-bar" service
server.setFooCallback(mFooCallback);
```

#### 接口扩展
假设指定服务在所有设备上实现了接口 IFoo，那么该服务在特定设备上可能会提供在接口扩展 IBetterFoo 中实现的附加功能，如下所示：
```
interface IFoo {
   ...
};

interface IBetterFoo extends IFoo {
   ...
};
```
感知到扩展接口的调用代码可以使用 castFrom() Java 方法将基本接口安全地转换为扩展接口：
```
IFoo baseService = IFoo.getService(true /* retry */); // throws NoSuchElementException if not available
IBetterFoo extendedService = IBetterFoo.castFrom(baseService);
if (extendedService != null) {
  // The service implements the extended interface.
} else {
  // The service implements only the base interface.
}
```

## 数据类型

给定一个 HIDL 接口文件，Java HIDL 后端会生成 Java 接口、存根和代理代码。它支持所有标量 HIDL 类型（[u]int{8,16,32,64}_t, float, double, 及 enum），
以及受支持 HIDL 类型的字符串、接口、safe_union 类型、结构类型、数组和矢量。Java HIDL 后端不支持联合类型、原生句柄、共享内存和 fmq 类型。

由于 Java 运行时本身不支持无符号整数的概念，所有无符号类型（以及基于这些类型的枚举）都会被自动视为其有符号的等效项，也就是说，uint32_t 在 Java 接口中会变为 int。
不会进行任何值转换；Java 端的实现人员必须将有符号值当做无符号值来使用。

### 枚举

枚举不会生成 Java 枚举类，而是转换为包含各个枚举用例的静态常量定义的内部类。如果枚举类派生自其他枚举类，则会沿用后者的存储类型。
基于无符号整数类型的枚举将重写为其有符号的等效项。由于基础类型是基元，因此即使没有零枚举器，枚举字段/变量的默认值也为零。

例如，一个类型为 uint8_t 的 SomeBaseEnum
```c/c++
enum SomeBaseEnum : uint8_t { foo = 3 };
enum SomeEnum : SomeBaseEnum {
    quux = 33,
    goober = 127
};
```
会变为：
```java
public final class SomeBaseEnum { public static final byte foo = 3; }
public final class SomeEnum {
    public static final byte foo = 3;
    public static final byte quux = 33;
    public static final byte goober = 127;
}

```
且：
```c/c++
enum SomeEnum : uint8_t {
    FIRST_CASE = 10,
    SECOND_CASE = 192
};
```
重写为：
```
public final class SomeEnum {
    static public final byte FIRST_CASE  = 10;  // no change
    static public final byte SECOND_CASE = -64;
}
```

### 字符串
Java 中的 String 为 utf-8 或 utf-16，但在传输时会转换为 utf-8 作为常见的 HIDL 类型。
另外，String 在传入 HIDL 时不能为 null。

### 数组和矢量
数组会转换为 Java 数组，矢量会转换为` ArrayList<T>`，其中 T 是相应的对象类型，
可能是 `vec<int32_t> => ArrayList<Integer>` 之类的封装标量类型。例如：
```c++
takeAnArray(int32_t[3] array);
returnAVector() generates (vec<int32_t> result);
```
…会变为：
```java
void takeAnArray(int[] array);
ArrayList<Integer> returnAVector();
```

### 结构
结构会转换为具有相似布局的 Java 类。例如：
```c/c++
struct Bar {
 vec<bool> someBools;
};
struct Foo {
 int32_t a;
 int8_t b;
 float[10] c;
 Bar d;
};
```
…会变为：
```java
class Bar {
 public final ArrayList<Boolean> someBools = new ArrayList();
};
class Foo {
 public int a;
 public byte b;
 public final float[] c = new float[10];
 public final Bar d = new Bar();
}
```
### 已声明的类型
在 `types.hal` 中声明的每个顶级类型都有自己的 .java 输出文件（根据 Java 要求）。
例如，以下 types.hal 文件会导致创建两个额外的文件（Foo.java 和 Bar.java）：
```c++
struct Foo {
 ...
};

struct Bar {
 ...

 struct Baz {
 };

 ...
};
```
Baz 的定义位于 Bar 的静态内部类中（在 Bar.java 内）。

## 接口错误和方法

### Void 方法
不返回结果的方法将转换为返回 void 的 Java 方法。例如，HIDL 声明：
```c++
doThisWith(float param);
```
…会变为：
```java
void doThisWith(float param);
```

### 单结果方法
返回单个结果的方法将转换为同样返回单个结果的 Java 等效方法。例如，以下方法：

doQuiteABit(int32_t a, int64_t b,
            float c, double d) generates (double something);

…会变为：

double doQuiteABit(int a, long b, float c, double d);

### 多结果方法
对于返回多个结果的每个方法，系统都会生成一个回调类，在其 onValues 方法中提供所有结果。 该回调会用作此方法的附加参数。例如，以下方法：
```
oneProducesTwoThings(SomeEnum x) generates (double a, double b);
```

会变为：

```
public interface oneProducesTwoThingsCallback {
    public void onValues(double a, double b);
}
void oneProducesTwoThings(byte x, oneProducesTwoThingsCallback cb);
```

`oneProducesTwoThings()` 的调用者通常使用匿名内部类或 lambda 在本地实现回调：

```
someInstanceOfFoo.oneProducesTwoThings(
         5 /* x */,
         new IFoo.oneProducesTwoThingsCallback() {
          @Override
          void onValues(double a, double b) {
             // do something interesting with a and b.
             ...
          }});
```
或：
```
someInstanceOfFoo.oneProducesTwoThings(5 /* x */,
    (a, b) -> a > 3.0 ? f(a, b) : g(a, b)));
```
还可以定义一个类以用作回调
```
class MyCallback implements oneProducesTwoThingsCallback {
  public void onValues(double a, double b) {
    // do something interesting with a and b.
  }
}
```
并传递 MyCallback 的一个实例作为 `oneProducesTwoThings()` 的第三个参数。

### 传输错误和终止通知接收方

由于服务实现可以在不同的进程中运行，在某些情况下，即使实现接口的进程已终止，客户端也可以保持活动状态。
调用已终止进程中托管的接口对象会失败并返回传输错误（调用的方法抛出的运行时异常）。
可以通过调用 `I<InterfaceName>.getService()` 以请求服务的新实例，从此类失败的调用中恢复。
不过，仅当崩溃的进程已重新启动且已向 servicemanager 重新注册其服务时，这种方法才有效（对 HAL 实现而言通常如此）。

接口的客户端也可以注册一个终止通知接收方，以便在服务终止时收到通知。如果调用在服务器刚终止时发出，仍然可能发生传输错误。
要在检索的 IFoo 接口上注册此类通知，客户端可以执行以下操作：

```
foo.linkToDeath(recipient, 1481 /* cookie */);
```

recipient 参数必须是由 HIDL 提供的 `HwBinder.DeathRecipient` 接口的实现。
该接口包含会在托管该接口的进程终止时调用的单个方法 serviceDied()。
```
final class DeathRecipient implements HwBinder.DeathRecipient {
    @Override
    public void serviceDied(long cookie) {
        // Deal with service going away
    }
}
```
cookie 参数包含使用 `linkToDeath()` 调用传递的 Cookie。您也可以在注册服务终止通知接收方后将其取消注册：
```
foo.unlinkToDeath(recipient);
```



## 导出常量

在接口不兼容 Java（例如由于使用联合类型而不兼容 Java）的情况下，可能仍需将常量（枚举值）导出到 Java 环境。这种情况需要用到 `hidl-gen -Ljava-constants …`，它会将已添加注释的枚举声明从软件包的接口文件提取出来，并生成一个名为 `[PACKAGE-NAME]-V[PACKAGE-VERSION]-java-constants` 的 java 库。请为每个要导出的枚举声明添加注释，如下所示：

```
@export
enum Foo : int32_t {
  SOME_VALUE,
  SOME_OTHER_VALUE,
};
```

如有必要，这一类型导出到 Java 环境时所使用的名称可以不同于接口声明中选定的名称，方法是添加注释参数 `name`：

```
@export(name="JavaFoo")
enum Foo : int32_t {
  SOME_VALUE,
  SOME_OTHER_VALUE,
};
```

如果依据 Java 惯例或个人偏好需要将一个公共前缀添加到枚举类型的值，请使用注释参数 `value_prefix`：

```
// File "types.hal".

package android.hardware.bar@1.0;

@export(name="JavaFoo", value_prefix="JAVA_")
enum Foo : int32_t {
  SOME_VALUE,
  SOME_OTHER_VALUE,
};
```

生成的 Java 类如下所示：

```
package android.hardware.bar.V1_0;

public class Constants {
  public final class JavaFoo {
    public static final int JAVA_SOME_VALUE = 0;
    public static final int JAVA_SOME_OTHER_VALUE = 1;
  };
};
```

最后，针对 `types.hal` 中声明的枚举类型的 Java 类型声明将划分到指定软件包中的类 `Constants` 内。声明为接口子级的枚举类型将划分到该接口的 Java 类声明下。