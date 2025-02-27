---
layout: post
title: Kotlin/Native 给鸿蒙使用（一）
categories: [Kotlin, KMP]
description: 通过 Kotlin/Native 直接访问系统底层能力文件，网络，多媒体，多线程等功能，可以突破 Kotlin/Android, Kotlin/iOS, Kotlin/JS 上层的限制，达到真正的一个API在 Android, iOS, Harmony 平台使用，而且还能保证良好的性能。
keywords: Kotlin, Native
mermaid: false
sequence: false
flow: false
mathjax: false
mindmap: false
mindmap2: false
---

![截屏2025-02-26 12.28.50.png]({{site.url}}/images/posts/2025-02-27-Kotlin-Native-For-Harmony/p0.jpg)

## 前言

之前在[# Kotlin Multiplatform 跨平台支持鸿蒙]({{site.url}}/2024/06/12/Kotlin-Multiplatform-Support-Harmony/)和[# Kotlin Multiplatform 封装鸿蒙API]({{site.url}}/2024/12/05/KMP-With-Harmony-API/)介绍了利用 Kotlin/JS 能力支持鸿蒙，通过把逻辑放在 `commonMain` 中，实现Android：`androidMain` 、iOS：`iosMain` 、Harmony：`jsMain`共享逻辑。在这个过程中，遇到一些限制，比如访问文件，数据库，多线程，多媒体，网络功能等，不能编写出一个共用 API 让 Android, iOS , Harmony 平台都调用，只有定义接口让各平台实现才能适配共享逻辑。为了突破限制，利用 Kotlin/Native 从底层访问系统能力以达到更丝滑，更简单和更容易的方式支持跨平台共享逻辑。

## 鸿蒙 NDK

要进行 Native 开发，Andrid 有 NDK，鸿蒙也有 [NDK](https://developer.huawei.com/consumer/cn/doc/harmonyos-guides-V5/ndk-development-overview-V5)。
> 鸿蒙 NDK 前置知识
>
>-   Linux C语言编程知识：内核、libc基础库基于POSIX等标准扩展而来，掌握基本的Linux C编程知识能够更好的帮助理解HarmonyOS NDK开发。
>    
>-   CMake使用知识：CMake是HarmonyOS默认支持的构建系统。请先通过[CMake官方文档](https://cmake.org/cmake/help/v3.16/guide/tutorial/)了解基础用法。
>
>-   Node Addons开发知识：ArkTS采用Node-API作为跨语言调用接口，熟悉基本的[Node Addons开发模式](https://nodejs.org/api/addons.html)，可以更好理解NDK中Node-API的使用。
>
>-   Clang/LLVM编译器使用知识：具备一定的Clang/LLVM编译器基础知识，能够帮助开发者编译出更优的Native动态库。

现在需要：复用已有C或C++库，也就是在鸿蒙项目中依赖三方库（`*.so`）-Kotlin/Native 产物。主要了解一下鸿蒙 Native 通用能力、目标架构 和 Node-API。

**通用能力**：鸿蒙基于 linux 内核，可以使用 libc，libc++，zlib，OpenGL，以及**POSIX 标准**等，所以完全支持文件操作、线程、网络等能力。

**[目标架构](https://developer.huawei.com/consumer/cn/doc/harmonyos-faqs-V5/faqs-compiling-and-building-92-V5)**：linux_arm64。鸿蒙手机、平板等移动设备使用 arm64 架构。

**[Node-API](https://developer.huawei.com/consumer/cn/doc/harmonyos-guides-V5/using-napi-interaction-with-cpp-V5)**：Java/Kotlin 代码与 Native（C/C++）代码进行交互需要 JNI，ArkTS/JS 代码与 Native（C/C++）代码需要 Node-API。

鸿蒙 NDK 和 Android NDK 的角色相似，Node-API 的角色和 JNI 的角色相似。

### 毕昇编译器简介

> 毕昇编译器是基于LLVM开源软件开发的一款用于C/C++等语言的native编译器，能将C/C++代码工程编译链接成可以在设备上运行的二进制。在无需改动用户代码的条件下，相比业界主流的开源LLVM或GCC编译器，毕昇编译器能提供更强大的优化能力，使编译链接出来的二进制的运行时长更短、指令数更少，帮助提升应用在设备上的运行流畅度。

如果在鸿蒙中依赖三方库（`*.so`），那么三方库最好使用毕昇编译器编译。鸿蒙 llvm 路径：`/Applications/DevEco-Studio.app/Contents/sdk/default/openharmony/native/llvm`。

交叉编译时指定 target 和 sysroot：
-   `--target=aarch64-linux-ohos`参数，通知编译器生成相应架构下符合HarmonyOS ABI的二进制文件：linux_arm64。
-   `--sysroot=/Applications/DevEco-Studio.app/Contents/sdk/default/openharmony/native/sysroot` 参数，告知编译器HarmonyOS系统头文件的所在位置。

## Kotlin/Native

### 对 Linux 提供的基础能力

Kotlin/Native 对 Linux 提供的基础能力可以在` ~/.konan/kotlin-native-prebuilt-macos-aarch64-2.1.10-RC2/klib` 文件目录中找到：
```
.

├── cache

├── common # 标准库

├── commonized

└── platform # 平台库
```
`common` 和 `platform` 就是 Kotlin/Native 对 Linux 平台提供的基础能力，在 platform 中支持 `linux_arm32_hfp`，`linux_arm64`，`linux_x64`。

下面看下 linux_arm64 目录：
```
.

├── org.jetbrains.kotlin.native.platform.builtin

├── org.jetbrains.kotlin.native.platform.iconv

├── org.jetbrains.kotlin.native.platform.linux

├── org.jetbrains.kotlin.native.platform.posix

└── org.jetbrains.kotlin.native.platform.zlib
```

| 目录 |  作用|
| --- | --- |
| `org.jetbrains.kotlin.native.platform.builtin` | C 标准库：内存、数学、字符串处理 |
| `org.jetbrains.kotlin.native.platform.iconv` | 字符编码转换 |
| `org.jetbrains.kotlin.native.platform.linux` |  Linux 专用接口：文件监听、高性能 I/O|
| `org.jetbrains.kotlin.native.platform.posix` | POSIX 标准接口：文件、线程、网络通信 |
| `org.jetbrains.kotlin.native.platform.zlib` | 数据压缩与解压 |

每个目录其实都是一个 klib（类似于 Java 的 `.jar`），通过 klib 就能访问这些能力。

`org.jetbrains.kotlin.native.platform.posix`提供了 `platform.posix.*` API，通过 `import platform.posix.*` 就可以访问文件、线程、网络等能力。

### 通过 @CName 指定导出符号

下载[Kotlin/Native 项目](https://github.com/Kotlin/kmp-native-wizard)模版，将 Kotlin 版本更新到最新（避免 bug）。

首先通过 `org.jetbrains.kotlin.native.platform.posix` 写一个简单的文件读取 `FileUtil.kt`：

```kotlin
import kotlinx.cinterop.ExperimentalForeignApi
import platform.posix.EOF
import platform.posix.fclose
import platform.posix.fgetc
import platform.posix.fopen
import platform.posix.fputs

@OptIn(ExperimentalForeignApi::class)
fun writeFile(filePath: String, content: String) {
    val file = fopen(filePath, "w") ?: throw Exception("File cannot be opened")
    fputs(content, file)
    fclose(file)
}

@OptIn(ExperimentalForeignApi::class)
fun readFile(filePath: String): String {
    val file = fopen(filePath, "r") ?: throw Exception("File cannot be opened")
    val content = mutableListOf<Char>()
    var c: Int
    while (true) {
        c = fgetc(file)
        if (c == EOF) break
        content.add(c.toChar())
    }
    fclose(file)
    return content.joinToString("")
}
```
FileUtil.kt 放在 `nativeMain` 中，可以给 Android, iOS, Harmony 使用。

在 `linuxArm64Main`中给 Harmony 使用，新建立一个 `HarmonyFileUtil.kt`：
```kotlin
import kotlin.experimental.ExperimentalNativeApi

@OptIn(ExperimentalNativeApi::class)
@CName("writeFile")
fun harmonyWriteFile(filePath: String, content: String) {
    //调用 FileUtil.kt 的 writeFile
    return writeFile(filePath, content)
}

@OptIn(ExperimentalNativeApi::class)
@CName("readFile")
fun harmonyReadFile(filePath: String): String {
    //调用 FileUtil.kt 的 readFile
    return readFile(filePath)
}
```
`@CName`注解的作用是： 使顶层函数可从 C/C++ 代码中访问。

`@CName(externName="writeFile")`指定导出的符号为：`writeFile`，   `@CName(externName="readFile")`指定导出的符号为：`readFile`。

这里展开稍微简单说下 Android。

 通过上一篇[# Kotlin/Native 给 Android 使用]({{site.url}}/2025/02/19/Kotlin-Native-For-Android/)介绍，在 `androidNativeArm64Main` 给 Android 使用，新建立一个 `AndroidFileUtil.kt`：
```kotlin
import kotlinx.cinterop.CPointer
import kotlinx.cinterop.ExperimentalForeignApi
import kotlinx.cinterop.cstr
import kotlinx.cinterop.invoke
import kotlinx.cinterop.memScoped
import kotlinx.cinterop.pointed
import kotlinx.cinterop.toKString
import platform.android.JNIEnvVar
import platform.android.jobject
import platform.android.jstring
import kotlin.experimental.ExperimentalNativeApi


@OptIn(ExperimentalForeignApi::class, ExperimentalNativeApi::class)
@CName(externName = "Java_com_wj_mylibrary_NativeLib_readFile")
fun androidReadFile(env: CPointer<JNIEnvVar>, thiz: jobject, filePath: jstring): jstring {
    memScoped {
        val newFilePathJString = env.pointed.pointed!!.GetStringChars!!.invoke(env, filePath, null)
        val newFilePathKString = newFilePathJString?.toKString() ?: ""
        env.pointed.pointed!!.ReleaseStringChars!!.invoke(env, filePath, newFilePathJString)
        //调用 FileUtil.kt 的 readFile
        val result = readFile(newFilePathKString)
        val content = result.cstr.ptr
        return env.pointed.pointed!!.NewStringUTF!!.invoke(
            env, content
        )!!
    }
}

@OptIn(ExperimentalForeignApi::class, ExperimentalNativeApi::class)
@CName(externName = "Java_com_wj_mylibrary_NativeLib_writeFile")
fun androidWriteFile(env: CPointer<JNIEnvVar>, thiz: jobject, filePath: jstring, content: jstring) {
    memScoped {
        val newFilePathJString = env.pointed.pointed!!.GetStringChars!!.invoke(env, filePath, null)
        val newFilePathKString = newFilePathJString?.toKString() ?: ""
        env.pointed.pointed!!.ReleaseStringChars!!.invoke(env, filePath, newFilePathJString)

        val newContentJString = env.pointed.pointed!!.GetStringChars!!.invoke(env, content, null)
        val newContentKString = newContentJString?.toKString() ?: ""
        env.pointed.pointed!!.ReleaseStringChars!!.invoke(env, content, newContentJString)
        //调用 FileUtil.kt 的 writeFile
        writeFile(newFilePathKString, newContentKString)
    }
}
```
此时， Android 能直接生成符合 JNI 规范的 `.so`。通过符号`Java_com_wj_mylibrary_NativeLib_writeFile` 访问 `writeFile`，通过 `Java_com_wj_mylibrary_NativeLib_readFile` 符号访问 `readFile`。

回到鸿蒙，鸿蒙是否能直接生成符合 Node-API 规范的 `.so`？利用 Kotlin/Native 的 `C Interop` 和 `@CName` 是可以做到的，这里不做介绍。


### 编译目标架构

在 build.gradle.kts 中配置：

``` kotlin
kotlin {

    //生成linux平台下的 .so
    val linuxTargets = listOf(linuxArm64())
    linuxTargets.forEach {
        it.binaries {
            executable()
            sharedLib {
                //生成 libkn.so
                baseName = "kn"
            }
        }
    }
}
```
给鸿蒙使用，指定 `linuxArm64()`即可。

在命令行执行：`./gradlew linkReleaseSharedLinuxArm64 --info`，`build/bin/linuxArm64/`目录下会生成动态库`libkn.so`和头文件`libkn_api.h`：
```
.
└── releaseShared
    ├── libkn.so
    └── libkn_api.h
```
libkn_api.h 头文件中有：

```c
extern "C" {
//...省略

//声明外部函数
extern const char* readFile(const char* filePath);
extern void writeFile(const char* filePath, const char* content);

//...省略

typedef struct {
   //...省略
   //kotlin与c类型映射
  libkn_KBoolean (*IsInstance)(libkn_KNativePtr ref, const libkn_KType* type);
  //...省略
  /* User functions. */
  struct {
    struct {
      const char* (*harmonyReadFile)(const char* filePath);
      void (*harmonyWriteFile)(const char* filePath, const char* content);
      const char* (*readFile_)(const char* filePath);
      void (*writeFile_)(const char* filePath, const char* content);
      void (*main)();
    } root;
  } kotlin;
} libkn_ExportedSymbols;
////声明外部函数：入口
extern libkn_ExportedSymbols* libkn_symbols(void);

}
```
头文件中，有 kotlin 与 c 的类型映射，比如 c: `bool` &rarr;  kotlin: `libkn_KBoolean`，c: `long long` &rarr;  kotlin: `libnative_KLong`。类型映射关系参看文档[# Kotlin/Native as a dynamic library – tutorial](https://kotlinlang.org/docs/native-dynamic-libraries.html#service-runtime-functions)。

因为添加了 @CName 注解，所以可以直接访问 `writeFile` 和 `readFile`。如果不添加 @CName 注解，可以通过`extern libkn_ExportedSymbols* libkn_symbols(void);`：

```c
libkn_ExportedSymbols* lib = libkn_symbols();

lib->kotlin.root.harmonyWriteFile();

lib->kotlin.root.harmonyReadFile();
```
访问 `harmonyWriteFile` 和 `harmonyReadFile`，达到相同目的。

查看 libkn.so 信息：

```
file libkn.so

ELF 64-bit LSB shared object, ARM aarch64, version 1 (SYSV), dynamically linked, BuildID[xxHash]=de143ecb6b255574, not stripped
```
信息表明：libkn.so 是 ARM 64 位架构编译的动态共享库，使用 ELF 格式，遵循System V ABI。库里包含了完整的符号表（未剥离），有唯一的构建 ID，可以通过动态链接加载到其他程序中。

查看 libkn.so 符号：

```
./aarch64-linux-musl-nm -g libkn.so

...省略
0000000000075048 T readFile
...省略
0000000000075190 T writeFile
...省略
```
查看 libkn.so 依赖：

```
./aarch64-linux-musl-readelf -d libkn.so

Dynamic section at offset 0x8f348 contains 32 entries:

  Tag        Type                         Name/Value

 0x0000000000000001 (NEEDED)             Shared library: [libresolv.so.2]

 0x0000000000000001 (NEEDED)             Shared library: [libm.so.6]

 0x0000000000000001 (NEEDED)             Shared library: [libpthread.so.0]

 0x0000000000000001 (NEEDED)             Shared library: [libutil.so.1]

 0x0000000000000001 (NEEDED)             Shared library: [libcrypt.so.1]

 0x0000000000000001 (NEEDED)             Shared library: [librt.so.1]

 0x0000000000000001 (NEEDED)             Shared library: [libdl.so.2]

 0x0000000000000001 (NEEDED)             Shared library: [libgcc_s.so.1]

 0x0000000000000001 (NEEDED)             Shared library: [libc.so.6]

 0x0000000000000007 (RELA)               0x12ce0

 0x0000000000000008 (RELASZ)             100152 (bytes)

 0x0000000000000009 (RELAENT)            24 (bytes)

 0x000000006ffffff9 (RELACOUNT)          3860
 
 //...省略
```
libkn.so 依赖：

-   libresolv.so.2：DNS 解析库。

-   libm.so.6：数学库（如三角函数、指数运算）。

-   libpthread.so.0：POSIX 线程库（多线程支持）。

-   libutil.so.1：工具函数库。

-   libcrypt.so.1：加密库。

-   librt.so.1：实时扩展库（如定时器、共享内存）。

-   libdl.so.2：动态链接库，支持运行时加载其他共享库。

-   libgcc_s.so.1：GCC 支持库，处理异常、浮点操作。

-   libc.so.6：C 标准库，基础系统调用封装。

在 FileUtil.kt 中只使用了文件操作功能，明显不需要那么多库。

在 gradle.build.kts 中添加按需链接 `-as-needed`：

```kotlin
//生成linux平台下的 .so
val linuxTargets = listOf(linuxArm64())
linuxTargets.forEach {
    it.binaries {
        executable {
            this.compilation.compileTaskProvider.configure {
                this.compilerOptions.freeCompilerArgs.addAll(
                    listOf("-linker-options", "-as-needed")
                )
            }
        }
        sharedLib {
            //生成 libkn.so
            baseName = "kn"
        }
    }
}
```
freeCompilerArgs 添加的编译选项会交给：

```
~/.konan/kotlin-native-prebuilt-macos-aarch64-2.1.10-RC2/bin/kotlinc-native -h

//...省略
-linker-option <arg>       Pass the given argument to the linker.
//...省略
```
编译再看 libkn.so 依赖：

```
./aarch64-linux-musl-readelf -d libkn.so

Dynamic section at offset 0x8f308 contains 27 entries:

  Tag        Type                         Name/Value

 0x0000000000000001 (NEEDED)             Shared library: [libpthread.so.0]

 0x0000000000000001 (NEEDED)             Shared library: [libdl.so.2]

 0x0000000000000001 (NEEDED)             Shared library: [libgcc_s.so.1]

 0x0000000000000001 (NEEDED)             Shared library: [libc.so.6]
 
  //...省略
```
因为 Kotlin/Native 在生成 linux_arm64 的 so 时，使用了`～/.konan/dependencies/aarch64-unknown-linux-gnu-gcc-8.3.0-glibc-2.25-kernel-4.9-2`中的GCC交叉编译工具链，所以依赖GCC的共享库：`libgcc_s.so.1`。

鸿蒙使用毕昇编译器，基于LLVM，libkn.so 不能依赖`libgcc_s.so.1`（依赖`libgcc_s.so.1`的libkn.so 在鸿蒙中使用会崩溃）。

要让 libkn.so 不依赖`libgcc_s.so.1`，选择鸿蒙毕昇编译器交叉编译或静态链接`libgcc_s.so.1`。这里选择静态链接。

Kotlin/Native 项目构建时，有一个编译工具链配置文件：～/.konan/kotlin-native-prebuilt-macos-aarch64-2.1.10-RC2/konan/konan.properties，在构建 linux_arm64 时，会使用 konan.properties 中有一个配置：`linkerGccFlags = -lgcc --as-needed -lgcc_s --no-as-needed -lc -lgcc --as-needed -lgcc_s --no-as-needed`，该配置表示：强制链接libgcc_s，所以 libkn.so 才会依赖`libgcc_s.so.1`。

去除依赖`libgcc_s.so.1`，在 gradle.build.kts 修改 linkerGccFlags：
```kotlin
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
}
```
当然直接在 konan.properties 中修改 linkerGccFlags 也可以。

再编译再看 libkn.so 依赖：

```
./aarch64-linux-musl-readelf -d libkn.so

Dynamic section at offset 0x8f268 contains 25 entries:

  Tag        Type                         Name/Value

 0x0000000000000001 (NEEDED)             Shared library: [libpthread.so.0]

 0x0000000000000001 (NEEDED)             Shared library: [libdl.so.2]
 
 //...省略
```
此时，libkn.so 符合要求，可以在鸿蒙中使用。

## 鸿蒙接入 Kotlin/Native So

首先在鸿蒙项目中创建 Native C++ 模块 khn（任意命名）：

![截屏2025-02-26 10.41.39.png]({{site.url}}/images/posts/2025-02-27-Kotlin-Native-For-Harmony/p1.png)


将 libkn.so 放在 khn/libs/arm64-v8a/ 目录下，将 libkn_api.h 放在 khn/libs/arm64-v8a/include 目录下。然后修改配置文件 `CMakeLists.txt`：
```c
# the minimum version of CMake.
cmake_minimum_required(VERSION 3.5.0)
project(khn)

set(NATIVERENDER_ROOT_PATH ${CMAKE_CURRENT_SOURCE_DIR})

if(DEFINED PACKAGE_FIND_FILE)
    include(${PACKAGE_FIND_FILE})
endif()

get_filename_component(PROJECT_MAIN_DIR "${CMAKE_CURRENT_SOURCE_DIR}" DIRECTORY)
get_filename_component(PROJECT_SRC_DIR "${PROJECT_MAIN_DIR}" DIRECTORY)
# 获取项目根目录
get_filename_component(PROJECT_DIR "${PROJECT_SRC_DIR}" DIRECTORY)


include_directories(${NATIVERENDER_ROOT_PATH}
                    ${NATIVERENDER_ROOT_PATH}/include)

add_library(khn SHARED napi_init.cpp)
//添加头文件 libkn_api.h
target_include_directories(khn PUBLIC ${PROJECT_DIR}/libs/arm64-v8a/include)
target_link_libraries(khn PUBLIC libace_napi.z.so)
//添加动态库 libkn.so
target_link_libraries(khn PUBLIC ${PROJECT_DIR}/libs/arm64-v8a/libkn.so)
```
接下来在`napi_init.cpp`中利用 Node_API 实现 writeFile 和 readFile 功能：

```c
#include "napi/native_api.h"
#include "libkn_api.h"
#include <cstring>

static napi_value NAPI_Global_writeFile(napi_env env, napi_callback_info info) {
    size_t argc = 2;
    napi_value args[2] = {nullptr};
    napi_get_cb_info(env, info, &argc, args, nullptr, nullptr);
    // 获取字符串的长度
    size_t filePathLength = 0;
    napi_get_value_string_utf8(env, args[0], nullptr, 0, &filePathLength);
    char *filePath = new char[filePathLength + 1];
    std::memset(filePath, 0, filePathLength + 1);
    napi_get_value_string_utf8(env, args[0], filePath, filePathLength + 1, &filePathLength);
    // 获取字符串的长度
    size_t contentLength = 0;
    napi_get_value_string_utf8(env, args[1], nullptr, 0, &contentLength);
    char *content = new char[contentLength + 1];
    std::memset(content, 0, contentLength + 1);
    napi_get_value_string_utf8(env, args[1], content, contentLength + 1, &contentLength);
    // 调用 libkn.so 中的 writeFile
    writeFile(filePath, content);
    // 或使用下面代码也可以
    // libkn_ExportedSymbols *lib = libkn_symbols();
    // lib->kotlin.root.harmonyWriteFile(filePath, content);

    delete[] filePath;
    delete[] content;
    return nullptr;
}
static napi_value NAPI_Global_readFile(napi_env env, napi_callback_info info) {
    size_t argc = 1;
    napi_value args[1] = {nullptr};
    napi_get_cb_info(env, info, &argc, args, nullptr, nullptr);
    // 获取字符串的长度
    size_t filePathLength = 0;
    napi_get_value_string_utf8(env, args[0], nullptr, 0, &filePathLength);
    char *filePath = new char[filePathLength + 1];
    std::memset(filePath, 0, filePathLength + 1);
    napi_get_value_string_utf8(env, args[0], filePath, filePathLength + 1, &filePathLength);
    // 调用 libkn.so 中的 readFile
    const char *content = readFile(filePath);
    // 或使用下面代码也可以
    // libkn_ExportedSymbols *lib = libkn_symbols();
    // const char *content = lib->kotlin.root.harmonyReadFile(filePath);

    napi_value result = nullptr;
    napi_status status = napi_create_string_utf8(env, content, strlen(content), &result);
    delete[] filePath;
    if (status != napi_ok) {
        return nullptr;
    };
    return result;
}

EXTERN_C_START
static napi_value Init(napi_env env, napi_value exports) {
    napi_property_descriptor desc[] = {
        {"writeFile", nullptr, NAPI_Global_writeFile, nullptr, nullptr, nullptr, napi_default, nullptr},
        {"readFile", nullptr, NAPI_Global_readFile, nullptr, nullptr, nullptr, napi_default, nullptr}};
    napi_define_properties(env, exports, sizeof(desc) / sizeof(desc[0]), desc);
    return exports;
}
EXTERN_C_END

static napi_module demoModule = {
    .nm_version = 1,
    .nm_flags = 0,
    .nm_filename = nullptr,
    .nm_register_func = Init,
    .nm_modname = "khn",
    .nm_priv = ((void *)0),
    .reserved = {0},
};

extern "C" __attribute__((constructor)) void RegisterKhnModule(void) { napi_module_register(&demoModule); }
```
通过`#include "libkn_api.h"`访问 writeFile 和 readFile 或 harmonyWriteFile 和 harmonyReadFile。

实现功能后，在 khn/src/main/cpp/types/libkhn/Index.d.ts 定义方法导出：

```typescript
export const writeFile: (filePath: string, content: string) => void;
export const readFile: (filePath: string) => string;
```
在 khn/src/main/ets/pages/KotlinNative.ets 再包装一下（也可以不包装）：
```typescript
import { readFile, writeFile } from 'libkhn.so';

export function nativeWriteFile(filePath: string, content: string): void {
  writeFile(filePath, content);
}

export function nativeReadFile(filePath: string): string {
  return readFile(filePath);
}
```
将 KotlinNative.ets 给其它模块使用，khn/Index.ets：
```js
export * from './src/main/ets/pages/KotlinNative'
```
鸿蒙项目中任意模块依赖 khn 模块：
```
"dependencies": {
  "khn": "file:../khn",
},
```
在模块中使用（测试代码）：
```ts
import { hilog } from '@kit.PerformanceAnalysisKit'
import { common } from '@kit.AbilityKit'
import { fileIo as fs } from '@kit.CoreFileKit'

// 获取应用文件路径
let context = getContext(this) as common.UIAbilityContext;
let filesDir = context.filesDir;

@Entry
@Component
struct Index {
  build() {
    Column() {
      Button("Kotlin/Native")
        .fontSize(14)
        .fontColor(Color.White)
        .backgroundColor(0xFF60DDAD)
        .width(160)
        .height(60)
        .onClick(() => {
          const path = filesDir + "/test.txt"
          hilog.info(0, "Native", "path=" + path)
          fs.open(path, fs.OpenMode.CREATE).then(() => {
            nativeWriteFile(path, "Hello Kotlin/Native, This is Harmony!")
            const content = nativeReadFile(path)
            hilog.info(0, "Native", "content=" + content)
          }).catch((e: Error) => {
            hilog.error(0, "Native", e.message)
          })
        }) }.alignItems(HorizontalAlign.Center).justifyContent(FlexAlign.Center).width("100%").height('100%')
  }
}
```
点击 Button Kotlin/Native，在 Device File Browser 中查看：

![截屏2025-02-26 12.28.50.png]({{site.url}}/images/posts/2025-02-27-Kotlin-Native-For-Harmony/p2.png)

可以看到 Kotlin/Native 生成的 libkn.so 在 Harmony 平台成功运行。

## 总结
通过 Kotlin/Native 直接访问系统底层文件，网络，多媒体，多线程等功能，可以突破 Kotlin/Android, Kotlin/iOS, Kotlin/JS 上层的限制，达到真正的一个API在 Android, iOS, Harmony 平台使用，而且还能保证良好的性能。

在 Kotlin/Native 给鸿蒙使用的过程中，由于 Kotlin/Native 构建 linux_arm64 与 鸿蒙 Native 构建 linux_arm64 编译工具链的不同，导致 Kotlin/Native 生成的动态库在鸿蒙平台运行会有找不到符号，找不到依赖库的问题，虽然在去除依赖上能解决问题，但是最好还是可以使用鸿蒙 NDK 编译工具链进行交叉编译。

