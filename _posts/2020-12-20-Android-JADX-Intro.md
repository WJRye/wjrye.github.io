---
layout: post
title: Android APK 反编译工具 JADX
categories: [Android]
description: JADX 支持将 APK, dex, aar, zip 中的 dalvik 字节码反编译为 Java 代码，也支持反编译 AndroidManifest.xml 和 resources.arsc 资源。
keywords: Android, JADX
mermaid: false
sequence: false
flow: false
mathjax: false
mindmap: false
mindmap2: false
---

## JADX 介绍

>  GitHub 地址：[https://github.com/skylot/jadx](https://github.com/skylot/jadx)

JADX 支持将  APK, dex, aar, zip 中的  dalvik 字节码反编译为 Java 代码，也支持反编译 AndroidManifest.xml 和 resources.arsc 资源。

## JADX 安装

首先安装 [JDK1.8](https://www.oracle.com/java/technologies/javase-downloads.html) 或以上版本，[Git](https://git-scm.com/)，以及 [Android](https://developer.android.google.cn/) 开发环境。

创建要下载的 JADX 存储文件路径，然后在命令行切换到该目录，执行以下命令：

> git clone https://github.com/skylot/jadx.git
> cd jadx
> ./gradlew dist 这个操作可能需要一段时间

如果 执行 `./gradlew dist` 遇到错误：

```
PKIX path building failed...省略
注意查看是连接哪个网址发生错误。
```

**这里从连接哪个网址发现是 gradle 的错误**，解决办法：

1.在 chrome 中输入 gradle 网址：[https://gradle.org/](https://gradle.org/)

2.点击网址输入栏的锁标识
 ![](/images/posts/2020-12-20-Android-JADX-Intro/p1.png)

3.点击证书查看详情
 ![](/images/posts/2020-12-20-Android-JADX-Intro/p2.png)

4.然后将证书拖到要存储的目录

5.查看 cacerts  列表，切换到 JAVA 的 security 路径（本机路径是：`/Library/Java/JavaVirtualMachines/jdk1.8.0_271.jdk/Contents/Home/jre/lib/security/` ），命令行执行：`keytool -list -keystore cacerts  路径`，本机是：

```
keytool -list -keystore  /Library/Java/JavaVirtualMachines/jdk1.8.0_271.jdk/Contents/Home/jre/lib/security/cacerts
```

默认密码为 `changeit`。

6.导入证书，命令执行命令（还是在路径：`/Library/Java/JavaVirtualMachines/jdk1.8.0_271.jdk/Contents/Home/jre/lib/security/`）：

```
keytool  -importcert  -file 第4步存储的证书路径地址
```

默认密码为 `changeit`。

如果这里遇到没有权限，在 Mac  系统下，这里需要：`sudo keytool  -importcert  -file 第4步存储的证书路径地址`。

7.再切换到 JADX 存储文件路径，命令行执行：

> cd jadx
> ./gradlew dist 这个操作可能需要一段时间

## JADX  使用

 进入 JADX 存储文件路径，进入 `jadx/build/jadx/bin` 目录，命令行执行：

```
cd jadx/build/jadx/bin
```

双击 gadx-gui，如下图：

![](/images/posts/2020-12-20-Android-JADX-Intro/p3.png)

然后点击 左上角菜单 文件，选择要打开的文件： APK 或 dex 或 arr 或 zip。

这里在网上下载了[喜马拉雅](https://www.ximalaya.com/)的 Android APK，导入后如下图：

![](/images/posts/2020-12-20-Android-JADX-Intro/p4.png)

到这里，  JADX  工具的简单介绍就完了。可以说这个工具：易用，简洁，强大，非常适合用来反编译。

 平时，反编译 APK 作用：

1. 代码是否符合预期，也就是经过压缩(shrinker), 优化(optimizer), 混淆(obfuscator)后，Java 代码（数据模型类）是否符合预期，R.java 代码中的资源 ID 是否符合预期，AndroidManifest.xml 文件的引用的类有没有错误 ，资源图片是否显示异常等。
2. 要发热修复时，查看 修复后的 APK 中的代码 和   原 APK 中的代码 diff，看修复是否符合期望。
3. 检查 APK 是否有安全性问题，源码是否易被解读，源码中的关键性代码或重要私密 KEY（RSA，AES 等密钥，用户信息等）是否容易被拿到等。

## 补充

 反编译 Android APK 至少需要了解 APK 目录结构含义和 APK 打包流程，这里简单介绍下。

### APK 目录结构含义

这里还是以 [喜马拉雅](https://www.ximalaya.com/)的 Android APK 为例子，直接将 APK  拖入 AndroidStudio :
![](/images/posts/2020-12-20-Android-JADX-Intro/p5.png)

1. res：存在资源的文件，drawable, anim, layout 等；
2. lib：支持CPU架构对应一个ABI（armeabi，armeabi-v7a，x86，mips，arm64- v8a，mips64，x86_64）相关的 so 库；
3. assets：资源文件，但文件不会被压缩或编译成二进制；
4. META-INF ：签名文件；
5. AndroidManifest.xml：清单列表文件，四大组件，权限，元数据等配置信息；
6. classes.dex：编写的Java代码，R.java，  aidl 相关  Java 代码经过虚拟机编译后产生的 dex  文件；                      
7. resources.arsc：建立 res文件 与R.java文件之间的映射关系；
8. 其它：各种配置文件。

### APK 打包流程

1. aapt 工具打包资源文件，生成 R.java 文件；
2. aidl 工具处理 AIDL 文件，生成对应的 .java 文件；
3. javac 工具编译 Java 文件，生成对应的 .class 文件；
4. 把 .class 文件转化成 Davik VM 支持的 .dex 文件；
5. apkbuilder 工具打包生成未签名的 .apk 文件；
6. [jarsigner](https://docs.oracle.com/javase/7/docs/technotes/tools/windows/jarsigner.html) 或 [apksigner](https://developer.android.google.cn/studio/command-line/apksigner?hl=en) 对未签名 .apk 文件进行签名；
7. zipalign 工具对签名后的 .apk 文件进行对齐处理；

zipalign 工具对签名后的 .apk 文件进行对齐处理的原因：目的是确保所有未压缩数据以相对于文件开头的特定对齐开始。具体来说，它会导致APK中的所有未压缩数据（如图像或原始文件）在4字节边界上对齐。对齐以后，系统就能更加快速的调用APP内的资源。
