---
layout: post
title: Kotlin Multiplatform 封装鸿蒙 API
categories: [Java, Kotlin]
description: 将鸿蒙API的 .d.ts 文件导出，使用 Dukat 或 Karakum 将 .d.ts 文件转换为 .kt 文件，在 KMP 项目中导入 .kt 文件，此时就可以是使用 expect 和 actual 访问鸿蒙平台特性。
keywords: 跨平台
mermaid: false
sequence: false
flow: false
mathjax: false
mindmap: false
mindmap2: false
---

## 前言

现在采用 Kotlin Multiplatform 做跨平台逻辑共享，主要聚焦在 Android 和 iOS 移动端平台。但在国内，鸿蒙平台（手机，平板，PC等）的加入，不得不考虑让 Kotlin Multiplatfrom 支持鸿蒙，否则在 Android 和 iOS 上做的事，又得在鸿蒙上重做一遍，花费人力和财力。

之前在[ # Kotlin Multiplatform 跨平台支持鸿蒙]({{site.url}}/2024/06/12/Kotlin-Multiplatform-Support-Harmony/)中介绍了利用 Kotlin/JS 能力，在 KMP 项目中导出 .js 和 .d.ts 产物给鸿蒙使用，以实现逻辑跨平台共享。虽然在现实项目中， 产物过大或性能不好等，但终是达到了跨平台目的。当然也可以利用 Kotlin/Natvie 去支持鸿蒙，这是另外一个话题。

在开发 KMP 项目中，单纯的业务逻辑，只需要放在 `commonMain` 中，所有平台（`androidMain`，`iosMain`，`jsMain`）直接共享。但遇到需要访问平台特性，就得平台各自实现，比如访问日志，文件，网络等。现在在 `androidMain`和 `iosMain` 中是默认能访问 Android 和 iOS 平台特性的，但在 `jsMain` 中能不能访问鸿蒙平台特性？如果不能访问，那么基础库（日志，文件，网络等）逻辑共享就不得不采用一些不优雅的方式支持鸿蒙（比如定义很多接口，让鸿蒙去实现，这将破坏应有的设计）。如果能访问，那该怎么访问？

在 Kotlin/JS 中，Kotlin 与 JavaScript 语言具有互操作性，也就是在 Kotlin 中能使用 JavaScript ，在 JavaScript 中也能使用 Kotlin。那么，要访问鸿蒙平台特性，就需要鸿蒙提供相关 JavaScript 才行，遗憾的是，鸿蒙并没有对外提供相关日志、文件、网络等 JavaScript 库。那这就不得不，强制让鸿蒙给出 JavaScript，然后在 `jsMain` 中访问鸿蒙平台特性。

当在 KMP 项目中，能访问鸿蒙平台特性后，那么共享逻辑基础库就能无缝对接各平台，这让地基坚固牢靠：

<img src="{{site.url}}/images/posts/2024-12-05-KMP-With-Harmony-API/p1.png" width="100%" height="100%" />

对于鸿蒙平台的主要流程是：

<img src="{{site.url}}/images/posts/2024-12-05-KMP-With-Harmony-API/p2.png" width="100%" height="100%" />


## JS 相关基础概念

要实现封装鸿蒙API，需要了解一下 JS 相关基础概念。

### commonJS、ES Module、AMD、UMD模块化规范

[CommonJS](https://wiki.commonjs.org/wiki/CommonJS) 是一种模块化规范，最早由 Node.js 在2009年采用，是 Node.js 的默认模块系统。

主要特点：

*   使用 `require()` 导入模块。
*   使用 `module.exports` 或 `exports` 导出模块。

ES Module (ESM)是 [ECMAScript](https://262.ecma-international.org/6.0/) 标准的模块化系统，最早在 ES6（2015）引入，浏览器和 Node.js 都支持。

主要特点：

*   使用 `import` 导入模块。
*   使用 `export` 导出模块。

[AMD](https://github.com/amdjs/amdjs-api/wiki/AMD)（Asynchronous Module Definition）：异步模块定义。AMD 是为了浏览器环境优化的模块系统。

主要特点：

*   使用 `define` 定义模块，使用 `require` 引入模块。

[UMD](https://github.com/umdjs/umd)(Universal Module Definition)：通用模块定义。UMD 是为了解决不同 JavaScript 环境（如 Node.js、浏览器、AMD）之间的兼容性问题而创建的模块系统。

主要特点：

*   兼容 CommonJS 和 AMD。

*   可以根据运行环境自动选择合适的加载方式。

*   适用于需要在不同环境下运行的库。

### ES5 和 ES6

ES5（ECMAScript 5）和 ES6（ECMAScript 6，也称为 ES2015）都是 JavaScript 的两个重要版本。ES5在2009年发布，ES6在2015年发布。ES6 是对 ES5 的一次重大升级，但它们在语法、特性和功能上有着显著的差异。
ES6 引入了许多新的语言特性，使得 JavaScript 更加强大和现代化。

### .js 和 .mjs 文件

.js 文件是 JavaScript 文件的标准扩展名，通常代表 CommonJS 模块系统。.mjs 文件是 ES 模块的专属文件。但ES5 和 ES6 也都可以使用 .js 文件来编写和保存 JavaScript 代码。

### [nodejs](https://dev.nodejs.cn/learn/)

> Node.js 是一个开源和跨平台的 JavaScript 运行时环境。 它是几乎任何类型项目的流行工具！
>
> Node.js 在浏览器之外运行 V8 JavaScript 引擎（Google Chrome 的内核）。 这使得 Node.js 非常高效。

## 鸿蒙 和 Kotlin/JS 对JS的支持

鸿蒙对[JS](https://developer.huawei.com/consumer/cn/doc/atomic-ascf-V5/logical-layer-js-V5)的支持：

> 支持 ES6 的 module 标准，使用 import 引入 js 依赖，同时支持 CommonJs 规范，使用require引入 js 依赖。
>
> 注意
>
> *   不支持使用 eval 执行 JS 代码。
> *   不支持使用 new Function 创建函数。

Kotlin/JS 对[JS](https://kotlinlang.org/docs/js-overview.html)的支持：

> Kotlin/JS 提供了转换 Kotlin 代码、Kotlin 标准库的能力，并且兼容 JavaScript 的任何依赖项。Kotlin/JS 的当前实现以 [ES5](https://www.ecma-international.org/ecma-262/5.1/) 为目标。
>
> 对于ES Module，默认是支持 ES5，但也支持 ES6。
>
> 另外，Kotlin/JS支持的[JavaScript Modules](https://kotlinlang.org/docs/js-modules.html)也包含： commonJS、AMD、UMD。

## 在鸿蒙中假设性尝试

首先，在鸿蒙中，[ArkTS支持的模块化规范](https://developer.huawei.com/consumer/cn/doc/harmonyos-guides-V13/module-principle-V13#commonjs%E4%B8%8Ees-module%E6%94%AF%E6%8C%81%E8%A7%84%E6%A0%BC)遵循ES Module 和 commonJS 模块。其次，ArkTS 使用 `import` 导入模块，使用 `export` 导出模块。所以这里以 ES Module模块来尝试。

在鸿蒙中的日志API`@ohos.hilog.d.ts`为：

```ts

declare namespace hilog{...}

export default hilog;

```

所以，如果要让 KMP 项目中的 `jsMain`访问鸿蒙平台日志特性，那么导出产物 .js 文件中，应该有

```js
import hilog from '@ohos.hilog';
```

现在，假设 KMP 项目中导出的 test.js 文件为：

```js
import hilog from '@ohos.hilog';

export function testLog() {
    hilog.debug(0xFFFF, "Kotlin", "This is from Kotlin/JS.")
}
```

test.d.ts 文件为：

```ts
export declare function testLog(): void;
```

将 test.js 和 test.d.ts 文件导入鸿蒙项目中的 shared 模块，并在 shared 模块 index.ets 中添加：

    export * from './src/main/ets/test'

在业务模块中使用 `testLog`，Index.ets：

```ts
import { testLog } from 'shared/src/main/ets/test'

@Entry({ routeName: ROUTE_NAME_KOTLIN_JS_PAGE })
@Component
struct Index {

   aboutToAppear(): void {
      testLog()
   }
}
```

运行，控制台输出：

```shell
12-01 22:42:49.528   9852-9852     A0FFFF/com.exa...cation/Kotlin  com.examp...lication  D     This is from Kotlin/JS
```

从输出结果，可以看出，只要 KMP 项目能够正确导出包含鸿蒙平台特性的 .js 和 .d.ts 文件，是可以在鸿蒙平台正常运行的。

接下来，将探究在 KMP 项目中如何去访问鸿蒙平台特性，构建相关基础库，如日志，文件，网络等。

## KMP 封装鸿蒙API

### 在 Kotlin 中使用 JavaScript

在 Kotlin 中使用 JavaScript 有两种方式：

1.  内联 [js()](https://kotlinlang.org/api/core/kotlin-stdlib/kotlin.js/js.html)
2.  修饰符 external

### 内联 JavaScript

使用 `external fun js(code: String): dynamic` ，可以将 JavaScript 代码内联到 Kotlin 中。参数 `code` 为 JavaScript 代码片段，在编译解析时会按照原样转换为 JavaScript 代码。
比如获取 uuid：

```kotlin
fun generateUUID(): String = js(
    """
    globalThis.crypto && globalThis.crypto.randomUUID
        ? globalThis.crypto.randomUUID()
        : 'FallbackUUID'
    """
) as String
```

需要注意的是，js() 返回值类型为 `dynamic`，在编译时不提供类型安全性。

### 修饰符 external

使用修饰符 `external` 修饰类，函数，属性等时，编译器会假定这些是由外部提供的（由开发人员或通过 npm 依赖项），不会将这些生成任何 JavaScript 代码，因此，使用 external 声明没有任何实体。

使用 npm 包 `uuid` 获取 uuid：

```kotlin
external object UUID {
    fun v4(): String
}
```

通过 external 修饰符使用 JavaScript 时，需要知道 JavaScript 代码如何转换为 Kotlin 代码。

从内联js()和修饰符 external两种方式中，在访问鸿蒙平台特性时，选择修饰符external的方式。

### JavaScript 模块

KMP 项目的产物适用于commonJS、ES Module、AMD、UMD模块化规范：

```koltin
enum JsModuleKind{
    MODULE_AMD,

    MODULE_PLAIN,

    MODULE_ES,

    MODULE_COMMONJS,

    MODULE_UMD;
}
```

默认是 MODULE\_UMD。

可以在 build.gradle.kts 设置模块为 MODULE\_ES：

```kotlin
compilerOptions.moduleKind.set(org.jetbrains.kotlin.gradle.dsl.JsModuleKind.MODULE_ES)
```

#### JsModule 注解

如果要告诉 Kotlin，external 修饰的类，函数，属性等是 JavaScript 模块，可以使用` @JsModule` 注解。

```kotlin
@JsModule("uuid")
external object UUID {
    fun v4(): String
}
```

当一些 JavaScript 库导出是包（命名空间），而不是类，函数，属性等时，需要使用注解`@file:JsModule`，放在文件顶部。

```Kotlin
@file:JsModule("ohos.hilog")
```

标有 @file:JsModule 注释的文件不能声明非外部成员。

#### JsNonModule 注解

当 @JsModule("module")中的 module 不是标准 JavaScript 模块，需要添加注解`@JsNonModule`。

### 鸿蒙 API 转换为  Kotlin

在 KMP 项目的 `jsMian`中要访问鸿蒙平台特性，需要结合 external 修饰符和 @JsModule注解或及@JsNonModule注解。但前提是，先要将鸿蒙平台 API 转换为 Kotlin。

在鸿蒙中，提供的 API 都在.d.ts文件中，比如日志 API：`@ohos.hilog.d.ts`：

```js
declare namespace hilog {

function debug(domain: number, tag: string, format: string, ...args: any[]): void;

function info(domain: number, tag: string, format: string, ...args: any[]): void;

function warn(domain: number, tag: string, format: string, ...args: any[]): void;

function error(domain: number, tag: string, format: string, ...args: any[]): void;

function fatal(domain: number, tag: string, format: string, ...args: any[]): void;

function isLoggable(domain: number, tag: string, level: LogLevel): boolean;

enum LogLevel {

    DEBUG = 3,

    INFO = 4,

    WARN = 5,

    ERROR = 6,
  
    FATAL = 7
}

}

export default hilog;
```

将 `@ohos.hilog.d.ts` 转换为 Kotlin，Kotlin 提供了 [Dukat 工具](https://github.com/Kotlin/dukat)，

> Dukat 工具处于[实验阶段](https://kotlinlang.org/docs/components-stability.html#stability-levels-explained)。它可能随时被删除或更改。

Dukat 目前处于停滞状态，不过，这里还有一个工具 [Karakum](https://github.com/karakum-team/karakum)：

> Converter of TypeScript declaration files to Kotlin declarations.

使用 [Karakum](https://github.com/karakum-team/karakum/blob/master/docs/guides/Basic_usage.md) 转换`@ohos.hilog.d.ts`后的 Kotlin 文件为：

```kotlin
// Generated by Karakum - do not modify it manually!

@file:JsModule("@ohos.hilog")
@file:Suppress(
    "NON_EXTERNAL_DECLARATION_IN_INAPPROPRIATE_FILE",
)
//可以修改
package ohos.hilog

external object hilog {

fun debug(domain: Double, tag: String, format: String, vararg args: Any?): Unit

fun info(domain: Double, tag: String, format: String, vararg args: Any?): Unit

fun warn(domain: Double, tag: String, format: String, vararg args: Any?): Unit

fun error(domain: Double, tag: String, format: String, vararg args: Any?): Unit

fun fatal(domain: Double, tag: String, format: String, vararg args: Any?): Unit

fun isLoggable(domain: Double, tag: String, level: LogLevel): Boolean

sealed interface LogLevel {
companion object {

val DEBUG: LogLevel

val INFO: LogLevel

val WARN: LogLevel

val ERROR: LogLevel

val FATAL: LogLevel
}
}
}

/* export default hilog; */

```

有了 Kotlin 文件后，将它导入 `jsMain` 中，此时就可以访问鸿蒙平台特性。

### 封装鸿蒙API

下面以封装日志为简单示例。

在上一篇[# Kotlin Multiplatform 访问不同平台特性 ]({{site.url}}/2024/11/22/KMP-Expect-Actual)中介绍了，如何利用 Kotlin Multiplatform 中的 `expect`和 `actual`机制访问不同平台特性。

在 `commonaMain`中，`commonMain/kotlin/XLog.kt`：

```kotlin
import kotlin.js.ExperimentalJsExport
import kotlin.js.JsExport

/**
 * 定义日志接口，对Android,iOS,Harmony平台抽象后的通用接口。（这里只是示例）
 */
@OptIn(ExperimentalJsExport::class)
@JsExport
interface XLog {
    fun debug(tag: String, message: String)
}

/**
 * 各个平台实现 XLog 接口
 */
expect val xLog: XLog

/**
 * 依赖该Module，对KMP项目所有内部Module使用，以及导出.js产物给其它是使用
 */
@OptIn(ExperimentalJsExport::class)
@JsExport
object KLog : XLog {
    override fun debug(tag: String, message: String) {
        xLog.debug(tag, message)
    }
}
```

在 Android 平台中，`androidMain/kotlin/XLog.android.kt`：

```kotlin
import android.util.Log

actual val xLog: XLog = object : XLog {
    override fun debug(tag: String, message: String) {
        Log.d(tag, message)
    }
}
```

在 iOS 平台中，`iosMain/kotlin/XLog.ios.kt`：

```kotlin
import platform.Foundation.NSLog

actual val xLog: XLog = object : XLog {
    override fun debug(tag: String, message: String) {
        NSLog(message)
    }
}
```

在 鸿蒙 平台中，`jsMain/kotlin/XLog.js.kt`：

```kotlin
actual val xLog: XLog = object : XLog {
    override fun debug(tag: String, message: String) {
        //TODO：使用转换 `@ohos.hilog.d.ts`后的 Kotlin 文件
    }
}
```

在 `jsMain/kotlin`目录下，新建包`ohos.api`，并把转换 `@ohos.hilog.d.ts`后的 Kotlin 文件`@ohos.hilog.kt` 导入进来，接下来就可以访问鸿蒙平台日志：

```kotlin
import ohos.api.hilog

actual val xLog: XLog = object : XLog {
    override fun debug(tag: String, message: String) {
        hilog.debug(0.0, tag, message)
    }
}
```

现在代码已经有了，需要构建出包含`import hilog from '@ohos.hilog'; `的 .js 产物。

#### 构建产物

在 Moduel 下的 build.gradle.kts 文件中配置：

```kotlin
kotlin {

    targets.configureEach {
        compilations.configureEach {
            compileTaskProvider.get().compilerOptions {
                freeCompilerArgs.add("-Xexpect-actual-classes")
            }
        }
    }
    //省略

    js(IR) {
        moduleName = "kmp-shared"
        compilations.all {
            compileTaskProvider.configure {
                compilerOptions.freeCompilerArgs.add("-Xerror-tolerance-policy=SYNTAX")
            }
            if (this.compilationName == "main") {
                packageJson {
                    this.name = "kmp-shared"
                    this.version = "0.0.1"
                }
            }
        }

        nodejs()
        binaries.executable()//生成可执行的js文件
        generateTypeScriptDefinitions()//生成 TypeScript定义.d.ts
        useEsModeuls()//支持es2015
}
    }

    sourceSets {
        //省略
        val jsMain by getting {
            dependencies {
                implementation(libs.kotlin.stdlib.js)
            }
        }
    }
}
```

在命令执行：`./gradlew :module:assemble`会生成 Android，iOS，Harmony 平台所有产物。也可以单独执行：`./gradlew :module:compileProductionExecutableKotlinJs`生成鸿蒙产物。

产物在根项目`build/js/packages/kmp-shared/kotlin`目录下：

<img src="{{site.url}}/images/posts/2024-12-05-KMP-With-Harmony-API/p3.png" width="100%" height="100%" />

因为使用的是ES6，所以产物是.mjs文件。现在产物有两个地方不符合预期：

1.  预期 .js 文件，实际 .mjs 文件
2.  预期在.js文件中`import hilog from '@ohos.hilog';`，实际在.mjs文件中`import { hilog as hilog} from '@ohos.hilog';`

##### 自定义task更改产物

自定义task主要做两件事：

1.  将.mjs文件修改为.js文件
2.  将`import { hilog as hilog} from '@ohos.hilog';`替换为`import hilog from '@ohos.hilog';`

自定义task示例：

```kotlin
tasks.register("jsProcessOutputs") {
    // 指定文件目录 (默认 Kotlin/JS 输出目录)
    val jsOutputDir = file("${project.rootProject.buildDir.path}/js/packages/kmp-shared/kotlin")
    doLast {
        val dir = file(jsOutputDir)
        if (!dir.exists()) return@doLast
        dir.walkTopDown().filter { it.extension == "mjs" }.forEach { mjsFile ->
            // 读取 .mjs 文件内容
            val content = mjsFile.readText()
            // 替换所有 `import { x as x }` 为 `import x`
            val updatedContent =
                content.replace(Regex("""import\s+{\s*(\w+)\s+as\s+\1\s*}""")) { match ->
                    "import ${match.groupValues[1]}"
                }
            val jsFile = File(mjsFile.parentFile, "${mjsFile.nameWithoutExtension}.js")
            jsFile.writeText(updatedContent)
        }
    }
}

// 将任务绑定到 Kotlin/JS 编译后的产物
tasks.named("compileProductionExecutableKotlinJs") {
    finalizedBy("jsProcessOutputs")
}
```

<img src="{{site.url}}/images/posts/2024-12-05-KMP-With-Harmony-API/p4.png" width="100%" height="100%" />

#### 在鸿蒙中使用

将 .js 和 .d.ts 文件导入鸿蒙项目中，`.d.ts `文件内容为：

```ts
type Nullable<T> = T | null | undefined
export declare interface XLog {
    debug(tag: string, message: string): void;
    readonly __doNotUseOrImplementIt: {
        readonly XLog: unique symbol;
    };
}
export declare const KLog: {
    getInstance(): {
        debug(tag: string, message: string): void;
        readonly __doNotUseOrImplementIt: XLog["__doNotUseOrImplementIt"];
    } & XLog;
};
```

在某个 page 中使用：

```ts
import { KLog } from 'shared/src/main/ets/kmp-shared'

@Entry({ routeName: ROUTE_NAME_KOTLIN_JS_PAGE })
@Component
struct Index {

aboutToAppear(): void {
  KLog.getInstance().debug("Harmony","This is from Kotlin/JS.")
}
}
```

控制台输出：

```shell
12-04 11:25:31.239   46631-46631   A0FFFF/com.exa...ation/Harmony  com.examp...lication  D     This is from Kotlin/JS.
```

## 总结

在 KMP 项目中，封装鸿蒙API的主要流程是：

<img src="{{site.url}}/images/posts/2024-12-05-KMP-With-Harmony-API/p5.png" width="100%" height="100%" />

1.  将鸿蒙 API 的 .d.ts 文件导出
2.  使用 Dukat 或 Karakum将.d.ts 文件转换为.kt文件
3.  在 KMP 项目中导入 .kt 文件
4.  编写适用于 Android，iOS，Harmony 平台的代码
5.  使用 ES Moduel 模块化规范
6.  将产物 .mjs 修改为 .js 文件，并将 .js 文件中的 `import { x as x} from y` 替换为 `import x from y`
7.  在鸿蒙项目中导入 .js 和 .d.ts 文件

在 KMP 项目，封装鸿蒙API或者封装各平台特性的主要目的是：

1.  各平台采用相同技术方案，保证一致性
2.  提供统一接口，屏蔽平台差异
3.  都使用 Kotlin 语言特性和工具支持，统一开发体验
4.  构建牢固可靠的地基，保证稳定性

## 参考文档：

1.[前端模块标准之CommonJS、ES6 Module、AMD、UMD介绍](https://juejin.cn/post/6959360215674257415)

2.[聊聊什么是CommonJs和Es Module及它们的区别 ](https://juejin.cn/post/6938581764432461854)

3.[JavaScript modules](https://kotlinlang.org/docs/js-modules.html)

4.[Use JavaScript code form Kotlin](https://kotlinlang.org/docs/js-interop.html)
