---
layout: post
title: Kotlin Multiplatform 跨平台支持鸿蒙
categories: [KMP]
description: 使用 KMP 的 Kotlin/JS 能力支持鸿蒙。
keywords: Kotlin, Harmony
mermaid: false
sequence: false
flow: false
mathjax: false
mindmap: false
mindmap2: false
topmost: true
---

## 引言

在国内，移动端支持鸿蒙平台已经不是什么新鲜事了。但对于公司来说，是否要支持鸿蒙平台，主要考虑的是成本和收益。收益很明确，挽留华为用户和防止华为用户丢失，以这个为底线，如果能带来新的用户，那当然最好。对于成本，也很明确，把在 Android 和 iOS 上做的事情，在鸿蒙平台上重新做一遍，以前1人/10天，现在加上熟悉平台成本，1人/12天。以这个粗略评估下来，对于大型应用程序（小红书、美团、微博等），成本是昂贵的，假设投入10人，100天，一天费用1万，那么至少得花费 1000万 才能支持鸿蒙平台，以后成本还得继续增加。为了降低成本或防止成本增加，有没有办法把在 Android，iOS，HarmonyOs要做的事情只做一遍？这就需要寻找跨平台技术方案，让代码在多平台共享。

### 关于 Kotlin Multiplatform

上一篇文章：[ 采用 Kotlin Multiplatform 做跨平台]({{site.url}}/2024/06/01/Kotlin-Multiplatform-Intro/)，做了 Kotlin Multiplatform 跨平台技术调研，知道 Kotlin Multiplatform 做跨平台主要分为两部分：

* **逻辑 代码共享**：使用 Kotlin Multiplatform 稳定支持 Android、iOS、Desktop、Server、Web(Kotlin/JS) 平台
* **UI 代码共享**：使用 Compose Multiplatform 稳定支持 Android、Desktop（JVM）平台，也可以在 iOS 平台使用

这里，主要探讨**逻辑 代码共享**。Kotlin Multiplatform 已经具备网络，数据库，协程，序列化，日期和时间等跨平台基础。使用这些能力，就可以将：网络、上传、下载，登录等基础组件迁移到 Kotlin Multiplatform，以支持 Android 和 iOS，不必再各自维护一套。

Kotlin Multiplatform 跨平台代码共享，给 Android 使用的产物是 JVM 字节码，给 iOS 使用的产物是本机二进制文件，那么给鸿蒙使用的产物应该是什么呢？

可以利用 Kotlin Multiplatform 的 Kotlin/JS 能力，输出产物 .js 和 .d.ts 文件给鸿蒙使用。

### 关于鸿蒙

在鸿蒙平台，能直接导入 .js 和 .d.ts 文件使用吗？

鸿蒙平台开发应用程序使用的是 ArkTS 开发语言：

> ArkTS是HarmonyOS优选的主力应用开发语言。ArkTS围绕应用开发在TypeScript（简称TS）生态基础上做了进一步扩展，继承了TS的所有特性，是TS的超集。因此，在学习ArkTS语言之前，建议开发者具备TS语言开发能力。

ArkTS 既然是 TypeScript 的超集，而 TypeScript 又是 JavaScript 的超集：

> [How does TypeScript relate to JavaScript, though?](https://www.typescriptlang.org/docs/handbook/typescript-from-scratch.html)
> TypeScript is a language that is a superset of JavaScript: JS syntax is therefore legal TS. Syntax refers to the way we write text to form a program.

那么，在语言上，ArkTS 兼容 TS 和 JS。接下来，只需要让鸿蒙识别 .js 和 .d.ts 文件-鸿蒙天生就可以。

## 开发环境准备

这里不再说如何配置开发环境。可参考：

* [KMP 环境配置](https://www.jetbrains.com/help/kotlin-multiplatform-dev/multiplatform-setup.html#check-your-environment)
* [鸿蒙 环境配置](https://developer.huawei.com/consumer/cn/doc/harmonyos-guides-V2/environment_config-0000001052902427-V2)

下面为本地环境。

### KMP 开发环境

配置开发环境时间较长，最好有梯子。

本地环境：macOs 14.3.1，Java(openjdk 17.0.11)，Android Studio(Jellyfish | 2023.3.1 RC 2)，Xcode(15.3)，CocoaPos(非必需)

在 Android Studio 中还需要安装插件：[Kotlin Multiplatform](https://plugins.jetbrains.com/plugin/14936-kotlin-multiplatform)。

![screenshot.png](/images/posts/2024-06-12-Kotlin-Multiplatform-Support-Harmony/p1.png)

配置好环境后，下载项目[Kotlin Multiplatform Wizard](https://kmp.jetbrains.com/?_gl=1*hk8hbt*_ga*MTYxOTI0MjI0Mi4xNzAyMzc1MjM1*_ga_9J976DJZ68*MTcxNzU1MjQ5OC40Ni4xLjE3MTc1NTYxNTcuMTQuMC4w&_ga=2.51358825.1924311792.1717042362-1619242242.1702375235)，**勾选 Desktop**。下载完成后，使用 Android Studio 打开即可。

除了 Android Studio，也可以使用 [Fleet](https://www.jetbrains.com/fleet/)。

### 鸿蒙开发环境

参考官方指导文档操作即可。

![截屏2024-06-05 10.49.46.png](/images/posts/2024-06-12-Kotlin-Multiplatform-Support-Harmony/p2.png)

配置好环境后，新建一个项目。

## 开始

流程：在 Android Studio 中，把 Kotlin Multiplatform (Kotlin/JS) 使用 Kotlin 编写的逻辑共享代码导出为 .js 和 .d.ts 文件，然后在 DevEco Studio 中导入 .js 和 .d.ts 文件，再使用逻辑共享代码并运行应用程序。

* AndroidStudio → KMP → Kotlin/JS → export →**`.js & .d.ts`** ← import ← DevEco Studio ← 鸿蒙应用程序

下面以 Kotlin/JS 实现一个简单定时器给鸿蒙用为例子。

### Kotlin/JS

项目插件版本：

* agp：8.2
* kotlin：2.0

项目结构：

<img title="" src="file:///images/posts/2024-06-12-Kotlin-Multiplatform-Support-Harmony/p3.png" alt="" width="" height="">

* composeApp 目录：为 UI 共享代码
* shared 目录：为逻辑共享代码

这里，主要关注逻辑共享代码 shared 目录，在这个目录下:

* androidMain 目录：为 Android 平台独立使用的代码
* commonMain 目录：为 所有平台 共享的代码
* iosMain 目录：为 iOS 平台独立使用的代码
* jsMain 目录：为 Web base on Kotlin/JS 使用的代码
* jvmMain 目录：为 Desktop 平台使用的代码
* wasmJsMain 目录：为 Web base on Kotlin/Wasm 使用的代码

下面，在 commonMain 目录下实现一个简单定时器，`Ticker.kt`：

```Kotlin
class Ticker {

    private val coroutineScope = CoroutineScope(SupervisorJob() + Dispatchers.Default)

    fun start(duration: Int, onTick: (Int) -> Unit) {
        this.coroutineScope.launch(Dispatchers.Main) {
            for (i in 1..duration) {
                delay(1000) // 每秒钟触发一次
                onTick(i) // 调用回调函数，传递已过去的秒数
            }
        }
    }

    fun stop() {
        this.coroutineScope.cancel()
    }
}
```

该实现利用了 Kotlin 协程（[github](https://github.com/Kotlin/kotlinx.coroutines)），需要添加依赖：

```kotlin
commonMain {
    dependencies {
        implementation("org.jetbrains.kotlinx:kotlinx-coroutines-core:1.9.0-RC")
    }
}
jsMain {
    dependencies {
        implementation("org.jetbrains.kotlinx:kotlinx-coroutines-core-js:1.9.0-RC")
    }
}
```

逻辑共享代码已经有了，还需要搞定：

1. Kotlin 代码给 JS 使用
2. 构建 Kotlin 代码的 .js 和 .d.ts 产物

#### 在 JS 中使用 Kotlin 代码

在 Android 开发中， Android ⇆ JS 代码可以互相调用。同样在 KMP 中，也提供了 Kotlin ⇆ JS 代码互相调用。

要实现 JS 调用 Kotlin 代码，其实很简单，只需要使用 Kotlin 注解 `@JsExport` 和 `@JsName`(可选)即可。

##### @JsExport 注解

使用 `@JsExport`注解标记接口、类或函数后，就可以导出给 JavaScript 使用。

标记类：

```Kotlin
@file:JsExport
@file:OptIn(ExperimentalJsExport::class)

import kotlin.js.ExperimentalJsExport
import kotlin.js.JsExport
import kotlin.js.JsName

@OptIn(ExperimentalJsExport::class)
@JsExport
class Ticker {
}
```

标记函数：

```kotlin
import kotlin.js.ExperimentalJsExport
import kotlin.js.JsExport

@OptIn(ExperimentalJsExport::class)
@JsExport
fun addOne(x: Int) = x + 1
```

`@JsExport` 注解的作用是：告诉 `Kotlin/JS IR compiler backend`（编译器） 需要将 Kotlin 代码转换为 JavaScript 代码。

将 Kotlin 代码转换成 JavaScript 代码，有一个重要的问题就是类型映射。并不是 Kotlin 的所有类型都能转换成 JavaScritp 所支持的类型。比如：Kotlin 的 Long 类型，就不能和 JavaScript 匹配，所以在共享代码中不能使用 Long 类型。这里不一一例举类型映射，感兴趣可以查看文档 [Use Kotlin code from JavaScript](https://kotlinlang.org/docs/js-to-kotlin-interop.html#jsexport-annotation)。

另外，要在鸿蒙平台使用，也要考虑 ArkTS 所支持的类型。例如：Kotlin 的 Map 类型，不能和 ArkTS 匹配，因此也不能在共享代码中使用 Map 类型。

##### @JsName 注解

`@JsName` 注解是配合 `@JsExport` 注解使用的。它的作用是指明导出的类和函数的名字，比如：

```Kotlin
@OptIn(ExperimentalJsExport::class)
@JsExport
class Ticker {

   @JsName("cancel")
   fun stop(){
   }
}
```

那么`Ticker.stop`在 JavaScript 中使用时就是 `Ticker.cancel`。

#### Gradle 构建产物

##### 构建配置选项

要将 `@JsExport` 注解标记的接口、类和函数，构建出 .js 和 .d.ts 产物。主要关注 `build.gradle.kts`和`gradle.properties` 配置文件。

在 build.gradle.kts 中，配置选项为 `js{}`:

```Kotlin
kotlin {
    js(IR) {
        moduleName = "kmp-shared"
        compilations.all {
            compileTaskProvider.configure {
                compilerOptions.freeCompilerArgs.add("-Xerror-tolerance-policy=SYNTAX")
            }
            if (this.compilationName == "main") {
                packageJson {
                   name = "kmp-shared"
                   version = "0.0.1"
                }
            }
        }
        nodejs()
        binaries.executable()
        generateTypeScriptDefinitions()
    }

    sourceSets {
        commonMain.dependencies {
            implementation(libs.kotlinx.coroutines.core)
        }
        jsMain.dependencies {
            dependencies {
                implementation(libs.kotlinx.coroutines.core.js)
            }
        }
    }
}
```

在`js{}`中指明 Kotlin 代码编译成 JavaScript 代码相关配置：

* `js(IR)`中 IR 表示：`Kotlin/JS IR compiler backend`（编译器），Kotlin/JS IR compiler backend 不是直接从 Kotlin 源代码生成 JavaScript 代码，而是采用了一种新方法。首先将 Kotlin 源代码转换为[Kotlin 中间表示 (IR)](https://kotlinlang.org/docs/whatsnew14.html#unified-backends-and-extensibility)，然后将其编译为 JavaScript。IR 是 Kotlin 1.4 版本引入的，之前的版本还有 LEGACY，所以 js(X)，X 可以是 IR、LEGACY、BOTH。

* `moduleName`表示：模块名称，比如moduleName="kmp-shared"，那么产物名称就是 kmp-shared.js 和 kmp-shared.d.ts

* `nodejs()`表示：用于在浏览器之外运行 JavaScript 代码

* `binaries.executable()`表示：生成可执行的 .js 文件

* `generateTypeScriptDefinitions()`表示：编译器将收集有声明` @JsExport`注解的接口、类、函数，自动在 .d.ts 文件中生成 TypeScript 定义

* `compilerOptions.freeCompilerArgs.add`表示：添加编译选项

* `-Xerror-tolerance-policy=SYNTAX｜SEMANTIC` 表示：忽略编译错误。其中`SYNTAX`：编译器将接受任何代码，即使它包含语法错误。无论编写什么，编译器仍将尝试生成可运行的可执行文件；`SEMANTIC`：编译器会接受语法正确但语义上不合理的代码。例如，将数字赋值给字符串变量（类型不匹配）

* `packageJson`表示：生成的 JavaScript 包的配置清单信息。信息存储在 package.json 中，包含：名称、版本、许可证、依赖性和其它相关元数据信息。其中`name`表示包的唯一标识符，`version`表示版本号。另外，也可以添加自定义元数据

在 gradle.properties 中，可以添加配置选项：

* `kotlin.incremental.js.ir=false` ：值 true 表示开发二进制文件开启增量编译，否则关闭，默认 true
* `kotlin.js.ir.output.granularity=whole-program` ：值 whole-program 表示每个项目的编译产物只有一个 .js 文件，值 pre-module 表示每个模块一个 .js 文件，默认 pre-module

##### 构建命令

在 gradle 中构建执行 Node.js 脚本的任务(gradle task)有：

```Shell
Kotlin node tasks
-----------------
jsNodeDevelopmentRun 
jsNodeProductionRun
jsNodeRun
```

* jsNodeDevelopmentRun 表示：在**开发阶段**构建执行 Node.js 脚本
* jsNodeProductionRun 表示：在**生产阶段**构建执行 Node.js 脚本
* jsNodeRun 表示：同时执行 jsNodeDevelopmentRun 和 jsNodeProductionRun

在命令行执行 `./gradlew :shared:jsNodeRun`，执行后在 **`根项目/build/js/packages/kmp-shared`** 目录下会生成 `.js 和 .d.ts` 相关产物：



<table>
  <tbody><tr>
      <td><img src="../../../../images/posts/2024-06-12-Kotlin-Multiplatform-Support-Harmony/p4.png"></td>
          <td><img src="../../../../images/posts/2024-06-12-Kotlin-Multiplatform-Support-Harmony/p5.png"></td>
 </tr>
</tbody></table>

上面图中展示的产物区别，是由 gradle.properties 中配置`kotlin.js.ir.output.granularity=pre-module ｜ whole-program` 控制的。假如目标项目已经导入了 kotlin-coroutines-core.js，那么可以选择
`pre-module` 配置生成产物，再导入剩余产物即可。如果目标项目所需依赖都来源于 KMP，那么选择 `whole-program` 配置即可。

这里选择 `whole-program` 配置，kmp-shared.js 为可执行的 JavaScript 代码，kmp-shared.d.ts 为 TypeScript 定义。

根据上面 @JsExport 和 @JsName 注解标记的类和函数，`kmp-shared.d.ts` 为：

```TypeScript
type Nullable<T> = T | null | undefined
export declare function addOne(x: number): number;
export declare class Ticker {
    constructor();
    start(duration: number, onTick: (p0: number) => void): void;
    cancel(): void;
}
export as namespace KotlinProject_shared;
```

现在已经得到 KMP 的导出产物： `kmp-shared.js` 和 `kmp-shared.d.ts`，那么接下来在鸿蒙中导入产物使用。

### 鸿蒙

在鸿蒙项目中，新建立模块 shared，并把 `kmp-shared.js` 和 `kmp-shared.d.ts` 放在 `shared/src/main/ets `目录下：

<img title="" src="file:///images/posts/2024-06-12-Kotlin-Multiplatform-Support-Harmony/p6.png" alt="" width="" height="">

如果要在 entry 模块下使用 `kmp-shared` ，需要：

1. 在 shared/Index.ets 中声明该模块对外提供的能力：

```js
export * from './src/main/ets/kmp-shared'
```

2. 在 entry 模块中添加 shared 模块依赖，修改 entry/oh-package.json5 的 dependencies：

```js
"dependencies": {
  "shared": "file:../shared"
}
```

#### Demo

代码示例，entry/src/main/ets/Index.ets：

```TypeScript
import { Ticker } from 'shared/src/main/ets/kmp-shared'
import { display } from '@kit.ArkUI'

@Entry
@Component
struct Index {
  private duration: number = 60
  @State private currentDuration: number = 1
  private screenWidth: number = 0
  private jsTicker: Ticker | undefined = undefined

  aboutToAppear(): void {
    this.screenWidth = display.getDefaultDisplaySync().width
    this.jsTicker = new Ticker()
    this.jsTicker.start(this.duration, (curDuration: number) => {
      this.currentDuration = curDuration
    })
  }

  aboutToDisappear(): void {
    this.jsTicker?.cancel()
  }

  build() {
    RelativeContainer() {
      Column() {
        Stack() {
          Circle()
            .stroke(0xFF2196F3)
            .strokeLineJoin(LineJoinStyle.Round)
            .strokeWidth(2)
            .strokeDashArray([5, 5])
            .fill(Color.Transparent)
            .width(px2vp(this.screenWidth / 2))
            .height(px2vp(this.screenWidth / 2))

          Text(this.currentDuration.toString())
            .id('TickTime')
            .fontSize(50)
            .fontWeight(FontWeight.Bold)
            .fontColor(0xFF60DDAD)
            .align(Alignment.Center)
        }

        Text('Hello: Harmony with Kotlin/JS!')
          .id('HelloWorld')
          .fontSize(20)
          .fontColor(0xFF2196F3)
          .padding({ top: 20, bottom: 10 })

        Text('@划水健儿')
          .id('MyName')
          .fontSize(20)
          .fontColor(0xFF2196F3)
      }.alignRules({
        center: { anchor: '__container__', align: VerticalAlign.Center },
        middle: { anchor: '__container__', align: HorizontalAlign.Center }
      })
    }
    .height('100%')
    .width('100%')
  }
}
```

效果示例：

![1718114896582141.gif](/images/posts/2024-06-12-Kotlin-Multiplatform-Support-Harmony/p7.gif)

## 总结

采用 Kotlin Multiplatform 做跨平台，可以使用其逻辑代码共享能力，不仅能稳定支持移动端 Android 和 iOS 平台，还能稳定支持鸿蒙平台，这对于国内来说是天时地利。

虽然使用 Kotlin/JS 支持鸿蒙平台的开发环境还不是那么丝滑（KMP 和 鸿蒙项目来回切换，才能进行开发和调试），但是已经看到 KMP 的巨大威力和潜力。

目前，可以使用 KMP 开发一些如日志，网络，上传，下载等基础组件，然后给 Android，iOS，鸿蒙平台使用。随着 KMP 的发展，跨平台能力越来越强，也许真能 Write once, run anywhere。

***

参考文档：

1. [Kotlin/JS](https://kotlinlang.org/docs/js-project-setup.html#execution-environments)
2. [Use Kotlin code from JavaScript](https://kotlinlang.org/docs/js-to-kotlin-interop.html#jsexport-annotation)
3. [Multiplatform Dsl Reference](https://kotlinlang.org/docs/multiplatform-dsl-reference.html)
