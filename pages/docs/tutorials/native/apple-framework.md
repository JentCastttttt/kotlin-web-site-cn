---
type: tutorial
layout: tutorial
title:  "Kotlin/Native 开发 Apple Framework"
description: "编译 Kotlin/Native 代码并在 Objective-C 与 Swift 中使用它"
authors: Eugene Petrenko，乔禹昂（翻译）
date: 2019-04-15
showAuthorInfo: true
issue: EVAN-5132
---

Kotlin/Native 提供与 Objective-C/Swift 的双向互操作性。
Objective-C framework 与库可以在 Kotlin 代码中使用。
Kotlin 模块同样可以在 Swift/Objective-C 代码中使用。
除此之外，Kotlin/Native 也拥有
[C 互操作性](https://github.com/JetBrains/kotlin-native/blob/master/INTEROP.md)。
这篇 [Kotlin/Native 开发动态库](dynamic-libraries.html)<!--
-->教程包含了更多信息。

在本教程中，我们将看到如何在 Objective-C 与 Swift
编写的 macOS 与 iOS 应用程序中使用 Kotlin/Native 代码。
我们将使用 Kotlin 代码构建一个 framework。

在本教程中我们将：
- [创建一个 Kotlin 库](#创建一个-kotlin-库)并将它编译为 framework
- 检查生成的 [Objective-C 与 Swift API](#生成framework-头) 代码
- 在 [Objective-C](#在-objective-c-中使用代码) 与 [Swift](#在-swift-中使用代码) 中使用 framework
- 为 [macOS](#xcode-与-macos-目标平台) 与 [iOS](#xcode-与-ios-目标平台) [配置 Xcode](#xcode-and-framework-dependencies) 以使用 framework
   
## 创建一个 Kotlin 库

Kotlin/Native 编译器可以使 Kotlin 代码为 macOS 与 iOS
生产一个 framework 的输出。生成的 framework 包含在 Objective-C 与 Swift
中所有使用所需的声明与二进制文件。
理解这项技术的最佳方式是自己进行一下尝试。
我们首先来创建一个小型的 Kotlin 库，并在 Objective-C 程序中使用它。

我们创建该 `hello.kt` 文件，并在其中编写库的内容：

<div class="sample" markdown="1" mode="kotlin" theme="idea" data-highlight-only="1" auto-indent="false">

```kotlin
package example

object Object {
  val field = "A"
}

interface Interface {
  fun iMember() {}
}

class Clazz : Interface {
  fun member(p: Int): ULong? = 42UL
}

fun forIntegers(b: Byte, s: UShort, i: Int, l: ULong?) { }
fun forFloats(f: Float, d: Double?) { }

fun strings(str: String?) : String {
  return "That is '$str' from C"
}

fun acceptFun(f: (String) -> String?) = f("Kotlin/Native rocks!")
fun supplyFun() : (String) -> String? = { "$it is cool!" }
```
</div>


[[include pages-includes/docs/tutorials/native/lets-create-gradle-build.md]]
[[include pages-includes/docs/tutorials/native/apple-framework-code.md]]

已经准备好的工程源代码可以在这里直接下载：
[[include pages-includes/docs/tutorials/native/apple-framework-link.md]]

我们将源文件移动到工程下的
`src/nativeMain/kotlin` 文件夹。当使用 [kotlin-多平台](../../reference/mpp-discover-project.html)<!--
-->插件的时候这是定位文件的默认路径。
使用插件。我们使用以下代码块来指示配置项目<!--
-->为我们生成动态或共享库：

<div class="sample" markdown="1" theme="idea" mode="kotlin" data-highlight-only>

```kotlin
binaries {
  framework {
    baseName = "Demo"
  }  
}
```
</div>

`macOS X64` 是单独的，Kotlin/Native 支持 iOS `arm32`、`arm64` 与 `X64`
目标平台。我们可以使用各自的函数替代 `macosX64`，
如表所示：

| 目标 平台/设备           | Gradle 函数      |
|------------------------|-----------------|
| macOS x86_64           | `macosX64()`    | 
| iOS ARM 32             | `iosArm32()`    | 
| iOS ARM 64             | `iosArm64()`    | 
| iOS Simulator (x86_64) | `iosX64()`      |
{:.zebra}
 
我们运行 `linkNative` Gradle 任务以<!--
-->[在 IDE 中](using-intellij-idea.html)构建该库，
或者使用如下的控制台命令：
[[include pages-includes/docs/tutorials/native/linkNative.md]]

依据配置的不同，构建生成的 framework
位于
`build/bin/native/debugFramework`
或
`build/bin/native/releaseFramework`
文件夹。
我们来看看里面是什么

## 生成 Framework 头文件

每个创建的 framework 头文件都包含在 `<Framework>/Headers/Demo.h` 中。
这个头文件不依赖目标平台（至少需要 Kotlin/Native v.0.9.2）。
它包含我们的 Kotlin 代码的定义与一些 Kotlin 级的声明。

注意，Kotlin/Native 导出符号的方式如有变更，恕不另行通知。

### Kotlin/Native 运行时声明

我们<!--
-->首先来看看 Kotlin 的运行时声明：

<div class="sample" markdown="1" mode="obj-c" theme="idea" data-highlight-only auto-indent="false">

```obj-c
NS_ASSUME_NONNULL_BEGIN

@interface KotlinBase : NSObject
- (instancetype)init __attribute__((unavailable));
+ (instancetype)new __attribute__((unavailable));
+ (void)initialize __attribute__((objc_requires_super));
@end;

@interface KotlinBase (KotlinBaseCopying) <NSCopying>
@end;

__attribute__((objc_runtime_name("KotlinMutableSet")))
__attribute__((swift_name("KotlinMutableSet")))
@interface DemoMutableSet<ObjectType> : NSMutableSet<ObjectType>
@end;

__attribute__((objc_runtime_name("KotlinMutableDictionary")))
__attribute__((swift_name("KotlinMutableDictionary")))
@interface DemoMutableDictionary<KeyType, ObjectType> : NSMutableDictionary<KeyType, ObjectType>
@end;

@interface NSError (NSErrorKotlinException)
@property (readonly) id _Nullable kotlinException;
@end;
```
</div>

Kotlin 类在 Objective-C 中拥有一个 `KotlinBase` 基类，该类在这里继承自
`NSObject` 类。我们同样也有集合与异常的包装器。
大多数的集合类型都从另一边映射到了相似的集合类型：

|Kotlin|Swift|Objective-C|
-------|-----|-----------|
|List|Array|NSArray|
|MutableList|NSMutableArray|NSMutableArray|
|Set|Set|NSSet|
|Map|Dictionary|NSDictionary|
|MutableMap|NSMutableDictionary|NSMutableDictionary|
{:.wide.zebra}


### Kotlin Numbers 与 NSNumber

下一步，`<Framework>/Headers/Demo.h` 包含了 Kotlin/Native 数字类型与 `NSNumber`
之间的映射。我们在 Objective-C 中拥有一个基类名为 `DemoNumber`，
而在 Swift 中是 `KotlinNumber`。它继承自 `NSNumber`。
这里有每个作为子类的 Kotlin 数字类型：

|Kotlin|Swift|Objective-C| Simple type |
-------|-----|-----------|
|`-`      |`KotlinNumber` |`<Package>Number` | `-` |
|`Byte`   |`KotlinByte`   |`<Package>Byte`   | `char` |
|`UByte`  |`KotlinUByte`  |`<Package>UByte`  | `unsigned char` |
|`Short`  |`KotlinShort`  |`<Package>Short`  | `short` |
|`UShort` |`KotlinUShort` |`<Package>UShort` | `unsigned short` |
|`Int`    |`KotlinInt`    |`<Package>Int`    | `int` |
|`UInt`   |`KotlinUInt`   |`<Package>UInt`   | `unsigned int` |
|`Long`   |`KotlinLong`   |`<Package>Long`   | `long long` |
|`ULong`  |`KotlinULong`  |`<Package>ULong`  | `unsigned long long` |
|`Float`  |`KotlinFloat`  |`<Package>Float`  | `float` |
|`Double` |`KotlinDouble` |`<Package>Double` | `double` |
|`Boolean`|`KotlinBoolean`|`<Package>Boolean`| `BOOL/Bool` |
{:.wide.zebra}

每个数字类型都有一个类方法，用于从相关的简单类型创建新实例。此外，还有一个实例方法<!--
-->用于提取一个简单的值。原理上，声明看起来像这样：

<div clacss="sample" markdown="1" mode="obj-c" theme="idea" data-highlight-only auto-indent="false">

```obj-c
__attribute__((objc_runtime_name("Kotlin__TYPE__")))
__attribute__((swift_name("Kotlin__TYPE__")))
@interface Demo__TYPE__ : DemoNumber
- (instancetype)initWith__TYPE__:(__CTYPE__)value;
+ (instancetype)numberWith__TYPE__:(__CTYPE__)value;
@end;
```

</div>
其中 `__TYPE__` 是简单类型的名称之一，而 `__CTYPE__` 是相关的 Objective-C 类型，例如 `initWithChar(char)`。

这些类型用于将装箱的 Kotlin 数字类型映射到 Objective-C 与 Swift。
在 Swift 中，我们可以简单的调用构造函数来创建一个示例，例如 `KotlinLong(value: 42)`。

### Kotlin 中的类与对象

我们来看看如何将 `class` 与 `object` 映射到 Objective-C 与 Swift。 
生成的 `<Framework>/Headers/Demo.h` 文件包含
`Class`、`Interface` 与 `Object` 的确切定义：

<div class="sample" markdown="1" mode="obj-c" theme="idea" data-highlight-only="1" auto-indent="false">

```obj-c
NS_ASSUME_NONNULL_BEGIN

__attribute__((objc_subclassing_restricted))
__attribute__((swift_name("Object")))
@interface DemoObject : KotlinBase
+ (instancetype)alloc __attribute__((unavailable));
+ (instancetype)allocWithZone:(struct _NSZone *)zone __attribute__((unavailable));
+ (instancetype)object __attribute__((swift_name("init()")));
@property (readonly) NSString *field;
@end;

__attribute__((swift_name("Interface")))
@protocol DemoInterface
@required
- (void)iMember __attribute__((swift_name("iMember()")));
@end;

__attribute__((objc_subclassing_restricted))
__attribute__((swift_name("Clazz")))
@interface DemoClazz : KotlinBase <DemoInterface>
- (instancetype)init __attribute__((swift_name("init()"))) __attribute__((objc_designated_initializer));
+ (instancetype)new __attribute__((availability(swift, unavailable, message="use object initializers instead")));
- (DemoLong * _Nullable)memberP:(int32_t)p __attribute__((swift_name("member(p:)")));
@end;
```
</div>

这段代码有各种 Objective-C attribute，旨在提供在
Objective-C 与 Swift 语言中使用该 framework 的帮助。
`DemoClazz`、`DemoInterface`、`DemoObject` 被分别创建为 `Clazz`、`Interface` 与
`Object`。`Interface` 被转换为 `@protocol`，同样 `class` 与 `object` 都以
`@interface` 表示。
`Demo` 前缀来自于 `kotlinc-native`
编译器的 `-output` 参数与 framework 的名称。
我们看到这里的可空的返回值类型 `ULong?` 被转换到 Objective-C 中的 `DemoLong*`。

### Kotlin 中的全局声明

在 `Demo` 作为 framework 名称的地方且为 `kotlinc-native` 设置了 `-output` 参数时，
所有 Kotlin 中的全局声明<!--
-->都被转化为 Objective-C 中的 `DemoLibKt` 以及 Swift 中的 `LibKt`。

<div class="sample" markdown="1" mode="obj-c" theme="idea" data-highlight-only="1" auto-indent="false">

```obj-c
NS_ASSUME_NONNULL_BEGIN

__attribute__((objc_subclassing_restricted))
__attribute__((swift_name("LibKt")))
@interface DemoLibKt : KotlinBase
+ (void)forIntegersB:(int8_t)b s:(int16_t)s i:(int32_t)i l:(DemoLong * _Nullable)l __attribute__((swift_name("forIntegers(b:s:i:l:)")));
+ (void)forFloatsF:(float)f d:(DemoDouble * _Nullable)d __attribute__((swift_name("forFloats(f:d:)")));
+ (NSString *)stringsStr:(NSString * _Nullable)str __attribute__((swift_name("strings(str:)")));
+ (NSString * _Nullable)acceptFunF:(NSString * _Nullable (^)(NSString *))f __attribute__((swift_name("acceptFun(f:)")));
+ (NSString * _Nullable (^)(NSString *))supplyFun __attribute__((swift_name("supplyFun()")));
@end;
```
</div>

我们看到 Kotlin `String` 与 Objective-C `NSString *` 是透明映射的。
类似地，Kotlin 的 `Unit` 类型被映射到 `void`。我们看到原始类型<!--
-->直接映射。不可空的原始类型透明地映射。
可空的原始类型被映射到 `Kotlin<TYPE>*` 类型，如[上](#kotlin-numbers-and-nsnumber)表所示。
包括高阶函数 `acceptFunF` 与 `supplyFun`，
都接收一个 Objective-C 块。

更多的关于所有其它类型的映射细节可以在这篇
[Objective-C 互操作](/docs/reference/native/objc_interop.html)<!--
-->文档中找到。

## 垃圾回收与引用计数

Objective-C 与 Swift 使用引用计数。Kotlin/Native 也同样拥有自己的垃圾回收。
Kotlin/Native 的垃圾回收与 Objective-C/Swift
引用计数相集成。我们不需要在 Swift 或 Objective-C 中使用任何其它特别的方式去控制
Kotlin/Native 实例的生命周期。

## 在 Objective-C 中使用代码

让我们在 Objective-C 中调用代码。为此我们使用下面的内容创建
`main.m` 文件：

<div class="sample" markdown="1" mode="obj-c" theme="idea" data-highlight-only="1" auto-indent="false">

```obj-c 
#import <Foundation/Foundation.h>
#import <Demo/Demo.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        [[DemoObject object] field];
        
        DemoClazz* clazz = [[ DemoClazz alloc] init];
        [clazz memberP:42];
        
        [DemoLibKt forIntegersB:1 s:1 i:3 l:[DemoULong numberWithUnsignedLongLong:4]];
        [DemoLibKt forIntegersB:1 s:1 i:3 l:nil];
        
        [DemoLibKt forFloatsF:2.71 d:[DemoDouble numberWithDouble:2.71]];
        [DemoLibKt forFloatsF:2.71 d:nil];
        
        NSString* ret = [DemoLibKt acceptFunF:^NSString * _Nullable(NSString * it) {
            return [it stringByAppendingString:@" Kotlin is fun"];
        }];
        
        NSLog(@"%@", ret);
        return 0;
    }
}
```
</div>

我们直接在 Objective-C 代码中调用 Kotlin 类。一个 Kotlin `object` 拥有类<!--
-->方法函数 `object`，这使我们在想获取对象的唯一实例时调用
`object` 方法就可以了。
广泛使用的用于创建 `Clazz` 类实例的模式。我们在
Objective-C 上调用 `[[ DemoClazz alloc] init]`。我们也可以使用<!--
-->没有参数的构造函数 `[DemoClazz new]`。
Kotlin 源中的全局声明的作用域位于 Objective-C 中的 `DemoLibKt` 类之下。
所有的方法都被转化为该类的类方法。
`strings` 函数转化为 Objective-C 中的 `DemoLibKt.stringsStr`函数，我们可以<!--
-->给它直接传递 `NSString`。而返回值也同样可以看作 `NSString`。

## 在 Swift 中使用代码

这个使用 Kotlin/Native 编译的 framework 拥有辅助 attribute
来使它在 Swift 中的使用更为容易。我们来将之前的 Objective-C 示例覆盖为
Swift。其结果是，我们将在 `main.swift` 中包含下面的代码：

<div class="sample" markdown="1" mode="swift" theme="idea" data-highlight-only="1" auto-indent="false">

```swift
import Foundation
import Demo

let kotlinObject = Object()
assert(kotlinObject === Object(), "Kotlin object has only one instance")

let field = Object().field

let clazz = Clazz()
clazz.member(p: 42)

LibKt.forIntegers(b: 1, s: 2, i: 3, l: 4)
LibKt.forFloats(f: 2.71, d: nil)

let ret = LibKt.acceptFun { "\($0) Kotlin is fun" }
if (ret != nil) {
  print(ret!)
}
``` 
</div>

这段 Kotlin 代码被转换为看起来非常相似的
Swift 代码。但是它们有一些小差异。在 Kotlin 中任何 `object`
只拥有一个实例。Kotlin `object Object` 现在在
Swift 中拥有一个构造函数，我们使用 `Object()` 语法来访问它唯一的实例。
在 Swift 中该实例总是相同的，所以 
`Object() === Object()` 为 true。 
方法与属性名称按原样转换。即 Kotlin `String` 被转换为 Swift `String`。
Swift 同样隐藏了 `NSNumber*` 的装箱。我们传递 Swift 闭包给 Kotlin，
与在 Swift 中调用 Kotlin lambda 函数是相同的。

更多关于类型映射的信息可以在这篇
[Objective-C 互操作](/docs/reference/native/objc_interop.html)<!--
-->文档中找到。

# Xcode 与 Framework 依赖

我们需要配置一个 Xcode 工程来使用我们的 framework。这个配置依赖于<!--
-->目标平台。

## Xcode 与 MacOS 目标平台

首先，我们需要导入该 framework 到 `General` 选项的 *target*
配置。选择 `Linked Frameworks and Libraries` 选项来导入<!--
-->我们的 framework。这将使 Xcode 查看我们的 framework 并同时解决来自
Objective-C 与 Swift 的导入。

第二步是配置 framework 生产的二进制文件的<!--
-->搜索路径。它也被称作 `rpath` 或[运行时搜索路径](https://en.wikipedia.org/wiki/Rpath)。
二进制文件使用路径来查找所需的 framework。我们不推荐，
在不需要的时候在操作系统中去安装其它 framework。我们应该了解<!--
-->未来应用程序的布局，举例来说，
我们可能在应用程序包下有 `Frameworks` 文件夹，其中包含我们所使用的所有 framework。
`@rpath` 参数可以被配置到 Xcode。我们需要打开
*project* 配置项并找到 `Runpath Search Paths` 选项。在这里我们指定编译
framework 的相对路径。

## Xcode 与 iOS 目标平台

首先，我们需要导入编译好的 framework 到 Xcode 工程。为此<!--
-->我们将 framework 添加到 *target* 配置页的 `General` 选项的
`Embedded Binaries` 块。

第二步是将 framework 路径导入到 *target* 配置页的 `Build Settings` 选项下的
`Framework Search Paths` 块。它可以使用 `$(PROJECT_DIR)`
宏来简化设置。
 
iOS 模拟器需要一个为 `ios_x64` 目标平台编译的 framework，它位于我们案例中的
`iOS_sim` 文件夹。

[这个 Stack Overflow 主题](https://stackoverflow.com/questions/30963294/creating-ios-osx-frameworks-is-it-necessary-to-codesign-them-before-distributin)<!--
-->包含了一些更多的建议。同样，
[CocoaPods](https://cocoapods.org/) 包管理器也可能有助于自动化该过程。

# 接下来

Kotlin/Native 与 Objective-C 以及 Swift 语言之间拥有双向互操作性。
Kotlin 对象集成了 Objective-C/Swift 的引用计数。Kotlin
对象可以被自动释放。
这篇 [Objective-C 互操作](https://github.com/JetBrains/kotlin-native/blob/master/OBJC_INTEROP.md)<!--
-->文档包含了更多关于互操作实现细节的信息。
当然，也可以导入一个外部的 framework 并在 Kotlin 中使用它。Kotlin/Native
附带一套良好的预导入系统 framework。

Kotlin/Native 同样支持 C 互操作。查看这篇
[Kotlin/Native 开发动态库](dynamic-libraries.html)<!--
-->教程，或者查看这篇
[C 互操作](/docs/reference/native/c_interop.html)文档

