---
layout: post
title: Android 静态代码检查
categories: [Android]
description: 在版本发布前尽量去发现代码质量问题，避免带到线上（被动反馈），可以在构建过程之前中去添加 静态代码检查环节，让每一次的构建都能自动地去分析代码是否存在质量问题。
keywords: Android, Gradle
mermaid: false
sequence: false
flow: false
mathjax: false
mindmap: false
mindmap2: false
---

## 背景
随着项目的不断迭代，以及代码的增加和开发人员的增加，代码规范或代码质量的把控，是当前版本发布前必要的一环。在当前开发流程中：编码&rarr;构建&rarr;测试&rarr;发布，代码规范或代码质量相关问题，只能靠人工 Review，或灰度和线上 Bugly 反馈。人工 Review 代码，可能比较费时以及遗漏部分Case，而灰度和线上 Bugly 反馈，为时已晚。所以，要在版本发布前尽量去发现代码质量问题，避免带到线上（被动反馈），可以在构建过程之前中去添加 **静态代码检查**环节，让每一次的构建都能自动地去分析代码是否存在质量问题。

## 项目当前代码质量问题例子
下面简单例举项目中的几个代码质量问题。

#### 重复类问题
项目中存在代码复制和粘贴的情况，例如：
类 `common/player/src/main/java/com/xxx/xxx/player/ijk/util/ArrayUtils.java` 和 类 `template/src/main/java/com/xxx/xxx/module/templatelist/ijk/util/ArrayUtils.java` ，类 `common/player/src/main/java/com/xxx/xxx/player/ijk/util/StringUtils.java`和 `template/src/main/java/com/xxx/xxx/module/templatelist/ijk/util/StringUtils.java`，类`common/player/src/main/java/com/xxx/xxx//ijk/util/CpuInfo.java`和 类`template/src/main/java/com/xxx/xxx/module/templatelist/ijk/util/CpuInfo.java`中的代码几乎完全一样。

#### Java 代码问题
1. 容易报控制针异常问题，例如：
`mainvideoeditor/src/xxx/java/com/xxx/xxx/module/editor/speed/adapter/SpeedRvAdapter.java` 类中的87行：

```
    public void reset() {
        if (mDataList == null && mDataList.isEmpty()) {
            return;
        }
        for (int i = 0; i < mDataList.size(); ++i) {
            mDataList.get(i).selected = false;
        }
        notifyDataSetChanged();
    }
```
2. 线程不安全问题，例如：

类`mainvideoeditor/src/xxx/java/com/xxx/xxx/module/filter/datacenter/VisualEffectsItemHelper.java` 中的29行：

```
public class VisualEffectsItemHelper {
    public static List<VisualEffectItem> VISUAL_EFFECTS_LIST;
    //...省略
    public static List<VisualEffectItem> getVisualEffectsList() {
        if (VISUAL_EFFECTS_LIST == null) {
            VISUAL_EFFECTS_LIST = new ArrayList<>(FX_COUNT);
            VISUAL_EFFECTS_LIST.add(new VisualEffectItem(
                    FX_RANK_BRIGHTNESS,
                    R.string.video_editor_visual_effect_fx_name_brightness,
                    R.drawable.ic_editor_visual_effect_fx_brightness,
                    EditFxVisualCustomFilterEffectHelper.get(EditVisualEffectsConst.FX_TYPE_BRIGHTNESS))
            );
         //...省略
     }
```
3. I/O流没有关闭问题，例如：
类`mainvideoeditor/src/xxx/java/com/xxx/xxx/module/humanaction/mediacodec/utils/GlUtil.java`中的234行：

```
  public static String readTextFromRawResource(final Context applicationContext,
                                                 @RawRes final int resourceId) {
        final InputStream inputStream =
                applicationContext.getResources().openRawResource(resourceId);
        final InputStreamReader inputStreamReader = new InputStreamReader(inputStream);
        final BufferedReader bufferedReader = new BufferedReader(inputStreamReader);
        String nextLine;
        final StringBuilder body = new StringBuilder();
        try {
            while ((nextLine = bufferedReader.readLine()) != null) {
                body.append(nextLine);
                body.append('\n');
            }
        } catch (IOException e) {
            return null;
        }
        return body.toString();
    }
```
#### Kotlin 代码问题
1. 声明异常却没有捕获，引发运行崩溃问题，例如：
类`material/src/main/java/com/xxx/xxx/module/lol/LoLRemoteFragment.kt` 中的97行：

```
    override fun onAttach(context: Context) {
        super.onAttach(context)
        mListener = if (context is OnFragmentInteractionListener) {
            context
        } else {
            throw RuntimeException("$context must implement OnFragmentInteractionListener")
        }
    }
```
2. 容易报空指针异常，例如：
类`material/src/main/java/com/xxx/xxx/module/material/viewmodel/MaterialMusicViewModel.kt`的36行：

```
    fun getMusicTab() {
        viewModelScope.launch {
            val response = try {
                repository.makeMusicTabRequest()
            } catch (e: Exception) {
                null
            }
            //livedata的value不能为空
            musicCategoryLiveData.value = response
        }
    }
```
## 预期收益
根据上面项目中的代码质量问题例子，做构建阶段前的静态代码检查，预期可以得到收益大概分为以下几点：

 1. 防止版本线上出现 Java 和 Kotlin 代码**低端错误**引起崩溃，如空指针，异常未捕获，部分线程，性能等问题；
 2. 防止项目中出现**复制粘贴代码**情况；
 3. 制定代码规范约束，统一开发人员的开发风格，方便代码的阅读和维护；
 4. 可以根据项目当前状况，扩展规则，解决日志，线程，文件，权限等使用相关问题，特别是权限的使用导致发版后在应用市场不能上架问题；
 5. 遏制重复资源，大资源，不合理资源的引入问题。

## 技术方案
根据项目的目前状况，静态代码检查工具需要满足的需求有至少以下几点：

 1. 增量检查，由于项目从未做过静态代码检查，全量检查会影响开发者的正常迭代时间，所以只需对修改或增加文件检查即可；
 2. 重复文件检查，由于项目部分历史原因，项目中存在复制和粘贴的代码，所以需要检查出这些代码，并把这些代码进行抽象或下沉到通用组件，不再有冗余代码；
 3. Java 和 Kotlin 代码检查，**目前急需的是要去发现一些易出错（线上异常引起崩溃），性能，安全性的代码，其它的代码风格，代码设计，易用性等可以后续再检查**；
 4. 资源文件检查，drawable, layout, assets等资源文件，当前可以不用去检查，但是需要支持制定相关规则（检查文件大小，无用资源等），方便后续进行检查；
 5. 自定义规则检查，除了通用的规则检查外，可以自定义一些适合当前项目状况的规则去做检查，如日志，线程，文件，权限使用规范等。

### 技术调研
目前对于Android项目来说，官方有提供  [lint](https://developer.android.google.cn/studio/write/lint?hl=zh-cn) 静态代码检查工具，它同时支持Java 和 Kotlin 代码检查。其它比较成熟的针对 Java 代码的检查工具有：[FindBugs](http://findbugs.sourceforge.net/), [PMD](https://pmd.sourceforge.io/pmd-6.36.0/index.html), [CheckStyle](https://checkstyle.org/checks.html)，[errorprone](https://errorprone.info/)；针对 Kotlin 代码的检查工具有：[ktlint](https://ktlint.github.io/) , [detekt](https://detekt.dev/docs/intro/)。由于项目当前是同时使用 Java 和 Kotlin 开发，所以需要综合来使用这些工具。

检查工具对比：

<table>
    <tr>
        <th>
            名称
        </th>
        <th>
            简要
        </th>
        <th>
            原理
        </th>
        <th>
            Gradle
        </th>
        <th>
            增量
        </th>
        <th>
            自定义
        </th>
    </tr>
    <tr>
        <td>
            lint
        </td>
        <td>
            lint 工具可以检查您的 Android 项目源文件是否有潜在的 bug，以及在正确性、安全性、性能、易用性、无障碍性和国际化方面是否需要优化改进。
        </td>
        <td>
            最新版本通过UAST来分析代码，以前的版本使用过AST和PSI来分析代码
        </td>
        <td>
            支持
        </td>
        <td>
            不支持（可Hook）
        </td>
        <td>
            支持
        </td>
    </tr>
    <tr>
        <td>
            FindBugs
        </td>
        <td>
            Java 代码检查工具，运行的是 Java 字节码，而不是源码。
        </td>
        <td>
            将字节码与一组缺陷模式进行对比来发现可能存在的问题。这些问题包括空指针引用、无限递归循环、死锁等。
        </td>
        <td>
            支持
        </td>
        <td>
            支持
        </td>
        <td>
            支持
        </td>
    </tr>
    <tr>
        <td>
            PMD
        </td>
        <td>
            Java 代码检查工具，PMD可以发现常见的编程缺陷，如未使用的变量、空捕获块、不必要的对象创建等。它主要关注Java和Apex，但支持其他六种语言。
        </td>
        <td>
            使用AST来分析代码
        </td>
        <td>
            支持
        </td>
        <td>
            支持
        </td>
        <td>
            支持
        </td>
    </tr>
    <tr>
        <td>
            CheckStyle
        </td>
        <td>
            Java 代码检查工具，通过检查对代码编码格式，命名约定，Javadoc，类设计等方面进行代码规范和风格的检查，从而有效约束开发人员更好地遵循代码编写规范。
        </td>
        <td>
            使用AST来分析代码
        </td>
        <td>
            支持
        </td>
        <td>
            支持
        </td>
        <td>
            支持
        </td>
    </tr>
    <tr>
        <td>
            errorprone
        </td>
        <td>
            Java 代码检查工具，在编译期间查找代码缺陷的代码检查工具 ，hook正常的build过程，错误产生时及时告知，提供修复建议，并允许基于这些修复建议制定相应模型。
        </td>
        <td>
            使用AST来分析代码
        </td>
        <td>
            支持
        </td>
        <td>
            不支持
        </td>
        <td>
            支持
        </td>
    </tr>
    <tr>
        <td>
            ktlint
        </td>
        <td>
            Kotlin代码检查工具，Kotlin 代码风格检查，遵循的 Android Kotlin 官方代码风格，可以自动格式化代码。
        </td>
        <td>
            使用Kotlin AST来分析代码
        </td>
        <td>
            支持
        </td>
        <td>
            支持
        </td>
        <td>
            支持
        </td>
    </tr>
    <tr>
        <td>
            detekt
        </td>
        <td>
            Kotlin代码检查工具，高度可配置的规则集，支持不同的报告格式，检查风格和潜在性风险。
        </td>
        <td>
            使用Kotlin AST来分析代码
        </td>
        <td>
            支持
        </td>
        <td>
            支持
        </td>
        <td>
            支持
        </td>
    </tr>
</table>

这里，对于重复类检查，主要会使用和 PMD 同一个开发者的 [CPD](https://pmd.sourceforge.io/pmd-6.36.0/pmd_userdocs_cpd.html)。

以上工具，根据我们项目当前现状，目前不适合接入工具有：

 - FindBugs ，由于是字节码检查，自定义规则成本高，且年久失修
 - CheckStyle，只是Java代码的规范和风格检查
 - errorprone，编译期间检查，且自定义规则成本高
 - ktlint，只是 Kotlin 代码的规范和风格检查


剩余工具相关内置规范：

 - lint：正确性、安全性、性能、易用性、无障碍性和国际化；
 - PMD：最佳做法、代码样式、设计、文档、容易出错、多线程、性能、安全、其他规则集；
 - CPD ：Java 和 Kotlin 重复代码；
 - detekt：代码注释和文档、复杂代码、协程、空代码块、异常、格式化代码、命名规范、性能，潜在性bug、风格。

**根据项目当前的情况，第一期只检查正确性，安全性，性能，多线程，容易出错，协程，异常，潜在性bug问题。等问题收敛后，再进一步检查易用性，设计，复杂代码等，以及后续自定义规则检查**。

### 技术实施
#### 总体流程
在静态代码检查初期，最主要的是自动化检查，且不影响常规开发。所以把检查放在CI 构建阶段之前的分析阶段中，在分析阶段去创建一个静态代码检查任务，每当代码 push 到 Gitlab，就去触发这个任务。下面为总体流程：
![在这里插入图片描述]({{site.url}}/images/posts/2022-09-08-Android-Code-Analyze/p1.jpg)


#### 技术细节
首先，根据项目的上下文环境，去集成每一个工具。关于重复代码检查，检查的是项目中的所有Java 和 Kotlin 文件（全量检查），所以要把 CPD 这个工具独立集成和触发。关于 Java 和 Kotlin 代码增量（新增或修改）检查， 需要把得到的增量文件，分发给 lint、PMD、detekt 工具去检查。 另外，lint 工具 不支持增量检查，还需要开发适配 lint 增量检查的插件。

下面为各个工具接入的简单概况，不介绍细节。

##### CPD 重复代码检查
根据 [CPD](https://pmd.sourceforge.io/pmd-6.36.0/pmd_userdocs_cpd.html) 接入文档，以及它的 CLI 。新建 `cpd.gradle` 文件（放在`.codequality`目录下，后面的都一样），编写相关脚本（这里只做核心介绍）：

```bash
    //将收集到项目 java 和 kotlin 文件列表分别交给 cpd

     runcpd(javalist, 340, 'java') { String output ->
            reportLines.addAll(output.readLines())
        }
     runcpd(kotlinlist, 210, 'kotlin') { String output ->
            reportLines.addAll(output.readLines())
        }

    void runcpd(File filelist, int tokens, String language, Closure closure) {
        def out = new ByteArrayOutputStream(1024)
        project.javaexec { spec ->
            spec.main = "net.sourceforge.pmd.cpd.CPD"
            spec.maxHeapSize = '2g'
            spec.classpath = this.classpath.get()
            spec.args '--minimum-tokens', tokens == 0 ? '200' : tokens
            spec.args '--language', language ?: 'java'
            spec.args '--format csv --failOnViolation false'.tokenize()
            spec.args '--filelist', filelist.path
            spec.args '--skip-lexical-errors', '--ignore-annotations'
            spec.standardOutput = out
            spec.ignoreExitValue = true
        }
        closure.call(out.toString("UTF-8"))
        logger.lifecycle("CPD run for $filelist")
    }
```
 使用 cpd 需要的 依赖有：
 

```bash
 'net.sourceforge.pmd:pmd-core:6.36.0'
 'net.sourceforge.pmd:pmd-java:6.36.0'
 'net.sourceforge.pmd:pmd-kotlin:6.10.0'
```
可以在 它的 github 上去下载最新版本：[https://github.com/pmd/pmd/releases/tag/pmd_releases/6.49.0](https://github.com/pmd/pmd/releases/tag/pmd_releases/6.49.0)。需要把这些添加到 `gradle` 的 `dependencies` 中，然后才能使用 `project.javaexec` 去执行。

##### PMD Java 代码检查
根据 PMD 的  [gradle](https://pmd.sourceforge.io/pmd-6.36.0/pmd_userdocs_tools_gradle.html) 接入文档，再结合项目环境，新建 `pmd.gradle` 文件（放在`.codequality`目录下）：

```bash
/*
 * des : inspect java code
 * doc : https://pmd.github.io/latest/pmd_userdocs_tools_gradle.html
 * use : should apply this script in root build.gradle
 * how to analyze result : https://pmd.github.io/latest/pmd_userdocs_report_formats.html , see html example.
 */
apply plugin: 'pmd'

def pmdConfigPath = new File(codequality, 'scripts/pmd').path

task pmd(type: Pmd) {
    FileCollection fileCollection
    //获取到 git diff 后的 java 文件
    if (project.hasProperty("commitJavaFiles")) {
        fileCollection = project.files(project.property("commitJavaFiles"))
    } else {
       //当前项目下的 src 目录下的所有文件，注意这里可能有 flavor
        fileCollection = project.files("src")
    }
    if (fileCollection == null) {
        return
    }
    //区分是 CI 环境，还是本地环境
    def isCiServer = false
    if (project.hasProperty("isCiServer")) {
        isCiServer = project.property("isCiServer")
    }
    if (isCiServer) {
        ignoreFailures = false
    } else {
        ignoreFailures = true
    }
    consoleOutput = true
    //规则集
    ruleSetFiles = files("${pmdConfigPath}/pmd-ruleset.xml")
    ruleSets = []
    source fileCollection
    include '**/*.java'
    exclude '**/gen/**'

    reports {
        xml.getRequired().set(false)
        html.getRequired().set(true)
        //结果输出到项目根目录下的 build/reports/pmd 文件夹中，只需要html格式的结果文档
        def destination = new File(new File("${project.rootProject.buildDir.path}/reports/pmd"), "${project.name}-pmd.html")
        html.destination(destination)
    }
}
```
[规则集](https://pmd.github.io/latest/pmd_userdocs_configuring_rules.html)的配置，在同目录下创建 `scripts/pmd/pmd-ruleset.xml`：

```bash
<?xml version="1.0"?>
<ruleset xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" name="Custom Rules"
    xmlns="http://pmd.sourceforge.net/ruleset/2.0.0"
    xsi:schemaLocation="http://pmd.sourceforge.net/ruleset/2.0.0 https://pmd.sourceforge.io/ruleset_2_0_0.xsd">

    <description>
        Custom ruleset for Android application
    </description>

    <!-- rule des :https://pmd.github.io/pmd-6.41.0/pmd_rules_java.html -->
    <exclude-pattern>.*/R.java</exclude-pattern>
    <exclude-pattern>.*/gen/.*</exclude-pattern>
    <!-- Your rules will come here -->
    <!--     容易出现运行时错误 -->
    <rule ref="category/java/errorprone.xml">
        <exclude name="BeanMembersShouldSerialize" />
    </rule>
    <!--    多线程时问题-->
    <rule ref="category/java/multithreading.xml" />
    <!--    需要优化代码的地方-->
    <rule ref="category/java/performance.xml" />
    <!--    安全问题-->
    <rule ref="category/java/security.xml" />
</ruleset>
```
这里前期只配置：容易出现运行时错误，多线程时问题，需要优化代码的地方，安全问题规则。

##### detekt Kotlin 代码检查
根据 detekt 的 [gradle](https://detekt.dev/docs/gettingstarted/gradle) 接入文档，首先在项目根目录下配置 `classpath`：

```bash
buildscript {
    repositories {
        gradlePluginPortal()
    }
    dependencies {
        classpath "io.gitlab.arturbosch.detekt:detekt-gradle-plugin:1.21.0"
    }
}
```

然后新建 `detekt.grale` 文件：

```bash
/*
 * des : inspect kotlin code
 * doc : https://github.com/detekt/detekt
 * use : should apply this script in root build.gradle
 * how to analyze result : https://detekt.dev/docs/introduction/reporting , see html example.
 */
apply plugin: 'io.gitlab.arturbosch.detekt'

def detektConfigPath = new File(codequality, 'scripts/detekt').path

detekt {
    //获取到 git diff 后的 kotlin 文件
    if (project.hasProperty("commitKotlinFiles")) {
        source = project.files(project.property("commitKotlinFiles"))
    }
    buildUponDefaultConfig = true // preconfigure defaults
    //规则集
    config = files("$detektConfigPath/detekt.yml")
    // point to your custom config defining rules to run, overwriting default behavior
    baseline = file("$detektConfigPath/baseline.xml")
    basePath = projectDir.path
    def isCiServer = false
    //区分是 CI 环境，还是本地环境
    if (project.hasProperty("isCiServer")) {
        isCiServer = project.property("isCiServer")
    }
    if (isCiServer) {
        ignoreFailures = false
    } else {
        ignoreFailures = true
    }
}

tasks.detekt.jvmTarget = "1.8"

tasks.named("detekt").configure {
    reports {
        // Enable/Disable HTML report (default: true)
        def destination = new File(new File("${project.rootProject.buildDir.path}/reports/detekt"), "${project.name}-detekt.html")
        html.required.set(true)
        html.outputLocation.set(destination)
    }
}
```
[规则集](https://detekt.dev/docs/introduction/configurations)的配置，在同目录下创建 `scripts/detekt/detekt.xml`：

```bash
#whiteProjectListbuild:
#  maxIssues: 0
#  excludeCorrectable: false
#  weights:
  # complexity: 2
  # LongParameterList: 1
  # style: 1
  # comments: 1

config:
  validation: true
  warningsAsErrors: false
#  checkExhaustiveness: false
  # when writing own rules with new properties, exclude the property path e.g.: 'my_rule_set,.*>.*>[my_property]'
  excludes: ''

processors:
  active: true
  exclude:
    - 'DetektProgressListener'
  # - 'KtFileCountProcessor'
  # - 'PackageCountProcessor'
  # - 'ClassCountProcessor'
  # - 'FunctionCountProcessor'
  # - 'PropertyCountProcessor'
  # - 'ProjectComplexityProcessor'
  # - 'ProjectCognitiveComplexityProcessor'
  # - 'ProjectLLOCProcessor'
  # - 'ProjectCLOCProcessor'
  # - 'ProjectLOCProcessor'
  # - 'ProjectSLOCProcessor'
  # - 'LicenseHeaderLoaderExtension'

console-reports:
  active: true
  exclude:
    - 'ProjectStatisticsReport'
    - 'ComplexityReport'
    - 'NotificationReport'
    - 'FindingsReport'
    - 'FileBasedFindingsReport'
  #  - 'LiteFindingsReport'

output-reports:
  active: true
  exclude:
  # - 'TxtOutputReport'
  # - 'XmlOutputReport'
  # - 'HtmlOutputReport'
  # - 'MdOutputReport'
#协程问题
coroutines:
  active: true
  GlobalCoroutineUsage:
    active: false
  InjectDispatcher:
    active: true
    dispatcherNames:
      - 'IO'
      - 'Default'
      - 'Unconfined'
  RedundantSuspendModifier:
    active: true
  SleepInsteadOfDelay:
    active: true
  SuspendFunWithCoroutineScopeReceiver:
    active: false
  SuspendFunWithFlowReturnType:
    active: true
#空代码块问题
empty-blocks:
  active: true
  EmptyCatchBlock:
    active: true
    allowedExceptionNameRegex: '_|(ignore|expected).*'
  EmptyClassBlock:
    active: true
  EmptyDefaultConstructor:
    active: true
  EmptyDoWhileBlock:
    active: true
  EmptyElseBlock:
    active: true
  EmptyFinallyBlock:
    active: true
  EmptyForBlock:
    active: true
  EmptyFunctionBlock:
    active: true
    ignoreOverridden: true
  EmptyIfBlock:
    active: true
  EmptyInitBlock:
    active: true
  EmptyKtFile:
    active: true
  EmptySecondaryConstructor:
    active: true
  EmptyTryBlock:
    active: true
  EmptyWhenBlock:
    active: true
  EmptyWhileBlock:
    active: true
#异常问题
exceptions:
  active: true
  ExceptionRaisedInUnexpectedLocation:
    active: true
    methodNames:
      - 'equals'
      - 'finalize'
      - 'hashCode'
      - 'toString'
  InstanceOfCheckForException:
    active: true
    excludes: ['**/test/**', '**/androidTest/**', '**/commonTest/**', '**/jvmTest/**', '**/jsTest/**', '**/iosTest/**']
  NotImplementedDeclaration:
    active: false
  ObjectExtendsThrowable:
    active: false
  PrintStackTrace:
    active: true
  RethrowCaughtException:
    active: true
  ReturnFromFinally:
    active: true
    ignoreLabeled: false
  SwallowedException:
    active: true
    ignoredExceptionTypes:
      - 'InterruptedException'
      - 'MalformedURLException'
      - 'NumberFormatException'
      - 'ParseException'
    allowedExceptionNameRegex: '_|(ignore|expected).*'
  ThrowingExceptionFromFinally:
    active: true
  ThrowingExceptionInMain:
    active: false
  ThrowingExceptionsWithoutMessageOrCause:
    active: true
    excludes: ['**/test/**', '**/androidTest/**', '**/commonTest/**', '**/jvmTest/**', '**/jsTest/**', '**/iosTest/**']
    exceptions:
      - 'ArrayIndexOutOfBoundsException'
      - 'Exception'
      - 'IllegalArgumentException'
      - 'IllegalMonitorStateException'
      - 'IllegalStateException'
      - 'IndexOutOfBoundsException'
      - 'NullPointerException'
      - 'RuntimeException'
      - 'Throwable'
  ThrowingNewInstanceOfSameException:
    active: true
  TooGenericExceptionCaught:
    active: true
    excludes: ['**/test/**', '**/androidTest/**', '**/commonTest/**', '**/jvmTest/**', '**/jsTest/**', '**/iosTest/**']
    exceptionNames:
      - 'ArrayIndexOutOfBoundsException'
      - 'Error'
      - 'Exception'
      - 'IllegalMonitorStateException'
      - 'IndexOutOfBoundsException'
      - 'NullPointerException'
      - 'RuntimeException'
      - 'Throwable'
    allowedExceptionNameRegex: '_|(ignore|expected).*'
  TooGenericExceptionThrown:
    active: true
    exceptionNames:
      - 'Error'
      - 'Exception'
      - 'RuntimeException'
      - 'Throwable'
#性能问题
performance:
  active: true
  ArrayPrimitive:
    active: true
  CouldBeSequence:
    active: false
    threshold: 3
  ForEachOnRange:
    active: true
    excludes: ['**/test/**', '**/androidTest/**', '**/commonTest/**', '**/jvmTest/**', '**/jsTest/**', '**/iosTest/**']
  SpreadOperator:
    active: true
    excludes: ['**/test/**', '**/androidTest/**', '**/commonTest/**', '**/jvmTest/**', '**/jsTest/**', '**/iosTest/**']
  UnnecessaryTemporaryInstantiation:
    active: true
#潜在bug问题
potential-bugs:
  active: true
  AvoidReferentialEquality:
    active: true
    forbiddenTypePatterns:
      - 'kotlin.String'
  CastToNullableType:
    active: false
  Deprecation:
    active: false
  DontDowncastCollectionTypes:
    active: false
  DoubleMutabilityForCollection:
    active: true
    mutableTypes:
      - 'kotlin.collections.MutableList'
      - 'kotlin.collections.MutableMap'
      - 'kotlin.collections.MutableSet'
      - 'java.util.ArrayList'
      - 'java.util.LinkedHashSet'
      - 'java.util.HashSet'
      - 'java.util.LinkedHashMap'
      - 'java.util.HashMap'
  DuplicateCaseInWhenExpression:
    active: true
  ElseCaseInsteadOfExhaustiveWhen:
    active: false
  EqualsAlwaysReturnsTrueOrFalse:
    active: true
  EqualsWithHashCodeExist:
    active: true
  ExitOutsideMain:
    active: false
  ExplicitGarbageCollectionCall:
    active: true
  HasPlatformType:
    active: true
  IgnoredReturnValue:
    active: true
    restrictToAnnotatedMethods: true
    returnValueAnnotations:
      - '*.CheckResult'
      - '*.CheckReturnValue'
    ignoreReturnValueAnnotations:
      - '*.CanIgnoreReturnValue'
    ignoreFunctionCall: []
  ImplicitDefaultLocale:
    active: true
  ImplicitUnitReturnType:
    active: false
    allowExplicitReturnType: true
  InvalidRange:
    active: true
  IteratorHasNextCallsNextMethod:
    active: true
  IteratorNotThrowingNoSuchElementException:
    active: true
  LateinitUsage:
    active: false
    excludes: ['**/test/**', '**/androidTest/**', '**/commonTest/**', '**/jvmTest/**', '**/jsTest/**', '**/iosTest/**']
    ignoreOnClassesPattern: ''
  MapGetWithNotNullAssertionOperator:
    active: true
  MissingPackageDeclaration:
    active: false
    excludes: ['**/*.kts']
  MissingWhenCase:
    active: true
    allowElseExpression: true
  NullCheckOnMutableProperty:
    active: false
  NullableToStringCall:
    active: false
  RedundantElseInWhen:
    active: true
  UnconditionalJumpStatementInLoop:
    active: false
  UnnecessaryNotNullOperator:
    active: true
  UnnecessarySafeCall:
    active: true
  UnreachableCatchBlock:
    active: true
  UnreachableCode:
    active: true
  UnsafeCallOnNullableType:
    active: true
    excludes: ['**/test/**', '**/androidTest/**', '**/commonTest/**', '**/jvmTest/**', '**/jsTest/**', '**/iosTest/**']
  UnsafeCast:
    active: true
  UnusedUnaryOperator:
    active: true
  UselessPostfixExpression:
    active: true
  WrongEqualsTypeParameter:
    active: true
```

这里前期只配置：协程问题，空代码问题，异常问题，性能问题，潜在bug问题规则。

##### lint Java 和 Kotlin 代码检查
根据 lint 的 [gradle](https://developer.android.google.cn/studio/write/lint?hl=zh-cn) 接入文档，新建 `lint.grale` 文件：

```bash
/*
 * des : inspect java and kotlin code by increment
 * doc :  https://juejin.cn/post/7093133800812576781#heading-8
 * use : should apply this script in root build.gradle
 * custom lint plugin : https://github.com/googlesamples/android-custom-lint-rules
 * */
project.afterEvaluate {

    if (!(project.getPlugins().hasPlugin('com.android.application') ||
            project.getPlugins().hasPlugin('com.android.library'))) {
        return
    }
    //自定义的 lint 增量插件
    project.apply plugin: 'com.xxx.xxx.lint'

    def lintConfigPath = new File(codequality, 'scripts/lint').path
    project.android {
        //doc : https://developer.android.google.cn/reference/tools/gradle-api/7.1/com/android/build/api/dsl/LintOptions?hl=en
        lint {
            // true:关闭lint报告的分析进度
            quiet true
            // true:错误发生后停止gradle构建
            abortOnError false
            // true:只报告error
            ignoreWarnings true
            // true:忽略有错误的文件的全/绝对路径(默认是true)
            absolutePaths true
            // true:检查所有问题点，包含其他默认关闭项
            checkAllWarnings false
            // true:所有warning当做error
            warningsAsErrors false
            // 关闭指定问题检查
            disable 'Usability', 'Accessibility', 'Internationalization','ResourceName'
            // true:error输出文件不包含源码行号
            noLines false
            // true：显示错误的所有发生位置，不截取
            showAll true
            // 回退lint设置(默认规则)
            lintConfig file("$lintConfigPath/default-lint.xml")
            // true：生成txt格式报告(默认false)
            textReport true
            // true：生成XML格式报告
            xmlReport false
            // true：生成HTML报告(带问题解释，源码位置，等)
            htmlReport true
            // html报告可选路径(构建器默认是lint-results.html )
            def destination = new File(new File("${project.rootProject.buildDir.path}/reports/lint"), "${project.name}-lint.html")
            htmlOutput file(destination)
            //  true：所有正式版构建执行规则生成崩溃的lint检查，如果有崩溃问题将停止构建
            checkReleaseBuilds true
            checkDependencies false
        }
    }
}
```
由于 lint 不支持增量文件检查，这里需要 hook 原 lint plugin 中添加检查文件的地方，hook 原理可以查看[AGP7.0增量Lint适配](https://juejin.cn/post/7093133800812576781)，这里不具体介绍。当拦截到 lint 插件中 Main 类的 run 方法时，通过 LintRequest 获取 project，然后将添加增量文件，此时 lint 就只会检查增量文件，不会检查项目所有文件。那么，为了适配 lint 增量检查需要做的就是，在运行 lint task 之前，将 git diff 出的文件列表，存储到一个本地文件中，然后运行 lint task 的时候，去读取这个文件中的文件列表，然后添加到 project 检查文件列表中，这样就实现了 lint 增量检查。这里不再展开细节。

##### 统一触发检查工具
接入完 CPD，PMD，detekt，lint 工具后，就是这么去做增量检查（版本迭代，新增或修改文件检查），这里主要结合 `git log` [命令](https://git-scm.com/docs/git-log/zh_HANS-CN)。再新建`code_inspect_increment.gradle`文件（还是放在`.codequality`目录下）

```bash
ext {
    //存储增量的Java文件，给pmd使用
    commitJavaFiles = new ArrayList<String>()
    //存储增量的kotlin文件，给detekt使用
    commitKotlinFiles = new ArrayList<String>()
    commitAuthor = ""
    appVersionName = ""
}
/**
 * 增量开关
 */
def isIncrement = false
/**
 * 是否是CI环境
 * @return true是，否则不是
 */
def isCiServer() {
    def isCiServer = false
    if (project.hasProperty("isCiServer")) {
        isCiServer = project.property("isCiServer")
    }
    return isCiServer
}
/**
 * 通过cmd命令获取结果值
 * @return string 结果值
 */
def getResultFromCmd(String cmd) {
    logger.lifecycle("${project.name} cmd:$cmd")
    def out = new ByteArrayOutputStream()
    if (System.properties['os.name'].toLowerCase().contains('windows')) {
        project.exec {
            ExecSpec execSpec ->
                executable 'cmd'
                args '/c', "$cmd"
                standardOutput = out
        }.assertNormalExitValue()
    } else {
        project.exec {
            ExecSpec execSpec ->
                executable 'bash'
                args '-c', "$cmd"
                standardOutput = out
        }.assertNormalExitValue()
    }
    return out.toString().trim()
}
/**
 * 获取提交作者
 * @return 提交作者
 */
def getCommitAuthor() {
    def commitAuthor = ""
    if (isCiServer()) {
        commitAuthor = "${System.env.CI_COMMIT_AUTHOR}"
        commitAuthor = commitAuthor.substring(0, commitAuthor.indexOf('<')).trim()
    } else {
        commitAuthor = getResultFromCmd("git config --get user.name")
    }
    return commitAuthor
}

/**
 * 获取提交 sha，用于 git diff 出增量文件，其它项目仓库，这个方法需要更改实现
 * @return 提交 sha
 */
def getCommitTargetSha() {
    //目前这个仓库只能根据上一个merge节点，进行代码增量检测
    def commitTargetSha = getResultFromCmd("git log --max-count=1 --pretty=%H --merges")
    return commitTargetSha
}
/**
 * 获取提交文件，只包含.java和.kt的文件
 * @return 文件相对路径数组
 */
def getCommitFiles() {
    def commitTargetSha = getCommitTargetSha()
    def commitSha = getResultFromCmd("git log --max-count=1 --pretty=%H")
    def commitAuthor = getCommitAuthor()
    def cmd = "git log --name-only --author=$commitAuthor --pretty=\"format:\" $commitTargetSha..$commitSha"
    def result = getResultFromCmd(cmd)
    def projectStartPath = getProjectStartPath()
    logger.lifecycle("projectStartPath=$projectStartPath")
    if (result != null && result.trim().length() > 0) {
        def files = result.readLines()
                .findAll { it.startsWith("${projectStartPath}") && isFilePathValid(it) }.toArray()
        return files.toUnique()
    }
    return null
}
/**
 * 判断文件路径是否合理
 * @param path 文件路径
 * @return true合理，否则不合理
 */
static def isFilePathValid(String path) {
    return (path.endsWith('.kt') || path.endsWith('.java') || path.endsWith('.xml') || path.endsWith('.png')) && !path.contains('/test/') && !path.contains('/androidTest/') && !path.contains('/generated/')
}
/**
 * 获取src目录下的所有文件，只包含.java和.kt的文件
 * @return 文件相对路径数组
 */
def getAllFiles() {
    def fileCollection = project.files("src")
    if (fileCollection == null) {
        return null
    }
    def files = fileCollection.asFileTree.getFiles()
    def iterator = files.iterator()
    def allFiles = new ArrayList<String>(files.size())
    while (iterator.hasNext()) {
        File file = iterator.next()
        def path = getProjectStartPath() + File.separator + project.relativePath(file.path)
        if (path != null && isFilePathValid(path)) {
            allFiles.add(path)
        }
    }
    return allFiles.toArray(new Object[allFiles.size()])
}
/**
 * 获取项目名称，如gradleengine,framework/asmbase
 * @return 项目名称
 */
def getProjectStartPath() {
    def projectRelativePath = project.relativePath(project.path).replaceAll(':', '/')
    return projectRelativePath.substring(1)
}

task runCodeInspectIncrement {
    group = "codeInspect"
    description = "存储增量文件"
    def commitJavaFiles = new ArrayList<String>()
    def commitKotlinFiles = new ArrayList<String>()

    def commitFiles = new Object[0]
    if (isIncrement) {
        commitFiles = getCommitFiles()
    } else {
        commitFiles = getAllFiles()
    }
    saveCommitFiles(commitFiles)
    if (commitFiles != null && commitFiles.length > 0) {
        def projectStartPath = getProjectStartPath()
        for (String s : commitFiles) {
            def commitFile = s.toString().trim()
            if (commitFile.endsWith('.kt')) {
                commitKotlinFiles.add(commitFile.substring(projectStartPath.length() + 1))
            } else if (commitFile.endsWith('.java')) {
                commitJavaFiles.add(commitFile.substring(projectStartPath.length() + 1))
            }
        }
    }
    if (!commitKotlinFiles.isEmpty()) {
        logger.lifecycle("${project.name} commitKotlinFiles:$commitKotlinFiles")
    }
    if (!commitJavaFiles.isEmpty()) {
        logger.lifecycle("${project.name} commitJavaFiles:$commitJavaFiles")
    }
    project.setProperty("commitKotlinFiles", commitKotlinFiles)
    project.setProperty("commitJavaFiles", commitJavaFiles)
    project.setProperty("commitAuthor", getCommitAuthor())
    project.setProperty("appVersionName", getAppVersionName())
}

/**
 * 存储提交文件
 * @param commitFiles 提交文件
 */
def saveCommitFiles(Object[] commitFiles) {
    //commitFiles 是对象
    def lintBuildDir = new File(project.buildDir, "code_inspect")
    if (!lintBuildDir.exists()) {
        lintBuildDir.mkdirs()
    }
    def changedFile = new File(lintBuildDir, "commit_files.txt")
    if (commitFiles == null || commitFiles.length == 0) {
        //添加一个默认文件，防止Lint全量扫描
        def buildFilePath = "build.gradle"
        commitFiles = new Object[]{buildFilePath}
    }
    def fileOutputStream = new FileOutputStream(changedFile)
    def rootProjectPath = project.rootProject.rootDir.path + File.separator
    //存储绝对路径
    for (int i = 0; i < commitFiles.length; i++) {
        def commitFile = commitFiles[i].toString()
        fileOutputStream.write((rootProjectPath + commitFile).getBytes("utf-8"))
        if (i != commitFiles.length - 1)
            fileOutputStream.write("\n".getBytes("utf-8"))
    }
    fileOutputStream.flush()
    fileOutputStream.close()
    logger.lifecycle("Save Commit Files In: ${changedFile.path} \n$commitFiles")
}


static def getAppVersionName() {
    return "unknown"
}

```
这里需要注意的是 `gradle task :  runCodeInspectIncrement`，这个自定义 的 task，它的作用就是 **通过 git 命令将修改或新增文件存储到 project 的额外变量中，以及存储到 本地文件中（给 lint 插件做增量扫描使用）**。


接下来，就是如何统一触发这些插件。再新建`code_inspect.gradle`文件（还是放在`.codequality`目录下）：

```bash
task runCodeInspect {
    group = "codeInspect"
    description = "静态代码检查，统一触发pmd detekt lint任务"
}

project.afterEvaluate {
    bindTask("lint", "runCodeInspect") {
        checkLintResultValid()
    }
    bindTask("pmd", "runCodeInspect") {
        checkPmdResultValid()
    }
    bindTask("detekt", "runCodeInspect") {
        checkDetektResultValid()
    }
}

/**
 * 绑定任务
 * @param taskName 任务名称
 * @param targetTaskName 被绑定的任务名称
 * @param action taskName指定的任务执行完成后的回调
 */
def bindTask(String taskName, String targetTaskName, groovy.lang.Closure action) {
    def task = project.tasks.findByName(taskName)
    if (task != null) {
        logger.lifecycle("${task} will be run")
        def targetTask = project.tasks.findByName(targetTaskName)
        task.doLast(action)
        targetTask.finalizedBy(task)
    }
}
```
上面的流程大概是，新建 `gradle task : runCodeInspect`，在这个 task 执行完的时候 (**finalizedBy**)，就去触发 pmd， detekt ，lint 相关的 task 去执行静态代码检查。检查完成后，在项目根目录下的 build/reports/pmd 或 build/reports/detekt 或  uild/reports/lint 查看结果 html 报告。

**在这里，html 报告，只是给自己看的一个报告，如果要给大家看或者老板看，那么重要的是怎样将这个分析结果报告数据可视化。这才是这个工具最重要的一环。这里不在展开，感兴趣可以结合自己项目背景，思考如何去做**。

最后，在项目根目录下的 `build.gradle` 中，让每个子项目都添加上面的脚本依赖：

```bash
ext {
    //是不是CI环境
    isCiServer = System.env.containsKey('CI_SERVER')
    //插件工具存放目录
    codequality = new File(projectDir, '.codequality')
    //静态代码检查开关
    isCodeInspectEnable = true
}
//重复代码检查
if (isCodeInspectEnable) {
    def cpdPath = new File(codequality, 'cpd.gradle').path
    apply from: "$cpdPath"
}

allprojects {

    if (isCodeInspectEnable ) {
        def basePath = new File(codequality, 'code_inspect_increment.gradle').path
        apply from: "$basePath"
        def pmdPath = new File(codequality, 'pmd.gradle').path
        apply from: "$pmdPath"
        def detektPath = new File(codequality, 'detekt.gradle').path
        apply from: "$detektPath"
        def lintPath = new File(codequality, 'lint.gradle').path
        apply from: "$lintPath"
        def codeInspectPath = new File(codequality, 'code_inspect.gradle').path
        apply from: "$codeInspectPath"
    }

}
```
重复代码检查，cpd 任务可以独立触发。因为重复代码的检查，期望是先全部解决。

##### CI 集成
在开发者将代码  push 到 Gitlab 后，在 CI 执行中，添加分析阶段，在分析阶段中添加 静态代码检查 任务，当检查出代码质量问题，就中断执行，也就是 `pipeline` 不通过（此时可以禁止合入代码），然后上传检查报告。这样就可以自动化对 修改或新增文件进行检查了。

接下来，在项目的 `.gitlab-ci.yml` 文件中，添加 `stage：analyze` 中，在 `analyze` 中添加 `job: inspect java and kotlin codes`：

```bash
stages:
  - 其它
  - analyze
  - 其它

inspect java and kotlin codes:
  tags:
    - apk
    - android
  stage: analyze
  script:
    - echo "$CI_COMMIT_BEFORE_SHA $CI_COMMIT_SHA"
    - echo "$CI_JOB_STAGE}_reports_${CI_PROJECT_NAME}_${CI_BUILD_REF_NAME}"
    - ./gradlew runCodeInspect
  before_script:
    - source change_java_version
  artifacts:
    name: "$CI_JOB_STAGE}_reports_${CI_PROJECT_NAME}_$CI_COMMIT_REF_SLUG"
    when: on_failure
    expire_in: 1 days
    paths:
      - "build/reports/*"
  only:
    - branches
  except:
    - master
  allow_failure: false
```
这里其实就是运行 `.codequality/code_inspect.gradle`中建立的 gradle task：`./gradlew runCodeInspect`。如果对 Gtilab CI/CD 不熟悉，可以查看上一篇文章：[Gitlab CI/CD 简单介绍](https://blog.csdn.net/wangjiang_qianmo/article/details/122867335)。

## 结语
上面对项目的当前代码质量问题，以及如何去解决这个问题，以及项目的预期收益进行了分析。这次静态代码检查的主要是解决重复代码，Java 和 Kotlin 代码的易出错，线程，安全，性能等优先问题。在静态代码检查接入项目后，项目的执行流程是：开发者 push 代码到 Gitlab ，触发 CI，在 CI 的分析阶段，然后执行 静态代码检查 job，在这个 job 中执行  `gradle task : runCodeInspect` 任务，当任务运行结束，将代码检查报告上传到 Gitlab，开发者下载报告，根据报告建议进行代码更改。

静态代码检查是代码开发完后的一种代码质量防控措施，如果提高代码质量，最主要的还是开发者编写代码的时候，进行全面和详细的思考。




