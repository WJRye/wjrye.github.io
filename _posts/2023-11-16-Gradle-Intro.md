---
layout: post
title: 使用 Gradle 命令了解项目构建信息
categories: [Gradle]
description: 不管是接触一个新项目，还是一直开发老项目，使用 Gradle 命令，可以对项目构建信息有一个快速的掌握。
keywords: Gradle
mermaid: false
sequence: false
flow: false
mathjax: false
mindmap: false
mindmap2: false
---

## 引言

首先，Gradle 作为使用 Android Studio 开发 Android 项目的默认构建工具，它里面的任何东西都基于两个概念：

- projects ( 项目 )
- tasks ( 任务 )

每一个构建由一个或多个 projects 构成，每一个 project 由一个或多个 tasks 构成。每一个 task 代表细化的构建节点，比如 APK 的构建过程：

> 2. 通过 aapt 打包 res 资源文件，生成 R.java、resources.arsc 和 res 文件（二进制和非二进制如 res/raw 和 pic 保持原样）
> 
> 3. 处理 .aidl 文件，生成对应的 Java 接口文件
> 
> 4. 通过 Java Compiler 编译 R.java、Java 接口文件、Java 源文件，生成 .class 文件
> 
> 5. 通过 dx/d8 工具，将 .class 文件和第三方库中的 .class 文件处理生成 classes.dex
> 
> 6. 通过 apkbuilder 工具，将 aapt 生成的 resources.arsc 和 res 文件、assets 文件和 classes.dex 一起打包生成 apk
> 
> 7. 通过 Jarsigner 工具，对上面的 apk 进行 debug 或 release 签名
> 
> 8. 通过 zipalign 工具，对签名后的 apk 进行对齐处理

在这个构建过程中的每一步，都有与之相关的 task 相呼应：

```
> Task :app:preBuild UP-TO-DATE
> Task :app:preDebugBuild UP-TO-DATE
> Task :app:mergeDebugNativeDebugMetadata NO-SOURCE
> Task :app:compileDebugAidl NO-SOURCE  //处理 aidl
> Task :app:compileDebugRenderscript NO-SOURCE
> Task :app:generateDebugBuildConfig UP-TO-DATE
> Task :app:checkDebugAarMetadata UP-TO-DATE
> Task :app:generateDebugResValues UP-TO-DATE
> Task :app:generateDebugResources UP-TO-DATE
> Task :app:mergeDebugResources UP-TO-DATE  //合并资源文件
> Task :app:packageDebugResources UP-TO-DATE
> Task :app:mapDebugSourceSetPaths UP-TO-DATE
> Task :app:parseDebugLocalResources UP-TO-DATE
> Task :app:createDebugCompatibleScreenManifests UP-TO-DATE
> Task :app:extractDeepLinksDebug UP-TO-DATE
> Task :app:processDebugMainManifest UP-TO-DATE
> Task :app:processDebugManifest UP-TO-DATE
> Task :app:processDebugManifestForPackage UP-TO-DATE
> Task :app:javaPreCompileDebug UP-TO-DATE
> Task :app:mergeDebugShaders
> Task :app:compileDebugShaders NO-SOURCE
> Task :app:generateDebugAssets UP-TO-DATE
> Task :app:mergeDebugAssets  //合并 assets 文件
> Task :app:compressDebugAssets
> Task :app:processDebugJavaRes NO-SOURCE
> Task :app:checkDebugDuplicateClasses
> Task :app:mergeDebugJniLibFolders
> Task :app:mergeLibDexDebug
> Task :app:mergeDebugNativeLibs NO-SOURCE
> Task :app:stripDebugDebugSymbols NO-SOURCE
> Task :app:validateSigningDebug
> Task :app:writeDebugAppMetadata
> Task :app:writeDebugSigningConfigVersions
> Task :app:desugarDebugFileDependencies
> Task :app:processDebugResources  // aapt 打包资源
> Task :app:mergeExtDexDebug
> Task :app:compileDebugKotlin UP-TO-DATE  // 编译 kotlin 文件
> Task :app:compileDebugJavaWithJavac UP-TO-DATE  //编译 java 文件
> Task :app:dexBuilderDebug  //.class 文件转换为 dex 归档文件
> Task :app:mergeProjectDexDebug
> Task :app:mergeDebugJavaResource
> Task :app:packageDebug  //打包 apk
> Task :app:createDebugApkListingFileRedirect
> Task :app:assembleDebug
```

Task（任务）作为构建过程中的最小原子工作单元。所以要了解 Android 项目的相关构建信息，其实就是了解相关 Gradle  Tasks。

在日常开发中，如果接触一个新项目，可能最先想了解的是：

- [ ] 这个项目有哪些个 Project？
- [ ] 这些Project 的依赖关系是怎么样的？
- [ ] 你要参与的Project 使用了哪些技术库？

如果一直在开发某项目，可能会时不时用到：

- [ ] 输出项目库依赖信息，找到某个库的依赖情况？
- [ ] 升级某个库，需要升级哪些依赖它的库，以及影响哪些业务模块？
- [ ] 输出项目依赖的 so 情况，找到 so 来源？

下面将简单介绍在 Android 项目构建中，会用到的一些 Gradle Task 命令。

 备注：假设你已经对 Task 有所提前了解。

## Gradle 命令

首先，在命令行执行的每一个 gradle 命令，其实都是执行的一个 task：

```
 ./gradlew 

> Task :help

Welcome to Gradle 7.3.3.

To run a build, run gradlew <task> ...

To see a list of available tasks, run gradlew tasks

To see more detail about a task, run gradlew help --task <task>

To see a list of command-line options, run gradlew --help

For more detail on using Gradle, see https://docs.gradle.org/7.3.3/userguide/command_line_interface.html

For troubleshooting, visit https://help.gradle.org

BUILD SUCCESSFUL in 2s
1 actionable task: 1 executed
```

可以看到输入`./gradlew `命令，其实是执行的 help task: `./gradlew :help`。

每一个 task 的基础信息包含：Path（路径）, Type（类型），Options（选项），Description（描述），Group（分组）：

```
./gradlew help --task :help            

> Task :help
Detailed task information for :help

Path
     :help

Type
     Help (org.gradle.configuration.Help)

Options
     --task     The task to show help for.

Description
     Displays a help message.

Group
     help
```

> Type（类型）可以在自定义Gradle Plugin 时，通过依赖：`com.android.tools.build:gradle`，查看 Task 源码。

task 信息有助于了解该 task 在项目构建中的作用和功能。

知道上面基本情况后，下面就开始了解项目构建信息。

### 基础信息

#### tasks

`Task :tasks` 用于输出项目所有的 task：

```
./gradlew :tasks

> Task :tasks

------------------------------------------------------------
Tasks runnable from root project 'gradle_command_demo'
------------------------------------------------------------

Android tasks
-------------
androidDependencies - Displays the Android dependencies of the project.
signingReport - Displays the signing info for the base and test modules
sourceSets - Prints out all the source sets defined in this project.

Help tasks
----------
buildEnvironment - Displays all buildscript dependencies declared in root project 'gradle_command_demo.
dependencies - Displays all dependencies declared in root project gradle_command_demo'.
dependencyInsight - Displays the insight into a specific dependency in root project 'gradle_command_demo'.
help - Displays a help message.
javaToolchains - Displays the detected java toolchains.
outgoingVariants - Displays the outgoing variants of root project 'gradle_command_demo'.
projects - Displays the sub-projects of root project 'gradle_command_demo'.
properties - Displays the properties of root project 'gradle_command_demo'.
tasks - Displays the tasks runnable from root project 'gradle_command_demo' (some of the displayed tasks may belong to subprojects).

//......省略
```

上面输出的是根项目相关的所有task，包含自定义的 task。如果要输出子项目的所有task，可以是使用：`/gradlew :<module>:tasks`，也可以使用：`./gradlew :tasks --all`。如果要查看单个 task 的信息，可以使用：`./gradlew :help --task <task> `，其中`<task>`是 task 的 Path。

#### buildEnvironment

`Task :buildEnvironment` 用于输出根项目的所有构建依赖，也就是根项目下 `build.gradle` 中声明的 `dependencies` 依赖详细信息：

```
./gradlew :buildEnvironment

> Task :buildEnvironment

------------------------------------------------------------
Root project 'gradle_command_demo'
------------------------------------------------------------

classpath
+--- com.android.tools.build:gradle:7.2.0
|    +--- com.android.tools:sdk-common:30.2.0
//......省略
+--- org.greenrobot:greendao-gradle-plugin:3.3.0
|    \--- org.greenrobot:greendao-code-modifier:3.3.0
//......省略
+--- com.google.protobuf:protobuf-gradle-plugin:0.8.18
|    +--- com.google.guava:guava:27.0.1-jre -> 30.1.1-jre (*)
//......省略
\--- org.jetbrains.kotlin:kotlin-gradle-plugin:1.6.21
     +--- org.jetbrains.kotlin:kotlin-gradle-plugin-api:1.6.21
//......省略
```

可以看到根项目依赖的 gradle 和 kotlin 插件相关信息，以及自定义或三方插件信息。

#### projects

`Task :projects` 用于输出项目列表：

```
./gradlew :projects

> Task :projects

------------------------------------------------------------
Root project 'gradle_command_demo'
------------------------------------------------------------

Root project 'gradle_command_demo'
+--- Project ':app'
+--- Project ':bgm'
+--- Project ':common'
|    +--- Project ':common:aiservice'
|    +--- Project ':common:rpc'
|    |    +--- Project ':common:rpc:proto-core'
|    |    \--- Project ':common:rpc:protogen'
+--- Project ':material'
```

可以看到该项目下所有的子项目，以及子项目层级。

#### propreties

`Task :propreties` 用于输出项目定义的属性信息，包含项目下`gradle.properties` 文件中定义的属性信息：

```
./gradlew :properties

> Task :properties

------------------------------------------------------------
Root project 'gradle_command_demo'
------------------------------------------------------------
allprojects: [root project 'gradle_command_demo', project ':app', project ':material', project ':bgm']
android.databinding.incremental: true
android.enableJetifier: true
android.injected.testOnly: false
android.lifecycleProcessor.incremental: true
android.useAndroidX: true
```

在实际项目中，属性信息包含项目基础信息、Java版本号、Plugin版本号、三方库版本号、Maven私有地址等。如果要查看某个子项目的属性信息，可以是使用：`/gradlew :<module>:propreties`

### 项目依赖

#### dependencies

`Task :app:dependencies`用于输出项目依赖项的树状结构，但在实际应用中，直接使用：`./gradlew :<module>:dependencies`会输出很多冗余信息。`dependencies` 有一个选项：

```
Options
     --configuration     The configuration to generate the report for.
```

`configuration` 有debug, release, debugAndroidTest, debugUnitTest, releaseUnitTest等阶段配置，这里选择 release 阶段 的 `releaseRuntimeClasspath：Dependencies for runtime/packaging`，那么就可以看到项目在打 release 包时的正式依赖项信息：

```
./gradlew :app:dependencies --configuration releaseRuntimeClassPath

> Task :app:dependencies

------------------------------------------------------------
Project ':app'
------------------------------------------------------------

releaseRuntimeClasspath - Runtime classpath of compilation 'release' (target  (androidJvm)).
+--- org.jetbrains.kotlin:kotlin-stdlib-jdk8:1.6.21
|    +--- org.jetbrains.kotlin:kotlin-stdlib:1.6.21
|    |    +--- org.jetbrains.kotlin:kotlin-stdlib-common:1.6.21
|    |    \--- org.jetbrains:annotations:13.0
|    \--- org.jetbrains.kotlin:kotlin-stdlib-jdk7:1.6.21
|         \--- org.jetbrains.kotlin:kotlin-stdlib:1.6.21 (*)
+--- androidx.core:core-ktx:1.7.0
+--- com.squareup.okhttp3:okhttp:3.12.13.18 (*)
//.....省略
```

如果 dependencies 信息太多，可以保存到文件：`./gradlew :app:dependencies --configuration releaseRuntimeClassPath > dependencies.txt`，这样更方便搜索和查看。项目依赖项信息非常有用，可以用来排查依赖冲突、分析库依赖关系等。

#### androidDependencies

上面的`Task :app:dependencies` 输出的是Android和Java 项目的依赖项，也就是有在项目build.gradle中声明：`plugins {
    id 'com.android.application'}`或`'plugins { id com.android.library'}`或`plugins {
    id 'java-library'}`的项目。

 `Task :app:androidDependencies`用于输出项目依赖的Android 库信息，以平铺的方式输出，并包含所有的 configuration：

```
releaseRuntimeClasspath - Dependencies for runtime/packaging
+--- org.jetbrains.kotlin:kotlin-stdlib-jdk8:1.6.21@jar
+--- androidx.core:core-ktx:1.7.0@aar
+--- com.google.android.material:material:1.5.0@aar
+--- androidx.constraintlayout:constraintlayout:2.0.1@aar
//......省略
```

这个输出信息中除了关注依赖的 Android 库外，还可以关注所支持的 configuration 信息，configuration 有助于在自定义 task 中进行使用，也有助于下面的 `dependencyInsight`使用。

#### dependencyInsight

执行`Task :app:dependencies`后输出的信息，可以用于查看某个项目的依赖项树状结构，也可以用于查看某个依赖库在某个项目中的依赖情况，比如：project: app 依赖 `com.squareup.okhttp3:okhttp:`。但是要罗列出`com.squareup.okhttp3:okhttp:`被哪些项目和库依赖的所有信息，就不是很方便，这时就需要使用`dependencyInsight`。

 `Task :app:dependencyInsight` 用于输出项目中特定依赖项的详细信息，关于这个 task 的基础信息为：

```
Detailed task information for :app:dependencyInsight

Path
     :app:dependencyInsight

Type
     DependencyInsightReportTask (org.gradle.api.tasks.diagnostics.DependencyInsightReportTask)

Options
     --configuration     Looks for the dependency in given configuration.

     --dependency     Shows the details of given dependency.

     --singlepath     Show at most one path to each dependency

Description
     Displays the insight into a specific dependency in project ':app'.

Group
     help
```

使用这个 task 的方式是：`./gradlew :app:dependencyInsight --configuration someConf --dependency someDep`，必须有 --configuration 和 --dependency 选项。对于 --configuration 选项参数，可以通过上面的`Task :app:androidDependencies` 查看。

现在假设使用`Task :app:dependencies` 输出的信息如下:

```
> Task :app:dependencies

------------------------------------------------------------
Project ':app'
------------------------------------------------------------

releaseRuntimeClasspath - Runtime classpath of compilation 'release' (target  (androidJvm)).
+--- org.jetbrains.kotlin:kotlin-stdlib-jdk8:1.6.21
|    //......省略
+--- androidx.core:core-ktx:1.7.0
|    +--- org.jetbrains.kotlin:kotlin-stdlib:1.5.31 -> 1.6.21 (*)
|    +--- androidx.annotation:annotation:1.1.0 -> 1.3.0
|    \--- androidx.core:core:1.7.0
|         +--- androidx.annotation:annotation:1.2.0 -> 1.3.0
|    //......省略
+--- androidx.appcompat:appcompat:1.4.1
|    +--- androidx.annotation:annotation:1.3.0
|    +--- androidx.core:core:1.7.0 (*)
|    +--- androidx.cursoradapter:cursoradapter:1.0.0
|    |    \--- androidx.annotation:annotation:1.0.0 -> 1.3.0
|    +--- androidx.activity:activity:1.2.4
|    |    +--- androidx.annotation:annotation:1.1.0 -> 1.3.0
|    |    +--- androidx.core:core:1.1.0 -> 1.7.0 (*)
|    |    //......省略
|    +--- androidx.fragment:fragment:1.3.6
|    |    +--- androidx.annotation:annotation:1.1.0 -> 1.3.0
|    |    +--- androidx.core:core:1.2.0 -> 1.7.0 (*)
|    |    +--- androidx.collection:collection:1.1.0 (*)
|    |    +--- androidx.viewpager:viewpager:1.0.0
|    |    |    +--- androidx.annotation:annotation:1.0.0 -> 1.3.0
|    |    |    +--- androidx.core:core:1.0.0 -> 1.7.0 (*)
|    |    |    \--- androidx.customview:customview:1.0.0 -> 1.1.0
|    |    |         +--- androidx.annotation:annotation:1.1.0 -> 1.3.0
|    |    |         +--- androidx.core:core:1.3.0 -> 1.7.0 (*)
|    |    |         \--- androidx.collection:collection:1.1.0 (*)
//......省略
\--- com.google.android.material:material:1.5.0
     +--- androidx.annotation:annotation:1.2.0 -> 1.3.0
     +--- androidx.appcompat:appcompat:1.1.0 -> 1.4.1 (*)
     +--- androidx.cardview:cardview:1.0.0
     |    \--- androidx.annotation:annotation:1.0.0 -> 1.3.0
     +--- androidx.coordinatorlayout:coordinatorlayout:1.1.0
     |    +--- androidx.annotation:annotation:1.1.0 -> 1.3.0
     |    +--- androidx.core:core:1.1.0 -> 1.7.0 (*)
     |    +--- androidx.customview:customview:1.0.0 -> 1.1.0 (*)
     |    \--- androidx.collection:collection:1.0.0 -> 1.1.0 (*)
     +--- androidx.constraintlayout:constraintlayout:2.0.1
     |    +--- androidx.appcompat:appcompat:1.2.0 -> 1.4.1 (*)
     |    +--- androidx.core:core:1.3.1 -> 1.7.0 (*)
     |    \--- androidx.constraintlayout:constraintlayout-solver:2.0.1
     //......省略

(*) - dependencies omitted (listed previously)
```

那么要查看项目对`androidx.core:core`库的依赖情况，只能一条一条梳理：

1. ` androidx.core:core-ktx:1.7.0` 依赖 `androidx.core:core:1.7.0`
2. `androidx.appcompat:appcompat:1.4.1` 依赖 `androidx.core:core:1.7.0`
3. `androidx.activity:activity:1.2.4` 依赖 `androidx.core:core:1.7.0`
4. `androidx.fragment:fragment:1.3.6` 依赖 `androidx.core:core:1.2.0 -> 1.7.0 (*)`
5. 等等

（虽然不同库对 `androidx.core:core`库依赖版本不一样，但最终都只会使用一版本，使用哪一个版本由项目配置决定，可以使用1.2.0 ，也可以使用1.7.0。）

使用`dependencyInsight`罗列出`androidx.core:core:1.7.0`库被依赖的所有信息：

```
./gradlew :app:dependencyInsight --configuration releaseRuntimeClassPath --dependency androidx.core:core:1.7.0


> Task :app:dependencyInsight
androidx.core:core:1.7.0
   variant "releaseVariantReleaseRuntimePublication" [
      org.gradle.category                             = library (not requested)
      org.gradle.dependency.bundling                  = external (not requested)
      org.gradle.libraryelements                      = aar (not requested)
      org.gradle.usage                                = java-runtime
      org.gradle.status                               = release (not requested)

      Requested attributes not found in the selected variant:
         com.android.build.api.attributes.BuildTypeAttr  = release
         org.gradle.jvm.environment                      = android
         com.android.build.api.attributes.AgpVersionAttr = 7.2.0
         org.jetbrains.kotlin.platform.type              = androidJvm
   ]
   Selection reasons:
      - By conflict resolution : between versions 1.7.0, 1.5.0, 1.1.0, 1.2.0, 1.0.1, 1.3.0, 1.3.1 and 1.0.0

androidx.core:core:1.7.0
+--- androidx.appcompat:appcompat:1.4.1
|    +--- releaseRuntimeClasspath
|    +--- com.google.android.material:material:1.5.0 (requested androidx.appcompat:appcompat:1.1.0)
|    |    \--- releaseRuntimeClasspath
|    \--- androidx.constraintlayout:constraintlayout:2.0.1 (requested androidx.appcompat:appcompat:1.2.0)
|         \--- com.google.android.material:material:1.5.0 (*)
\--- androidx.core:core-ktx:1.7.0
     \--- releaseRuntimeClasspath

androidx.core:core:1.0.0 -> 1.7.0
+--- androidx.dynamicanimation:dynamicanimation:1.0.0
|    \--- com.google.android.material:material:1.5.0
|         \--- releaseRuntimeClasspath
+--- androidx.legacy:legacy-support-core-utils:1.0.0
|    \--- androidx.dynamicanimation:dynamicanimation:1.0.0 (*)
+--- androidx.loader:loader:1.0.0
|    +--- androidx.fragment:fragment:1.3.6
|    |    +--- com.google.android.material:material:1.5.0 (requested androidx.fragment:fragment:1.0.0) (*)
|    |    +--- androidx.appcompat:appcompat:1.4.1
|    |    |    +--- releaseRuntimeClasspath
|    |    |    +--- com.google.android.material:material:1.5.0 (requested androidx.appcompat:appcompat:1.1.0) (*)
|    |    |    \--- androidx.constraintlayout:constraintlayout:2.0.1 (requested androidx.appcompat:appcompat:1.2.0)
|    |    |         \--- com.google.android.material:material:1.5.0 (*)
|    |    \--- androidx.viewpager2:viewpager2:1.0.0 (requested androidx.fragment:fragment:1.1.0)
|    |         \--- com.google.android.material:material:1.5.0 (*)
|    \--- androidx.legacy:legacy-support-core-utils:1.0.0 (*)
\--- androidx.viewpager:viewpager:1.0.0
     \--- androidx.fragment:fragment:1.3.6 (*)

//......省略

androidx.core:core:1.3.1 -> 1.7.0
\--- androidx.constraintlayout:constraintlayout:2.0.1
     \--- com.google.android.material:material:1.5.0
          \--- releaseRuntimeClasspath

androidx.core:core:1.5.0 -> 1.7.0
\--- com.google.android.material:material:1.5.0
     \--- releaseRuntimeClasspath

(*) - dependencies omitted (listed previously)
```

首先，可以看到`androidx.core:core`库在项目中有多个版本：

```
  Selection reasons:
      - By conflict resolution : between versions 1.7.0, 1.5.0, 1.1.0, 1.2.0, 1.0.1, 1.3.0, 1.3.1 and 1.0.0
```

其次，查看库被依赖的情况，例如：

```
androidx.core:core:1.7.0
+--- androidx.appcompat:appcompat:1.4.1
|    +--- releaseRuntimeClasspath
|    +--- com.google.android.material:material:1.5.0 (requested androidx.appcompat:appcompat:1.1.0)
|    |    \--- releaseRuntimeClasspath
|    \--- androidx.constraintlayout:constraintlayout:2.0.1 (requested androidx.appcompat:appcompat:1.2.0)
|         \--- com.google.android.material:material:1.5.0 (*)
\--- androidx.core:core-ktx:1.7.0
     \--- releaseRuntimeClasspath
```

androidx.core:core:1.7.0 被 androidx.appcompat:appcompat:1.4.1 依赖，androidx.appcompat:appcompat:1.4.1 被 com.google.android.material:material:1.5.0 依赖，com.google.android.material:material:1.5.0 作为依赖的源头。以此类推，androidx.core:core:1.7.0 被 androidx.core:core-ktx:1.7.0 依赖，androidx.core:core-ktx:1.7.0 作为依赖的源头。

以此，可以看到`androidx.core:core:1.7.0`库在项目中被依赖的全部详细信息，这将有助于做库的版本升级。

在这里，如果只是为了查看`androidx.core:core`，可以不区分版本号：

```
./gradlew :app:dependencyInsight --configuration releaseRuntimeClassPath --dependency androidx.core:core
```

另外，如果要查看依赖项的路径，可以添加选项 --singlepath（每个依赖项最多显示一个路径）：

```
./gradlew :app:dependencyInsight --configuration releaseRuntimeClassPath --dependency androidx.core:core:1.7.0 --singlepath

> Task :app:dependencyInsight
androidx.core:core:1.7.0
   variant "releaseVariantReleaseRuntimePublication" [
      org.gradle.category                             = library (not requested)
      org.gradle.dependency.bundling                  = external (not requested)
      org.gradle.libraryelements                      = aar (not requested)
      org.gradle.usage                                = java-runtime
      org.gradle.status                               = release (not requested)

      Requested attributes not found in the selected variant:
         com.android.build.api.attributes.BuildTypeAttr  = release
         org.gradle.jvm.environment                      = android
         com.android.build.api.attributes.AgpVersionAttr = 7.2.0
         org.jetbrains.kotlin.platform.type              = androidJvm
   ]
   Selection reasons:
      - By conflict resolution : between versions 1.7.0, 1.5.0, 1.1.0, 1.2.0, 1.0.1, 1.3.0, 1.3.1 and 1.0.0

androidx.core:core:1.7.0
\--- androidx.appcompat:appcompat:1.4.1
     \--- releaseRuntimeClasspath

androidx.core:core:1.0.0 -> 1.7.0
\--- androidx.dynamicanimation:dynamicanimation:1.0.0
     \--- com.google.android.material:material:1.5.0
          \--- releaseRuntimeClasspath

androidx.core:core:1.0.1 -> 1.7.0
\--- androidx.appcompat:appcompat-resources:1.4.1
     \--- androidx.appcompat:appcompat:1.4.1
          \--- releaseRuntimeClasspath

androidx.core:core:1.1.0 -> 1.7.0
\--- androidx.activity:activity:1.2.4
     \--- androidx.appcompat:appcompat:1.4.1
          \--- releaseRuntimeClasspath

androidx.core:core:1.2.0 -> 1.7.0
\--- androidx.drawerlayout:drawerlayout:1.1.1
     \--- com.google.android.material:material:1.5.0
          \--- releaseRuntimeClasspath

androidx.core:core:1.3.0 -> 1.7.0
\--- androidx.customview:customview:1.1.0
     \--- androidx.drawerlayout:drawerlayout:1.1.1
          \--- com.google.android.material:material:1.5.0
               \--- releaseRuntimeClasspath

androidx.core:core:1.3.1 -> 1.7.0
\--- androidx.constraintlayout:constraintlayout:2.0.1
     \--- com.google.android.material:material:1.5.0
          \--- releaseRuntimeClasspath

androidx.core:core:1.5.0 -> 1.7.0
\--- com.google.android.material:material:1.5.0
     \--- releaseRuntimeClasspath
```

可以看到依赖的每一个路径。

### 构建选项

使用 `./gradlew --help` 可以看到构建支持的所有选项操作：

```
USAGE: gradlew [option...] [task...]

-?, -h, --help                     Shows this help message.
-a, --no-rebuild                   Do not rebuild project dependencies.
-b, --build-file                   Specify the build file. [deprecated]
--build-cache                      Enables the Gradle build cache. Gradle will try to reuse outputs from previous builds.
//......省略
```

每个选项都有助于分析构建时的问题，下面将对其中几个选项进行简单介绍。

#### --profile

--profile 选项输出构建时会执行的任务列表，以及每一个任务耗时情况：

```
./gradlew :app:assembleDebug --profile
```

执行完后，会输出一个 html  格式的报告文档：

![loading-ag-501](/images/posts/2023-11-16-Gradle-Intro/p1.webp)
在这个报告文档中，有每一个任务的执行时长明细，这对优化构建时间非常有用。

备注：优化构建时间，对于开发来说，是首要的事情，因为开发每天花费时间最多的就是构建应用程序。

#### -q

-q 选项表示静默构建，只会打印出 error 信息：

```
./gradlew :app:assembleDebug -q
```

可以节约构建时间。

#### --offline

--offline 选项表示构建时不访问网络资源：

```
./gradlew :app:assembleDebug --offline -q 
```

默认情况，构建时会访问网络资源，比如访问 Maven 仓库资源。因此使用 --offline 选项也可以节约构建时间。在开发阶段，第一次构建完成后，后续在只更改代码或资源文件的情况下，构建都可以使用 --offline。

#### --info

--info 选项表示构建时，打印出 info 信息：

```
./gradlew :app:assembleDebug --info
```

info 信息包含构建时，所有 task 相关的信息，这有助于排查构建问题，或帮助熟悉项目构建情况。

#### --s

--s 选项表示构建时，打印出异常的所有栈信息：

```
./gradlew :app:assembleDebug --s
```

这有助于排查构建问题。

#### --refresh-dependencies

--refresh-dependencies 选项表示构建时，刷新相关依赖：

```
./gradlew :app:assembleDebug --refresh-dependencies
```

如果更新了项目相关依赖库版本，在构建后发现没有生效，就可以使用这个选项。

#### --no-build-cache

--no-build-cache 选项表示构建时，不使用构建缓存：

```
./gradlew :app:assembleDebug --no-build-cache
```

如果更改了项目代码，在构建后发现没有生效，就可以使用这个选项，或者执行 clean。

备注：这个选项在 Gitlab CI 中构建，或远程构建项目时，发现更改代码没有生效，可以选择使用。

### so 依赖信息

 `Task :app:dependencies` 或 `Task :app:androidDependencies` 可以输出项目依赖的 aar 信息，但要输出项目依赖的 so 信息，并没有相关的 task 或者构建选项可以直接使用。

在 apk 产物中，项目依赖的本地或三方 so 都会放在 lib 文件夹下面：
![在这里插入图片描述](/images/posts/2023-11-16-Gradle-Intro/p2.webp)
另外，构建 apk 时，打包 res 资源文件、处理 .aidl 文件、生成 .class 文件等都有相关的 task 相对应。那么，将 so 都放到 lib 文件下面，可以猜测也应该有相关的 task 相对应。

在构建项目时 `./gradlew :app:assembleDebug --console=plain`：

```
//......省略
> Task :app:mergeDebugJniLibFolders
> Task :app:mergeLibDexDebug
> Task :app:mergeDebugNativeLibs NO-SOURCE
> Task :app:stripDebugDebugSymbols NO-SOURCE
 //......省略
```

可以看到其中有一个 task：`Task :app:mergeDebugNativeLibs NO-SOURCE`，再通过 help 看下这个 task 的基础信息：

```
> Task :help
Detailed task information for :app:mergeDebugNativeLibs

Path
     :app:mergeDebugNativeLibs

Type
     MergeNativeLibsTask (com.android.build.gradle.internal.tasks.MergeNativeLibsTask)

Description
     -

Group
     -
```

没有描述，通过 Type 查看源码：

```
/**
 * Task to merge native libs from a project and possibly its dependencies
 */
@DisableCachingByDefault
abstract class MergeNativeLibsTask : NonIncrementalTask() {
   @get:InputFiles
    @get:PathSensitive(PathSensitivity.RELATIVE)
    @get:SkipWhenEmpty
    @get:IgnoreEmptyDirectories
    abstract val projectNativeLibs: ConfigurableFileCollection //本项目 so 文件列表

    @get:InputFiles
    @get:PathSensitive(PathSensitivity.RELATIVE)
    @get:SkipWhenEmpty
    @get:IgnoreEmptyDirectories
    abstract val subProjectNativeLibs: ConfigurableFileCollection //子项目 so 文件列表

    @get:InputFiles
    @get:PathSensitive(PathSensitivity.RELATIVE)
    @get:SkipWhenEmpty
    @get:IgnoreEmptyDirectories
    abstract val externalLibNativeLibs: ConfigurableFileCollection //三方库 so 文件列表
}
```

这个任务其实就是合并项目本地依赖或三方依赖的Native库。那么，要输出项目依赖的 so 信息，就可以通过` Task:app:mergeDebugNativeLibs` 实现。

在 app 下的 build.gradle 中添加 ` Task:app:mergeDebugNativeLibs`  相关监听：

```
project.afterEvaluate {
    project.android.applicationVariants.all { variant ->
        //获取构建变体的名称
        logger.lifecycle("${variant.name}")
        def variantName = variant.name
        def name = String.valueOf(variantName.charAt(0)).toUpperCase() + variantName.substring(1)
        def mergeNativeLibsTask = project.tasks.getByName("merge${name}NativeLibs")
        mergeNativeLibsTask.doLast { task ->
            logger.lifecycle("project native libs:")
            //当前项目相关的 so 文件列表
            logger.lifecycle("${task.projectNativeLibs.getFiles()}")
            logger.lifecycle("------")
            //子项目相关的 so 文件列表
            logger.lifecycle("sub project native libs:")
            logger.lifecycle("${task.subProjectNativeLibs.getFiles()}")
            logger.lifecycle("------")
            //三方库相关的 so 文件列表
            logger.lifecycle("external project native libs:")
            logger.lifecycle("${task.externalLibNativeLibs.getFiles()}")
        }
    }
}
```

然后执行 `./gradlew :app:mergeDebugNativeLibs`：

```
> Task :app:mergeDebugNativeLibs
project native libs:
[]
------
sub project native libs:
[/Users/wangjiang/Public/software/workplace/demo/tuwen/build/intermediates/library_jni/debug/jni, /Users/wangjiang/Public/software/workplace/demo/common/box2d/build/intermediates/library_jni/debug/jni]
------
external project native libs:
[/Users/wangjiang/.gradle/caches/transforms-3/e1e62cbfcd84bfede405cbb308e1bac9/transformed/jetified-weibo-7.7.0-btool-r3/jni, /Users/wangjiang/.gradle/caches/transforms-3/15985ccd1ee3aaa552f8eb8a95712cc3/transformed/jetified-android-database-sqlcipher-3.5.9/jni]
```

上面输出的信息可以看见 so 文件列表，但是信息比较粗糙，可以再精细一下：

```
project.afterEvaluate {
    project.android.applicationVariants.all { variant ->
        //获取构建变体的名称
        logger.lifecycle("${variant.name}")
        def variantName = variant.name
        def name = String.valueOf(variantName.charAt(0)).toUpperCase() + variantName.substring(1)
        def mergeNativeLibsTask = project.tasks.findByName("merge${name}NativeLibs")
        if(mergeNativeLibsTask!=null) {
            mergeNativeLibsTask.doLast { task ->
                //当前项目相关的 so 文件列表
                printProjectSoInfo("project native libs:", false, task.projectNativeLibs.getFiles())
                //子项目相关的 so 文件列表
                printProjectSoInfo("sub project native libs:", false, task.subProjectNativeLibs.getFiles())
                //三方库相关的 so 文件列表
                printProjectSoInfo("external project native libs:", true, task.externalLibNativeLibs.getFiles())
            }
        }
    }
}

class NativeLibInfo {
    String libName //项目或三方库名称
    List<String> soRelativePathList = new ArrayList<>() //相对项目或三方库的 so 路径

    @Override
    String toString() {
        return "{" + libName + ":" + soRelativePathList + "}"
    }
}

def printProjectSoInfo(String name, boolean isExternal, Set<File> fileSet) {
    logger.lifecycle(name)
    def projectNames = new HashSet<String>()
    def nativeLibsInfoList = new ArrayList<NativeLibInfo>()
    def rootProjectPath = project.rootProject.projectDir.path
    def buildName = project.rootProject.buildDir.name
    fileSet.forEach { file ->
        def projectName
        if (!isExternal) {
            projectName = file.path.substring(rootProjectPath.length() + 1, file.path.indexOf(buildName) - 1)
        } else {
            projectName = file.parentFile.name
        }
        projectNames.add(projectName)
        def childFiles = file.listFiles().toList()
        def nativeLibInfo = new NativeLibInfo()
        nativeLibInfo.libName = projectName
        while (childFiles.size() > 0) {
            def childFile = childFiles.remove(0)
            if (childFile.isDirectory()) {
                childFiles.addAll(childFile.listFiles())
            } else {
                nativeLibInfo.soRelativePathList.add(childFile.path.substring(file.path.length() + 1))
            }
        }
        nativeLibsInfoList.add(nativeLibInfo)
    }
    logger.lifecycle("${projectNames}")
    logger.lifecycle("${nativeLibsInfoList}")
    logger.lifecycle("------")
}
```

再执行 `./gradlew :app:mergeDebugNativeLibs`：

```
> Task :app:mergeDebugNativeLibs
project native libs:
[]
[]
------
sub project native libs:
[tuwen, common/box2d]
[{tuwen:[armeabi-v7a/libBBStudioAudioSpeed.so, armeabi-v7a/libTuwenAudio.so, arm64-v8a/libBBStudioAudioSpeed.so, arm64-v8a/libTuwenAudio.so]}, {common/box2d:[armeabi-v7a/libMBox2d.so, armeabi-v7a/libbox2d.so, arm64-v8a/libMBox2d.so, arm64-v8a/libbox2d.so]}]
------
external project native libs:
[weibo-7.7.0-btool-r3,android-database-sqlcipher-3.5.9]
[{weibo-7.7.0-btool-r3:[armeabi-v7a/libweibosdkcore.so, armeabi-v7a/libwind.so, arm64-v8a/libweibosdkcore.so, arm64-v8a/libwind.so, armeabi/libweibosdkcore.so, armeabi/libwind.so]}, {android-database-sqlcipher-3.5.9:[armeabi-v7a/libsqlcipher.so, x86/libsqlcipher.so, arm64-v8a/libsqlcipher.so, armeabi/libsqlcipher.so, x86_64/libsqlcipher.so]}]
```

这时就可以看到哪个 so 被哪个项目或三方库依赖。当然，也可以自定义其它输出方式，比如输出一个例出项目 so 依赖信息的 html 报告文档。

## 总结

不管是接触一个新项目，还是一直开发老项目，使用 Gradle 命令，可以对项目构建信息有一个快速的掌握。要分析项目 aar 或 jar 依赖信息，可以使用 `Task :app:dependencies`或`./gradlew :app:dependencyInsight --configuration someConf --dependency someDep `；要分析项目 so 依赖信息，可以监听 ` Task :app:mergeDebugNativeLibs`。依赖信息对于解决依赖冲突，库升级问题非常有帮助。另外，一些常用的构建选项如： `--profile, --info, --refresh-dependencies, --no-build-cache, --offline`等，也有助于节约构建时间，或排查构建问题。总之，了解或掌握一些 Gradle 命令，对于开发来说，是很有帮助的。
