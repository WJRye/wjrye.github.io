---
layout: post
title: Kotlin/Native 给鸿蒙使用（二）
categories: [Kotlin, KMP]
description: 在Kotlin/Native中，利用Kotlin与C语言的互操作性，以及提供的 cinterop工具，不仅能访问鸿蒙平台的Native能力，而且还能直接生成符合 Node-API 规范的 `.so`。
keywords: KMP
mermaid: false
sequence: false
flow: false
mathjax: false
mindmap: false
mindmap2: false
---

![p0.png]({{site.url}}/images/posts/2025-03-12-Kotlin-Native-For-Harmony/p0.png)

## 前言
在上一篇[# Kotlin/Native 给鸿蒙使用（一）]({{site.url}}/2025/02/27/Kotlin-Native-For-Harmony)中，介绍了利用 Kotlin/Native 对 `linux_arm64` 提供的能力支持鸿蒙。通过这些能力，不仅可以访问系统底层能力，比如访问文件，线程，网络等，而且还能保证良好的性能。这让跨平台更容易和更友好。

因为 KMP 主要聚焦在 Android 和 iOS，所以集成了 Android 和 iOS 平台的很多原生能力，`~/.konan/kotlin-native-prebuilt-macos-aarch64-2.1.10-RC2/klib/platform` ：
```
.

├── android_arm64
│   ├── org.jetbrains.kotlin.native.platform.android
...省略
│   ├── org.jetbrains.kotlin.native.platform.egl # OpenGL EGL
│   ├── org.jetbrains.kotlin.native.platform.gles # OpenGL ES
...省略

├── ios_arm64
│   ├── org.jetbrains.kotlin.native.platform.ARKit # AR
│   ├── org.jetbrains.kotlin.native.platform.AVFAudio # 音频
│   ├── org.jetbrains.kotlin.native.platform.AVFoundation # 多媒体
...省略
```
那么，KMP 能不能集成鸿蒙平台的原生能力，如访问鸿蒙日志，OpenGL能力等。在 Kotlin/Native 中是可以的，通过利用 Kotlin 与 C 语言的互操作性，以及提供的 cinterop 工具，不仅能访问鸿蒙平台的 Native 能力，而且还能直接生成符合 Node-API 规范的 `.so`。

##  cinterop 工具

在 Kotlin/JS 中有[Dukat 工具](https://github.com/Kotlin/dukat)和 [Karakum 工具](https://github.com/karakum-team/karakum)，将`.d.ts`转换为 Kotlin。在 Kotlin/Native 中有 [cinterop 工具](https://kotlinlang.org/docs/native-definition-file.html)，将 `C 库` 转换为 Kotlin。

下面以在 macos_arm64 生成 curl 库的 Kotlin 绑定为例。（[Kotlin/Native 项目](https://github.com/Kotlin/kmp-native-wizard)模版。）

定义一个文件 `curl.def`（[查看def文件定义规则](https://kotlinlang.org/docs/native-definition-file.html)）：

```
headers = curl/curl.h # 指定要导入的 C/C++ 头文件
headerFilter = curl/* # 防止引入不相关的头文件
compilerOpts.osx = -I/opt/homebrew/include # 指定 macOS 平台编译器的选项
linkerOpts.osx = -L/opt/homebrew/lib -lcurl # 指定 macOS 平台链接器的选项，在`/opt/homebrew/lib` 目录下找 curl
```
把该文件放在 src 目录下：`src/nativeInterop/cinterop/curl.def`，在 build.gradle.kts 中配置：

```kotlin
kotlin {
    applyDefaultHierarchyTemplate()
    macosArm64().apply {
        this.compilations["main"].cinterops {
            val curl by creating {
            defFile(project.file("./src/nativeInterop/cinterop/curl.def").path)
                packageName("curl")
            }
        }
        this.binaries {
            executable {
                entryPoint = "main"
            }
            sharedLib {
                baseName = "kn"
            }
        }
    }
}
```
同步项目。如果在项目中使用 curl 库找不索引，可以看下`项目目录/.kotlin/metadata/kotlinCInteropLibraries`文件夹下有没有 curl 相关 klib，没有就重启动一下 AndroidStudio。在 AndroidStudio 中，可以通过 klib 查看 curl 库生成的 Kotlin 代码。

在 Main.kt 中使用 curl 库：
```kotlin
import curl.CURLOPT_URL
import curl.curl_easy_cleanup
import curl.curl_easy_init
import curl.curl_easy_perform
import curl.curl_easy_setopt
import kotlin.experimental.ExperimentalNativeApi
import kotlinx.cinterop.*


@OptIn(ExperimentalForeignApi::class)
fun main() {
    memScoped {
        val curl = curl_easy_init()
        if (curl != null) {
            val url = "https://www.bilibili.com"
            curl_easy_setopt(curl, CURLOPT_URL, url)
            curl_easy_perform(curl)
            curl_easy_cleanup(curl)
        } else {
            println("Failed to initialize curl")
        }
    }
}
```
在命令行执行` ./gradlew linkMacosArm64 --info`，在 `build/bin/macosArm64` 目录会生成产物：
```
.
├── debugExecutable
│   ├── KotlinNativeTemplate.kexe
│   └── KotlinNativeTemplate.kexe.dSYM
├── debugShared
│   ├── libkn.dylib
│   ├── libkn.dylib.dSYM
│   └── libkn_api.h
├── debugTest
│   ├── test.kexe
│   └── test.kexe.dSYM
├── releaseExecutable
│   ├── KotlinNativeTemplate.kexe
│   └── KotlinNativeTemplate.kexe.dSYM
└── releaseShared
    ├── libkn.dylib
    ├── libkn.dylib.dSYM
    └── libkn_api.h
```
查看是否可运行，可以执行一下 `KotlinNativeTemplate.kexe`。或在命令行执行`./gradlew runDebugExecutableMacosArm64` 或`./gradlew runReleaseExecutableMacosArm64`。

## Kotlin 与 C 互操作性

把 C 库 libcurl 生成 Kotlin 相关的 klib，依赖于 Kotlin 与 C 的互操作性。

### 类型映射

在 Kotlin 代码中调用 C，需要类型映射。下面简单介绍一下，具体可查看[文档](https://kotlinlang.org/docs/native-c-interop.html)。

基础类型：有符号、无符号整型和浮点类型，被映射到具有相同位宽的 Kotlin 对应类型。如：

| C | Kotlin  | 说明 | 例子  |
| --- | --- | --- | --- |
| int | Int | 32 位整数 | `int i = 42;` → `val i: Int = 42` |
| unsigned int | UInt | 无符号32 位整数 | `unsigned int ui = 1000;` → `val ui: UInt = 1000u` |

其它 char, short, long, float, double, bool 类型等类似。

指针和数组：被映射到 `CPointer<T>?`。如：

| C | Kotlin  | 说明 | 例子  |
| --- | --- | --- | --- |
| T* | `CPointer<T>` | C 指针映射为 `CPointer<T>`，具体类型由 `T` 决定 | `int* arr = ...;` → `val arr: CPointer<IntVar> = ...` |
| char* | String | C 字符串转换为 Kotlin `String`（需内存管理） | `char* s = "Hello";` → `val str: String = s.toKString()` |
| T[] 数组 | `CPointer<TVar>` | C 数组通过指针访问 | `int arr[10];` → `val arrPtr: CPointer<IntVar> = ...`  |
| 函数指针 | `CFunction<ArgsType>`  | 函数指针通过 `CFunction` 类型表示，需用 `staticCFunction` 包装 Kotlin 函数 | `int (*callback)(int);` → `val callback: CFunction<(Int) -> Int> = ...`|

枚举：被映射到 Kotlin 枚举或整型值。如：

| C | Kotlin  | 说明 |
| --- | --- | --- |
enum | Kotlin `enum` 或 `Int` | C 枚举默认映射为 `Int`，可通过配置生成 Kotlin 枚举类 |

结构体和联合体：被映射到通过点符号（即 `someStructInstance.field1`）访问字段的类型。

typedef：被表示为 `typealias`（类型别名）。

空指针：被表示为 `null`。

### 分配和释放内存

可以使用`NativePlacement`手动分配和释放内存：

```kotlin
import kotlinx.cinterop.*

@OptIn(kotlinx.cinterop.ExperimentalForeignApi::class)
fun main() {
    val size: Long = 0
    val buffer = nativeHeap.allocArray<ByteVar>(size)
    nativeHeap.free(buffer)
}
```
分配内存，往往忘记释放内存。可以使用`memScoped { }`自动释放内存：

```kotlin
import kotlinx.cinterop.*
import platform.posix.*

@OptIn(ExperimentalForeignApi::class)
val fileSize = memScoped {
    val statBuf = alloc<stat>()
    val error = stat("/", statBuf.ptr)
    statBuf.st_size
}
```

## 集成鸿蒙 hilog

在鸿蒙 Native 使用 `hilog`，动态库是：`libhilog_ndk.z.so`，头文件是`hilog/log.h`。定义文件 `src/nativeInterop/cinterop/hilog.def`：

```
headers = hilog/log.h bits/alltypes.h
compilerOpts.linux = -I/Applications/DevEco-Studio.app/Contents/sdk/default/openharmony/native/sysroot/usr/include -I/Applications/DevEco-Studio.app/Contents/sdk/default/openharmony/native/sysroot/usr/include/aarch64-linux-ohos
linkerOpts.linux = -L/Applications/DevEco-Studio.app/Contents/sdk/default/openharmony/native/sysroot/usr/lib -L/Applications/DevEco-Studio.app/Contents/sdk/default/openharmony/native/sysroot/usr/lib/aarch64-linux-ohos -lhilog_ndk.z
```
在 build.gradle.kts 中配置：

```kotlin
//构建二进制文件
kotlin {
    applyDefaultHierarchyTemplate()
    //生成linux平台下的 .so
    val linuxTargets = listOf(linuxArm64())
    linuxTargets.forEach {
        it.binaries {
            executable {
                entryPoint = "main"
                this.compilation.compileTaskProvider.configure {
                    this.compilerOptions.freeCompilerArgs.addAll(
                        listOf(
                            "-Xoverride-konan-properties=linkerGccFlags=-lgcc",
                            "-linker-options", "-as-needed",
                        )
                    )
                }
            }
            sharedLib {
                //生成 libkn.so
                baseName = "kn"
            }
        }
        it.compilations["main"].cinterops {
            val hilog by creating {
                defFile(project.file("./src/nativeInterop/cinterop/hilog.def").path)
                packageName("hilog")
            }
        }
    }
}
```
封装 hilog：
```kotlin
import hilog.LOG_APP
import hilog.LOG_DEBUG
import hilog.LOG_ERROR
import hilog.LogLevel
import kotlinx.cinterop.ExperimentalForeignApi

object HILog {
    private const val DOMAIN = 0u

    @OptIn(ExperimentalForeignApi::class)
    fun d(tag: String, msg: String) {
        hilog.OH_LOG_Print(LOG_APP, LOG_DEBUG, DOMAIN, tag, "%{public}s", msg)
    }

    @OptIn(ExperimentalForeignApi::class)
    fun e(tag: String, msg: String) {
        hilog.OH_LOG_Print(LOG_APP, LOG_ERROR, DOMAIN, tag, "%{public}s", msg)
    }

    @OptIn(ExperimentalForeignApi::class)
    fun isLoggable(tag: String, level: LogLevel) {
        hilog.OH_LOG_IsLoggable(DOMAIN, tag, level)
    }
    //...省略
}
```
在命令行执行：`./gradlew linkLinuxArm64` 生成产物 `libkn.so` 和 `libkn_api.h`，查看`libkn.so`依赖：

```
./aarch64-linux-musl-readelf -d libkn.so

Dynamic section at offset 0x95718 contains 27 entries:

  Tag        Type                         Name/Value

 0x0000000000000001 (NEEDED)             Shared library: [libpthread.so.0]

 0x0000000000000001 (NEEDED)             Shared library: [libhilog_ndk.z.so]

 0x0000000000000001 (NEEDED)             Shared library: [libdl.so.2]
 
 //...省略
```
依赖 hilog 库 `libhilog_ndk.z.so`。
## 集成鸿蒙 napi

在 Kotlin/Native 集成的 Android 平台原生能力中，通过 `org.jetbrains.kotlin.native.platform.android` klib 就能访问 JNI，然后直接生成符合 JNI 规范的 `.so`。同样，通过集成鸿蒙 napi，就能直接生成符合 Node-API 规范的 `.so`。

在鸿蒙 Native 使用 `napi`，动态库是：`libace_napi.z.so`，头文件是`
napi/native_api.h napi/common.h
`。定义文件 `src/nativeInterop/cinterop/napi.def`：

```
headers = napi/native_api.h napi/common.h
compilerOpts.linux = -I/Applications/DevEco-Studio.app/Contents/sdk/default/openharmony/native/sysroot/usr/include -I/Applications/DevEco-Studio.app/Contents/sdk/default/openharmony/native/sysroot/usr/include/aarch64-linux-ohos
linkerOpts.linux = -L/Applications/DevEco-Studio.app/Contents/sdk/default/openharmony/native/sysroot/usr/lib -L/Applications/DevEco-Studio.app/Contents/sdk/default/openharmony/native/sysroot/usr/lib/aarch64-linux-ohos -lace_napi.z
```
在 build.gradle.kts 中配置：
```kotlin
//构建二进制文件
kotlin {
    applyDefaultHierarchyTemplate()
    //生成linux平台下的 .so
    val linuxTargets = listOf(linuxArm64())
    linuxTargets.forEach {
        //...省略
        it.compilations["main"].cinterops {
            val hilog by creating {
                defFile(project.file("./src/nativeInterop/cinterop/hilog.def").path)
                packageName("hilog")
            }
            val napi by creating {
                defFile(project.file("./src/nativeInterop/cinterop/napi.def").path)
                packageName("napi")
            }
        }
    }
}
```
现在已经生成了 `libace_napi.z` 库的 Kotlin 绑定，就可以在 Kotlin 中使用 napi。

### 用 Kotlin 实现 napi_init.cpp
在鸿蒙项目中创建 Native C++ 模块 hn（任意命名），napi_init.cpp 为：

```c
#include "napi/native_api.h"

static napi_value Add(napi_env env, napi_callback_info info)
{
    size_t argc = 2;
    napi_value args[2] = {nullptr};

    napi_get_cb_info(env, info, &argc, args , nullptr, nullptr);

    napi_valuetype valuetype0;
    napi_typeof(env, args[0], &valuetype0);

    napi_valuetype valuetype1;
    napi_typeof(env, args[1], &valuetype1);

    double value0;
    napi_get_value_double(env, args[0], &value0);

    double value1;
    napi_get_value_double(env, args[1], &value1);

    napi_value sum;
    napi_create_double(env, value0 + value1, &sum);

    return sum;

}

EXTERN_C_START
static napi_value Init(napi_env env, napi_value exports)
{
    napi_property_descriptor desc[] = {
        { "add", nullptr, Add, nullptr, nullptr, nullptr, napi_default, nullptr }
    };
    napi_define_properties(env, exports, sizeof(desc) / sizeof(desc[0]), desc);
    return exports;
}
EXTERN_C_END

static napi_module demoModule = {
    .nm_version = 1,
    .nm_flags = 0,
    .nm_filename = nullptr,
    .nm_register_func = Init,
    .nm_modname = "hn",
    .nm_priv = ((void*)0),
    .reserved = { 0 },
};

extern "C" __attribute__((constructor)) void RegisterHnModule(void)
{
    napi_module_register(&demoModule);
}
```
#### 用 Kotlin 实现 napi_module 模块注册

![p0.png]({{site.url}}/images/posts/2025-03-12-Kotlin-Native-For-Harmony/p1.png)

新建 `src/linuxArm64Main/kotlin/NAPIInit.kt`：

```kotlin
@OptIn(ExperimentalForeignApi::class)
val demoModule: napi_module = memScoped {
    val module = alloc<napi_module>()
    module.nm_version = 1 // nm版本号，默认值为1
    module.nm_flags = 0u // nm标识符
    module.nm_filename = null // 文件名，暂不关注，使用默认值即可
    val func: napi.napi_addon_register_func = staticCFunction { p1: napi_env?, p2: napi_value? ->
        Init(p1!!, p2!!)
    }
    module.nm_register_func = func  // 指定nm的入口函数，这里是 Init
    module.nm_modname = "hn".cstr.getPointer(this) // 指定ArkTS页面导入的模块名，这里是 hn
    module.nm_priv = null // 暂不关注，使用默认即可
    for (i in 0 until 4) {
        module.reserved[i] = null // 暂不关注，使用默认值即可
    }
    module
}

@CName("KNInit")
@OptIn(ExperimentalForeignApi::class, ExperimentalNativeApi::class)
fun register() {
    //napi native模块注册
    napi_module_register(demoModule.readValue())
}
```
通过 `@CName("KNInit")` 注解，让鸿蒙能访问 KNInit，也就能进行模块注册。

在实现的过程中，在 `项目目录/.kotlin/metadata/kotlinCInteropLibraries`文件夹找到 napi 相关 klib，通过 klib 查看 napi 库生成的 Kotlin 代码，然后对比 napi_init.cpp，并根据 Kotlin 与 C 互操作性，就能编写出 Kotlin 实现。

#### 用 Kotlin 实现 Init 初始化

```kotlin
@OptIn(ExperimentalNativeApi::class, ExperimentalForeignApi::class)
@CName("Init")
fun Init(env: napi_env, exports: napi_value): napi_value = memScoped {
    // 定义属性描述符
    val desc = allocArray<napi_property_descriptor>(1).apply {
        this[0].utf8name = "add".cstr.getPointer(memScope)
        val func: napi.napi_callback = staticCFunction { p1: napi_env?, p2: napi_callback_info? ->
            Add(p1!!, p2!!)
        }
        this[0].method = func
        this[0].attributes = napi_default
        this[0].data = null
    }
    // 注册属性到 exports
    napi_define_properties(env, exports, 1u, desc)
    return exports
}
```
Init 函数 在 `napi_init.cpp` 中是 `EXTERN_C_START ...Init... EXTERN_C_END`，所以也需要添加`@CName("Init")`注解。Init 的作用是初始化并进行接口映射，比如接口 Add &rarr; add。

#### 用 Kotlin 实现 Add 功能

```kotlin
@OptIn(ExperimentalForeignApi::class)
fun Add(env: napi_env, info: napi_callback_info): napi_value = memScoped {
    HILog.e("Native", "Add function called!")
    val argc = alloc<size_tVar>().apply { value = 2uL }
    val args = allocArray<napi_valueVar>(2).apply {
        this[0] = null
        this[1] = null
    }
    // 获取参数
    napi_get_cb_info(env, info, argc.ptr, args, null, null)
    // 检查类型
    val valuetype0 = alloc<napi_valuetype.Var>()
    val valuetype1 = alloc<napi_valuetype.Var>()
    napi_typeof(env, args[0], valuetype0.ptr)
    napi_typeof(env, args[1], valuetype1.ptr)
    // 转 double
    val value0 = alloc<DoubleVar>()
    val value1 = alloc<DoubleVar>()
    napi_get_value_double(env, args[0], value0.ptr)
    napi_get_value_double(env, args[1], value1.ptr)
    HILog.e("Native", "${value0.value}+${value1.value}=${value0.value + value1.value}")
    // 计算和
    val sum = alloc<napi_valueVar>()
    napi_create_double(env, value0.value + value1.value, sum.ptr)
    return sum.value!!
}
```
Add 功能参照`napi_init.cpp`中的 Add，并根据 Node-API 规范编写。

在 Add 函数里面使用封装的 hilog: `HILog`。

### 构建产物

现在已经在 Kotlin/Native 中集成了 hilog 和 napi，在命令行执行`./graldew linkLinuxArm64` 生成产物 `libkn.so `和 `libkn_api.h`。

查看头文件`libkn_api.h`：

```c
//...省略

//声明外部函数
extern void* Init(void* env, void* exports);
extern void KNInit();

typedef struct {
  //...省略  
  struct {
    struct {
      struct {
        libkn_KType* (*_type)(void);
        libkn_kref_HILog (*_instance)();
        void (*d)(libkn_kref_HILog thiz, const char* tag, const char* msg);
        void (*e)(libkn_kref_HILog thiz, const char* tag, const char* msg);
        void (*isLoggable)(libkn_kref_HILog thiz, const char* tag, libkn_KUInt level);
      } HILog;
      const char* (*harmonyReadFile)(const char* filePath);
      void (*harmonyWriteFile)(const char* filePath, const char* content);
      void* (*get_demoModule)();
      void* (*Add)(void* env, void* info);
      void* (*Init_)(void* env, void* exports);
      void (*register_)();
      const char* (*readFile_)(const char* filePath);
      void (*writeFile_)(const char* filePath, const char* content);
      void (*main)();
    } root;
  } kotlin;
} libkn_ExportedSymbols;
//声明外部函数：入口
extern libkn_ExportedSymbols* libkn_symbols(void);
```
查看动态库 `libkn.so` 依赖：

```
./aarch64-linux-musl-readelf -d libkn.so

Dynamic section at offset 0x95718 contains 27 entries:

  Tag        Type                         Name/Value

 0x0000000000000001 (NEEDED)             Shared library: [libpthread.so.0]

 0x0000000000000001 (NEEDED)             Shared library: [libhilog_ndk.z.so]

 0x0000000000000001 (NEEDED)             Shared library: [libace_napi.z.so]

 0x0000000000000001 (NEEDED)             Shared library: [libdl.so.2]
```
依赖 hilog 库 `libhilog_ndk.z.so` 和 napi 库`libace_napi.z.so`。

## 鸿蒙接入 Kotlin/Native So


![p0.png]({{site.url}}/images/posts/2025-03-12-Kotlin-Native-For-Harmony/p2.png)

将 libkn.so 放在 khn/libs/arm64-v8a/ 目录下，将 libkn_api.h 放在 khn/libs/arm64-v8a/include 目录下，并修改配置文件`CMakeLists.txt`，同[# Kotlin/Native 给鸿蒙使用（一）](https://juejin.cn/post/7475719627608965147#heading-7)。

此时，在 `napi_init.cpp` 中，删除其它代码，只保留 `RegisterHnModule` ：

```c
#include "libkn_api.h"


extern "C" __attribute__((constructor)) void RegisterHnModule(void)
{
    KNInit();
}
```
接下来在 src/main/cpp/types/libhn/Index.d.ts 定义方法导出：

```ts
export const add: (a: number, b: number) => number;
```
将 `add` 给其它模块使用，hn/Index.ets：

```ts
export { add } from 'libhn.so';
```
鸿蒙项目中任意模块依赖 hn 模块：

```json
"dependencies": {
  "khn": "file:../khn",
},
```
在模块中使用（测试代码）：

```ts
import { hilog } from '@kit.PerformanceAnalysisKit';
import { add } from 'hn/Index'

@Entry
@Component
struct Index {
  @State message: string = 'Kotlin/Native';

  build() {
    Row() {
      Column() {
        Text(this.message)
          .fontSize(50)
          .fontWeight(FontWeight.Bold)
          .onClick(() => {
            hilog.error(0x0000, 'Native', 'Test NAPI 2 + 3 = %{public}d', add(2, 3));
          })
      }
      .width('100%')
    }
    .height('100%')
  }
}
```
点击 Text Kotlin/Native，在控制台中查看：

![p0.png]({{site.url}}/images/posts/2025-03-12-Kotlin-Native-For-Harmony/p3.png)

可以看到 Kotlin/Native 封装的 hilog: `HILog`，以及封装 napi 生成符合 Node-API 规范的 `libkn.so`在 Harmony 平台成功运行。

## 总结

在 Kotlin/Native 中，除了使用 `linux_arm64` 基础能力支持鸿蒙外，还可利用 Kotlin 与 C 语言的互操作性，以及提供的 cinterop 工具，把鸿蒙平台的原生能力集成进来，如日志，OpenGL等，这进一步提升了 Kotlin/Native 的能力：设计合理的架构，全面覆盖 Android，iOS, Harmony 平台。

