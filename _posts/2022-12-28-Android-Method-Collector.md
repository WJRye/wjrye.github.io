---
layout: post
title: Android 查看隐私权限方法调用者集合
categories: [Android]
description: 在 App 上架的时候，如果隐私不合规，可以通过查看隐私权限方法调用者来快速定位。
keywords: Android
mermaid: false
sequence: false
flow: false
mathjax: false
mindmap: false
mindmap2: false
---

## 背景

辛辛苦苦迭代完当前版本，准备推送 App 到应用市场上架，却收到拒审通知：**App隐私合规上架护航版检测报告**，

报告内容：

> 场景2:APP以隐私政策弹窗的形式向用户明示收集使用规则，未经用户同意，存在收集设备MAC地址、IMEI等信息的行为。
> 检测结果: 存在问题
> 改进建议: (1)在APP首次运行或用户注册时以弹窗、突出链接等明显方式主动提示用户阅读隐私政策; (2)在用户点击同意隐私政策之前，不应提前收集IMEI、设备MAC地址和软件安装列表等信息，以及不应提前向用户申请手机、通讯、短信等敏感权限。

这真是晴天霹雳！！！此时，老板又让你半天时间解决。那么，该如何去排查这个问题呢？

遇到上面这种问题，第一时间，可能想到解决思路是： 查看 git 日志，找到启动模块的更改。如果是项目本身的更改，那很好办，直接更改代码逻辑即可。如果不是，是引入或升级三方库导致的，那么只能根据获取 IMEI、设备MAC地址和软件安装列表等信息的 API 进行方法搜索，然而在 Android Studio 里面搜索的结果却是：
![](/images/posts/2022-12-28-Android-Method-Collector/p1.png)
这信息对问题排除起不到任何帮助。

## 技术目标

我们期望有个工具，能扫描出启动模块中自身模块代码以及三方依赖库中所有使用 API 获取 获取 IMEI、设备MAC地址和软件安装列表等信息的地方，然后做版本对比，就可以知道新增代码或依赖库哪里有调用隐私方法。类似下面这张结果报告：

![在这里插入图片描述](/images/posts/2022-12-28-Android-Method-Collector/p2.jpeg)

## 技术方案

根据之前一篇文章：[Android 静态代码检查](https://blog.csdn.net/wangjiang_qianmo/article/details/126759343)。知道 lint 可以自定义规则检查，而且 lint 中有提供 **.class 文件的检查**，这样就可以对自身项目代码以及三方库代码进行**目标类方法调用者检查**，并把这个规则集成到 lint 中，与 项目**静态代码检查** 一起工作。

关于 .class 文件的检查，lint 中也是使用了 asm ，所以这个和字节码插桩实现是同样的效果。单集成在 lint 中，技术成本更低。

目标类方法调用者检查：根据配置文件中要检查的类方法（达到动态配置功能，按需检查），检查后找到所有调用该类方法的地方，并输出 Json 和 Html 文件报告，并且还可以输出版本 diff 后的报告。

## 技术实践

### 技术流程

在自定义 lint 中 定义一个 Detector，用于接收 项目和三方库 class 文件扫描。在 Detector 执行过程中，将 class 和 class method 信息收集起来，等执行完成，将收集到的类方法信息与配置文件中的要检查的类方法信息进行匹配，匹配完成后输出 Json 和 Html 格式文件报告，如果有上一个执行结果，将与之 diff 后再输出结果。

下面为流程图：

![](/images/posts/2022-12-28-Android-Method-Collector/p3.png)

### 自定义 lint 规则

google 在 github 上提供了一个自定义 lint 规则的 demo：[https://github.com/googlesamples/android-custom-lint-rules](https://github.com/googlesamples/android-custom-lint-rules)。

这里，新建立一个 MethodCollectorDetector，并让它实现 `Detector.ClassScanner` 接口，主要告诉 lint ，需要的是 class 文件的检查。在 Issue 的 Implementation  中的作用域，需要注意的是：`Scope.CLASS_FILE_SCOPE` 和 `Scope.ALL_CLASSES_AND_LIBRARIES` 的区别，前者作用域只包含项目内的 class 文件，后一个作用域才会包含三方库中的 class  文件。

下面为伪代码：

```
class MethodCollectorDetector : Detector(), Detector.ClassScanner {

    override fun getApplicableAsmNodeTypes(): IntArray {
        //需要的是方法
        return intArrayOf(
            AbstractInsnNode.METHOD_INSN
        )
    }


    override fun checkInstruction(
        context: ClassContext,
        classNode: ClassNode,
        method: MethodNode,
        instruction: AbstractInsnNode
    ) {
        super.checkInstruction(context, classNode, method, instruction)
        //目标所属类名称：instruction.owner
        //目标所属类方法名称：instruction.name
        //调用者所属类名称：classNode.name
        //调用者所属类方法名称：method.name
        //调用者所属类方法行数：ClassContext.findLineNumber(method)

        //上面信息与配置文件中的类方法信息进行匹配，匹配成功，就添加到结果中
        (instruction as? MethodInsnNode)?.let {
            if (match(
                    instruction.owner,
                    instruction.name,
                    classNode.name,
                    method.name
                )
            ) {
                add( instruction.owner,
                    instruction.name,
                    classNode.name,
                    method.name)

                context.report(
                    ISSUE, context.getLocation(instruction),
                    msg
                )
            }
        }
    }


    override fun beforeCheckRootProject(context: Context) {
        super.beforeCheckRootProject(context)
        //在检查根项目之前，读取配置文件中的类方法信息
    }


    override fun afterCheckRootProject(context: Context) {
        super.afterCheckRootProject(context)
        //将收集到的信息分析并生成 json 和 html 报告
    }

    companion object {

        private const val BUILD_NAME = "build"
        private const val BUILD_REPORTS_NAME = "reports"
        private const val METHOD_COLLECTOR_NAME = "method-collector"

        val ISSUE = Issue.create(
            "MethodCollector",  //唯一 ID
            "在目录${BUILD_NAME}/${BUILD_REPORTS_NAME}/${METHOD_COLLECTOR_NAME}下查看相关方法调用者信息",  //简单描述
            "根据配置文件中类方法，找到调用它的地方，包括项目和三方库",  //详细描述
            Category.CORRECTNESS,  //问题种类（正确性、安全性等）
            6, Severity.WARNING,  //问题严重程度（忽略、警告、错误）
            Implementation( //实现，包括处理实例和作用域
                MethodCollectorDetector::class.java,
                Scope.ALL_CLASSES_AND_LIBRARIES
            )
        )
    }
}
```

下面是类方法配置 .json 文件示例：

```
{
  "methods": [
    {
      "owner": "android/location/LocationManager",
      "name": "getLastKnownLocation",
      "message": "大佬，麻烦请用项目xxx库中xxx类的xxx方法获取",
      "excludes": [
        {
          "caller": "androidx/appcompat/app/TwilightManager",
          "name": "getLastKnownLocationForProvider"
        }
      ]
    },
    {
      "owner": "android/net/wifi/WifiInfo",
      "name": "getBSSID",
      "message": "大佬，麻烦请用项目xxx库中xxx类的xxx方法获取"
    },
    {
      "owner": "android/net/wifi/WifiInfo",
      "name": "getSSID",
      "message": "大佬，麻烦请用项目xxx库中xxx类的xxx方法获取",
      "excludes": [
        {
          "caller": "org/chromium/net/AndroidNetworkLibrary",
          "name": "getWifiSSID"
        },
        {
          "caller": "org/chromium/net/NetworkChangeNotifierAutoDetect$WifiManagerDelegate",
          "name": "getWifiSsid"
        }
      ]
    },
    {
      "owner": "android/net/wifi/WifiInfo",
      "name": "getMacAddress",
      "message": "大佬，麻烦请用项目xxx库中xxx类的xxx方法获取"
    },
    {
      "owner": "android/telephony/TelephonyManager",
      "name": "getImei",
      "message": "大佬，麻烦请用项目xxx库中xxx类的xxx方法获取"
    },
    {
      "owner": "android/telephony/TelephonyManager",
      "name": "getDeviceId",
      "message": "大佬，麻烦请用项目xxx库中xxx类的xxx方法获取"
    },
    {
      "owner": "android/content/Context",
      "name": "getExternalFilesDir",
      "message": "大佬，麻烦请用项目xxx库中xxx类的xxx方法获取"
    },
    {
      "owner": "android/content/Context",
      "name": "getExternalFilesDir",
      "message": "大佬，麻烦请用项目xxx库中xxx类的xxx方法获取"
    }
  ],
  "output": {
    "type": "single"
  }
}
```

## 总结

在 App 上架的时候，如果隐私不合规，可以通过上面这种：查看隐私权限方法调用者来快速定位。这种方法的原理，可以通过自定义 lint 规则来做，也可以通过 asm 字节码插桩来做。它们的核心都是通过扫描项目和三方库的 class 文件，然后再检查类文件中的类方法。

其实，上面这个工具，还可以满足需求，例如：

- 在项目中，代码收敛，比如将所有使用 getExternalFilesDir 方法的地方都替换为一个 工具类方法；
- 在项目中，某个库方法有更改，找到所有使用的地方包括其它 aar 库；

Demo Github 地址：[https://github.com/WJRye/lint_mthod_collector](https://github.com/WJRye/lint_mthod_collector)
