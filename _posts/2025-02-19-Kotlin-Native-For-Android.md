---
layout: post
title: Kotlin/Native 给 Android 使用
categories: [Kotlin, KMP]
description: Kotlin/Native 能让 Kotlin 代码直接生成符合 JNI 规范的 Native 代码，可以不用再写 .cpp 代码。
keywords: Kotlin, Native
mermaid: false
sequence: false
flow: false
mathjax: false
mindmap: false
mindmap2: false
---

![截屏2025-02-18 13.29.52.png]({{site.url}}/images/posts/2025-02-19-Kotlin-Native-For-Android/p1.png)


## 前言

首先，简单回顾一下 Android Native 项目。

在 Android 项目中，利用 JNI（Java Native Interface） 可以让 Java/Kotlin 代码与 Native 代码（C/C++）进行交互，从而实现音视频编解码，图形渲染等复杂功能。

因为 Java/Kotlin 代码运行在 JVM 环境中， C/C++ 代码运行在 Native（Linux） 环境中，要实现跨语言交互，就必须要一套规范和约束-JNI，才能让 JVM 能找到 Native 中的函数。

在 Android Native 模块中，项目结构一般为：

    └── src

        ├── main

        │   ├── AndroidManifest.xml

        │   ├── cpp

        │   │   ├── CMakeLists.txt  # CMake 构建脚本

        │   │   ├── include # 头文件

        │   │   └── nativelib.cpp # C/C++ 代码

        │   ├── java

        │   │   └── com # Java 代码 

        │   └── jniLibs # so

        │       ├── arm64-v8a

        │       └── armeabi-v7a

其中 `nativelib.cpp` 通过 `#include <jni.h>`等 对 Native 功能进行访问：

```c++
#include <jni.h>
#include <string>
//三方头文件
#include "include/libkn_api.h"

extern "C" JNIEXPORT jstring JNICALL
Java_com_wj_nativelib_NativeLib_stringFromJNI(JNIEnv *env, jobject /* this */) {
    std::string hello = "Hello from C++";
    return env->NewStringUTF(hello.c_str());
}

extern "C" JNIEXPORT jstring JNICALL
Java_com_wj_nativelib_NativeLib_nativeFunction(JNIEnv *env, jobject /* this */) {
    // 调用第三方库函数
    libkn_ExportedSymbols *libkn = libkn_symbols();
    std::string nativeFunction = libkn->kotlin.root.nativeFunction();
    return env->NewStringUTF(nativeFunction.c_str());
}

JNIEXPORT jint JNICALL
JNI_OnLoad(JavaVM *vm, void *reserved) {
    JNIEnv *env = nullptr;
    // 获取 JNIEnv，JNIEnv 用于调用 Java 层的方法和访问 Java 对象
    if (vm->GetEnv((void **)&env, JNI_VERSION_1_6) != JNI_OK) {
        return JNI_ERR;
    }
    // 执行初始化操作，比如：注册方法、初始化全局变量等

    // 返回 JNI 版本号
    return JNI_VERSION_1_6;  // 返回支持的 JNI 版本
}
```

在 com/wj/nativelib/NativeLib.kt 中还需定义与`nativelib.cpp`匹配的 Native API：

```kotlin
package com.wj.nativelib

class NativeLib {

    external fun stringFromJNI(): String
    external fun nativeFunction(): String

    companion object {
        // Used to load the 'nativelib' library on application startup.
        init {
            System.loadLibrary("nativelib")
        }
    }
}
```

此时才能让 Java/Kotlin 代码调用 Native 代码 `nativeFunction()`等。

现在，Kotlin/Native 能让 Kotlin 代码直接生成符合 JNI 规范的 Native 代码，可以不用再写 .cpp 代码。

## Kotlin/Native

### 对 Android 提供的基础能力

Kotlin/Native 对 Android 提供的基础能力可以在` ~/.konan/kotlin-native-prebuilt-macos-aarch64-2.1.10-RC2/klib` 文件目录中找到：

    .

    ├── cache

    ├── common # 标准库

    ├── commonized

    └── platform # 平台库

`common` 和 `platform` 就是 Kotlin/Native 对 Andriod 平台提供的基础能力，在 platform 中支持 android\_arm32，android\_arm64，android\_x64，android\_x86。

下面看下 android\_arm64 目录：

    ├── android_arm64

    │   ├── org.jetbrains.kotlin.native.platform.android 

    │   ├── org.jetbrains.kotlin.native.platform.builtin

    │   ├── org.jetbrains.kotlin.native.platform.egl

    │   ├── org.jetbrains.kotlin.native.platform.gles

    │   ├── org.jetbrains.kotlin.native.platform.gles2

    │   ├── org.jetbrains.kotlin.native.platform.gles3

    │   ├── org.jetbrains.kotlin.native.platform.gles31

    │   ├── org.jetbrains.kotlin.native.platform.glesCommon

    │   ├── org.jetbrains.kotlin.native.platform.linux

    │   ├── org.jetbrains.kotlin.native.platform.media

    │   ├── org.jetbrains.kotlin.native.platform.omxal

    │   ├── org.jetbrains.kotlin.native.platform.posix

    │   ├── org.jetbrains.kotlin.native.platform.sles

    │   └── org.jetbrains.kotlin.native.platform.zlib

| 目录                                                | 作用                                 |
| ------------------------------------------------- | ---------------------------------- |
| `org.jetbrains.kotlin.native.platform.android`    | Android 平台的 Kotlin/Native 绑定       |
| `org.jetbrains.kotlin.native.platform.builtin`    | 内建的 Kotlin/Native 标准库支持            |
| `org.jetbrains.kotlin.native.platform.egl`        | OpenGL EGL 库的 Kotlin/Native 绑定     |
| `org.jetbrains.kotlin.native.platform.gles`       | OpenGL ES 绑定                       |
| `org.jetbrains.kotlin.native.platform.gles2`      | OpenGL ES 2 绑定                     |
| `org.jetbrains.kotlin.native.platform.gles3`      | OpenGL ES 3 绑定                     |
| `org.jetbrains.kotlin.native.platform.gles31`     | OpenGL ES 3.1 绑定                   |
| `org.jetbrains.kotlin.native.platform.glesCommon` | OpenGL ES 通用 API                   |
| `org.jetbrains.kotlin.native.platform.linux`      | Linux 平台 API 绑定                    |
| `org.jetbrains.kotlin.native.platform.media`      | 媒体处理相关 API（可能与 Android 多媒体 API 绑定） |
| `org.jetbrains.kotlin.native.platform.omxal`      | OpenMAX AL（多媒体处理框架）                |
| `org.jetbrains.kotlin.native.platform.posix`      | POSIX API（适用于 Linux/Android）       |
| `org.jetbrains.kotlin.native.platform.sles`       | OpenSL ES（音频 API 绑定）               |
| `org.jetbrains.kotlin.native.platform.zlib`       | zlib 压缩库绑定                         |

每个目录其实都是一个 klib（类似于 Java 的 `.jar`），通过 klib 就能访问这些能力。

`org.jetbrains.kotlin.native.platform.android`提供了 Android Native API，其中包括`platform.android.JNIEnvVar`、`platform.android.JNI_OnLoad`、`platform.android.jobject`等JNI API。通过JNI API 就可以不用再写 .cpp 代码。

### 生成符合 JNI 规范的 `.so`

下载[Kotlin/Native 项目](https://link.juejin.cn/?target=https%3A%2F%2Fgithub.com%2FKotlin%2Fkmp-native-wizard "https://github.com/Kotlin/kmp-native-wizard")模版，将 Kotlin 版本更新到最新（当前`kotlin = "2.1.10-RC2"`）。

首先在 build.gradle.kts 配置支持 android\_arm64：

    kotlin {
        //生成android平台下的 .so
        val androidTargets = listOf(androidNativeArm64())
        androidTargets.forEach {
            it.binaries {
                executable()
                sharedLib {
                    //生成 libkn.so
                    baseName = "kn"
                }
            }
        }  
    }

在 `Main.kt` 中通过 `@CName`注解 和 `platform.android.*` 生成符合 JNI 规范的`.so`：

```kotlin
import kotlinx.cinterop.*
import platform.android.*
import kotlin.experimental.ExperimentalNativeApi

@OptIn(ExperimentalForeignApi::class, ExperimentalNativeApi::class)
@CName(externName="Java_com_wj_mylibrary_NativeLib_getStringFromNative")
fun getStringFromNative(env: CPointer<JNIEnvVar>, thiz: jobject): jstring {
    memScoped {
        return env.pointed.pointed!!.NewStringUTF!!.invoke(
            env, "Hello from Kotlin/Native!".cstr.ptr
        )!!
    }
}

//动态注册
fun sayHello() {
    __android_log_print(ANDROID_LOG_ERROR.toInt(), "Kotlin", "Hello %s", "Kotlin Native!")
}

@OptIn(ExperimentalNativeApi::class, ExperimentalForeignApi::class)
@CName(externName="JNI_OnLoad")
fun JNI_OnLoad(vm: CPointer<JavaVMVar>, preserved: COpaquePointer): jint {
    return memScoped {
        val envStorage = alloc<CPointerVar<JNIEnvVar>>()
        val vmValue = vm.pointed.pointed!!
        val result = vmValue.GetEnv!!(vm, envStorage.ptr.reinterpret(), JNI_VERSION_1_6)
        if (result == JNI_OK) {
            val env = envStorage.pointed!!.pointed!!
            val jclass = env.FindClass!!(envStorage.value, "com/wj/mylibrary/NativeLib".cstr.ptr)
            val jniMethod = allocArray<JNINativeMethod>(1)
            jniMethod[0].fnPtr = staticCFunction(::sayHello)
            jniMethod[0].name = "sayHello".cstr.ptr
            jniMethod[0].signature = "()V".cstr.ptr
            env.RegisterNatives!!(envStorage.value, jclass, jniMethod, 1)
        }
        JNI_VERSION_1_6
    }
}

fun main() {
    println("Hello from Kotlin/Native")
}

fun nativeFunction(): String {
    return "Hello from Kotlin/Native"
}
```

`@CName`注解的作用是： 使顶层函数可从 C/C++ 代码中访问。

`@CName(externName="Java_com_wj_mylibrary_NativeLib_getStringFromNative")`指定导出的符号为：Java\_com\_wj\_mylibrary\_NativeLib\_getStringFromNative。

在命令行执行：`./gradlew linkAndroidNativeArm64 --info`，`build/bin/androidNativeArm64/`目录下会生成动态库`libkn.so`和头文件`libkn_api.h`：

    .
    ├── releaseExecutable

    │   └── KotlinNativeTemplate.kexe

    └── releaseShared

        ├── libkn.so

        └── libkn_api.h

libkn\_api.h 头文件中有：

```c
//声明外部函数
extern libkn_KInt JNI_OnLoad(void* vm, void* preserved);

extern void* Java_com_wj_mylibrary_NativeLib_getStringFromNative(void* env, void* thiz);

```

简单查看一下 libkn.so 信息：

    file libkn.so

    ELF 64-bit LSB shared object, ARM aarch64, version 1 (SYSV), dynamically linked, not stripped

再看一下 libkn.so 符号：

    ./aarch64-linux-musl-nm -g libkn.so

    000000000004e438 T JNI_OnLoad

    000000000004e4f8 T Java_com_wj_mylibrary_NativeLib_getStringFromNative

    ...省略

### 在 Android 中使用 `.so`

在 Android 新建 Module `mylibrary`：
.

├── build.gradle.kts

└── src

    ├── main

    │   ├── AndroidManifest.xml

    │   ├── java

    │   │   └── com

    │   │       └── wj

    │   │           └── mylibrary

    │   │               └── NativeLib.kt

    │   └── jniLibs

    │       └── arm64-v8a

    │           └── libkn.so

build.gradle.kts 配置：

```koltin
android {
    namespace = "com.wj.mylibrary"

    defaultConfig {
        
        ndk {
            abiFilters.clear() // 清除所有默认的 ABI
            abiFilters.add("arm64-v8a") // 只添加 arm64-v8a
        }
    }

    sourceSets {
        this.getByName("main") {
            this.jniLibs.srcDirs("src/main/jniLibs")
        }
    }
}
```

在 NativeLib.kt 中声明 Native 方法：

```kotlin
package com.wj.mylibrary

class NativeLib {
    external fun getStringFromNative(): String
    external fun sayHello()

    companion object {
        init {
            System.loadLibrary("kn")
        }
    }
}
```

测试 Native 方法：

```kotlin
fun testNative() {
    val nativeLib = NativeLib()
    val value = nativeLib.getStringFromNative()
    Log.e("Native", value)
    nativeLib.sayHello()
}
```

日志平台输出：

![截屏2025-02-18 13.29.52.png]({{site.url}}/images/posts/2025-02-19-Kotlin-Native-For-Android/p2.png)

## 总结

在 Kotlin/Native 中能直接访问 Android Native API，OpenGL，音视频处理等能力，同时结合 `@CName` 注解，生成符合 `JNI` 规范的 `.so`，这让 Android Native 开发，更加的丝滑，简单和容易。

