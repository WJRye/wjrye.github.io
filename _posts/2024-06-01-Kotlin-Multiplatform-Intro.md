---
layout: post
title: 采用 Kotlin Multiplatform 做跨平台
categories: [KMP]
description: 采用 Kotlin Multiplatform 做跨平台
keywords: Kotlin, 跨平台
mermaid: false
sequence: false
flow: false
mathjax: false
mindmap: false
mindmap2: false
---

是否要采用Kotlin Multiplatform 做跨平台，首先需要了解清楚几个前置问题：

1. Kotlin Multiplatform 是什么？
2. 为什么学习 Kotlin Multiplatform？
3. 为什么选择 Kotlin Multiplatform？
4. 是否要选择 Kotlin Multiplatform 作为自己未来的发展方向？
5. 何时入场？

## Kotlin Multiplatform 是什么？

一句话概括：Kotlin Multiplatform 是 JetBrains 推出的使用 Kotlin 语言开发的开源跨平台框架，目前
支持 Android、iOS、Web、Desktop平台，但主要聚焦在 Android 和 iOS 移动端平台，其中通过 Kotlin Multiplatform 实现**逻辑**在各个平台代码共享，通过 Compose Multiplatform 实现 **UI** 在各个平台
代码共享。

所以学习 Kotlin Multiplatfrom 的首要条件是：熟悉 Kotlin 语言（早期拥抱 Kotlin 语言开发的人有优势）。

为什么主要聚焦在移动端平台？还是因为移动端有刚需：

> 据Statista 称，2022 年第三季度，Google Play Store 上共有 355 万个移动应用程序，App Store 上共有 160 万个应用程序，Android 和 iOS 目前占[全球移动操作系统市场的 99%](https://gs.statcounter.com/os-market-share/mobile/worldwide)。

再加上国内的安卓应用商店中的移动应用程序。那么多移动应用程序，难道就没有办法解决跨平台开发的问题吗？同一件事情非得做两遍？技术在进步，而且又有需求，肯定会解决，至于何时为解决，持续关注。

这里，先回顾一下流行的跨平台框架： Flutter 和 React Native

> **Flutter 和 React Native**
> 
> ### [Flutter](https://www.jetbrains.com/help/kotlin-multiplatform-dev/cross-platform-mobile-development.html#flutter)
> 
> Flutter 由 Google 创建，是一个使用 Dart 编程语言的跨平台开发框架。Flutter 支持原生功能，例如位置服务、相机功能和硬盘访问。如果您需要创建 Flutter 不支持的特定应用功能，则可以使用 Platform [Channel 技术](https://brightmarbles.io/blog/platform-channel-in-flutter-benefits-and-limitations/)编写特定于平台的代码。
> 
> Flutter 构建的应用需要共享所有 UX 和 UI 层，因此它们可能并不总是感觉 100% 原生。该框架的一大优点是其热重载功能，它允许开发人员进行更改并立即查看。
> 
> 在列情况下，这个框架可能是最佳选择：
> 
> - 您希望在您的应用程序之间共享 UI 组件，但您希望您的应用程序看起来接近原生应用程序。
> 
> - 该应用程序预计会对 CPU/GPU 造成很大负载，性能可能需要优化。
> 
> - 您需要开发 MVP（最小可行产品）。
>   
>   Flutter 构建的最受欢迎的应用程序包括 Google Ads、阿里巴巴的闲鱼、eBay Motors 和 Hamilton。
> 
> ### [React Native](https://www.jetbrains.com/help/kotlin-multiplatform-dev/cross-platform-mobile-development.html#react-native)
> 
> Facebook 于 2015 年推出了 React Native，这是一个开源框架，旨在帮助移动工程师构建混合原生/跨平台应用。它基于 ReactJS（一个用于构建用户界面的 JavaScript 库）。换句话说，它使用 JavaScript 为 Android 和 iOS 系统构建移动应用。
> 
> React Native 提供对多个第三方 UI 库的访问，其中包含现成的组件，可帮助移动工程师在开发过程中节省时间。与 Flutter 一样，借助快速刷新功能，您可以立即看到所有更改。
> 
> 在以下情况下，您应该考虑在您的应用中使用 React Native：
> 
> - 您的应用程序相对简单并且有望是轻量级的。
> - 开发团队精通 JavaScript 或 React。
> 
> 使用 React Native 构建的应用程序包括 Facebook、Instagram、Skype 和 Uber Eats。

## 为什么学习 Kotlin Multiplatform

为什么学习 KMP ？或者学习 KMP 的目的是什么？

对于个人而言，目的是：

- 成为全栈工程师：KMP 涵盖了前端、后端、移动端开发，在多个平台上共享代码
- 成为平台架构师：设计在多个平台（如iOS、Android、Web、Desktop、Server）的代码复用和模块化，以及如何在每个平台进行开发和部署等
- 研发自主产品：个人编写和维护一套UI和逻辑代码，让产品可以在Android，iOS，Desktop，Web上运行

如果个人期望向这些领域发展，可以主动去学习 KMP。

现在，部分公司已经看到了 KMP 的好处，那么对于公司而言，目的是：**跨平台开发，优化开发时间**。此时，在公司中，首先接触 KMP 的是处在技术领导和基础架构岗位的人，因为由他们推动落地。

对于**跨平台开发，优化开发时间**，这里又涉及**降本增效**问题，个人可结合自身和公司情况，做出相应的判断和决定。

如果公司着手 KMP 项目，那么首先受到影响的应该是做基础库的人，比如网络，日志，上传，下载等基础库，一般都会使用 KMP 重做。

## 为什么选择 Kotlin Multiplatform

目前已经有 Flutter、React Native跨平台框架，KMP 的优势有：

1. 支持代码在 Android, iOS，Desktop，Web, Server 平台重用。另外，在国内，也**支持代码在鸿蒙平台重用**
2. Compose Multiplatform 支持 UI 共享，Kotlin Multiplatform 支持逻辑共享

除了优势以外，KMP 突破了性能、访问设备原生功能，UI一致性的关键限制：

- 性能问题：通过使用不同的编译器后端（compiler backends），[Kotlin](https://kotlinlang.org/docs/multiplatform.html#code-sharing-between-platforms) 可编译为平台格式 - Android 为 JVM 字节码，iOS 为本机二进制文件。因此，共享代码的性能与原生编写的代码的性能相同
- 访问设备原生功能问题：KMP 可以通过 `expect` 和 `actual` 机制访问 Android 和 iOS SDK，以及 js 库等
- UI 一致性问题：Compose Multiplaform 支持了在 Android, iOS, DeskTop, Web 平台上的UI一致性

另外，如果你是Android 开发，那么使用 Kotlin 就可以轻松入手 KMP；如果你 iOS 开发，Kotlin 遵循与 Swift 类似的概念，那么也可以轻松入门。

> 编程思路不要受制于编程工具

## 是否要选择 Kotlin Multiplatform 作为自己未来的发展方向？

KMP 作为跨平台框架，到底有没有技术生命力，是否要选择它来作为自己未来的发展方向？这里，可以借鉴左耳朵耗子在“如何选择技术”中提出的观点来考虑：

> 选择一项技术，以确定自己未来的发展方向，要考虑几个条件：
> 
> 1. 这项技术是否已被大公司使用？大公司是否投入时间和金钱来支持这项技术？  
> 2. 这项技术是否有“杀手级”应用？也就是说，用它开发的某个产品出类拔萃？  
> 3. 这项技术的社区是否有热度？不管它是开源还是闭源技术，其所在社区的发展势态如何？  
> 4. 是否有其他人为这项技术作出贡献？其他软件是否有与其兼容？对于该技术，是否有标准委员会？这些都说明了技术生存与发展的基础条件。
> 
> 满足以上两个条件的技术，可选择；满足以上三个条件的技术一定会很火；如果以上四个条件都满足，那么这项技术就是未来。比如，Java 满足以上四个条件，技术社区十分火爆，Go 语言也类似。

对于这几个条件，KMP 的现状是：

1. Kotlin Multiplatform 和 Compose Multiplatform 由 [JetBrains 和 Google 共同主导建设](https://android-developers.googleblog.com/2024/05/android-support-for-kotlin-multiplatform-to-share-business-logic-across-mobile-web-server-desktop.html)
2. [大型公司](https://www.jetbrains.com/help/kotlin-multiplatform-dev/case-studies.html)：Forbes、Netflix、飞利浦、百度、快手等已经利用 KMP 做跨平台开发
3. Kotlin Multiplatform 和 Compse Multiplatform 在 Github、Reddit、StackOverflow 和 Google Trends 一直有热度，比如 Github 上 [Compose Multiplatform](https://github.com/JetBrains/compose-multiplatform)：[ **15k** stars](https://github.com/JetBrains/compose-multiplatform/stargazers)、[ **212** watching](https://github.com/JetBrains/compose-multiplatform/watchers)、[ **1.1k** forks](https://github.com/JetBrains/compose-multiplatform/forks)-
4. 有长期的发展规划：[2024 年 Kotlin Multiplatform 开发路线图](https://blog.jetbrains.com/zh-hans/kotlin/2023/11/kotlin-multiplatform-development-roadmap-for-2024/)
5. Kotlin 有语言委员会：[Language Committee guidelines](https://kotlinfoundation.org/language-committee-guidelines/)，每一项决定都是严格的

综上，KMP 是有技术生命力的，选择作为未来的发展方向也是可靠的。

## 何时入场？

技术日新月异，一项技术崛起又衰落。Flutter、React Native 已经是成熟的跨平台框架，KMP 成熟吗？KMP 是否还处于早期阶段：开发者需要花费大量的时间来解决 bug，而不是去实现新的功能。选择何时入场，首先需要关注的就是 KMP 的稳定性。

### [稳定性](https://www.jetbrains.com/help/kotlin-multiplatform-dev/supported-platforms.html#core-kotlin-multiplatform-technology-stability-levels)

Kotlin Multiplatform 所支持平台的稳定性级别：

| Platform（平台）             | Stability level（稳定级别） |
| ------------------------ | --------------------- |
| Android                  | Stable                |
| iOS                      | Stable                |
| Desktop (JVM)            | Stable                |
| Server-side (JVM)        | Stable                |
| Web based on Kotlin/Wasm | Alpha                 |
| Web based on Kotlin/JS   | Stable                |
| watchOS                  | Best effort           |
| tvOS                     | Best effort           |

- Stable 表示：放心使用。后续也会将根据严格的[向后兼容性规则](https://kotlinfoundation.org/language-committee-guidelines/)进行开发
- Alpha 表示：使用有风险，自行决定是否要使用
- Best effort 表示：仅供使用。如果发展好，就继续，否则就放弃

根据上面的稳定性，使用 KMP 编写 **逻辑** 共享代码，可以稳定支持 Android、iOS、Desktop、Server、Web(Kotlin/JS) 平台。

另外：

1. KMP 计划支持的 Native 平台，可以参考文档：[Kotlin/Native target support](https://kotlinlang.org/docs/native-target-support.html#tier-1)
2. 对于 KMP 目前支持的 Library 和 API ，可以参考文档：[Library](https://kotlinlang.org/docs/collections-overview.html)、[API](https://kotlinlang.org/api/latest/jvm/stdlib/)

KMP 目前有哪些能力？已经具备[网络](https://github.com/ktorio/ktor)，[数据库](https://github.com/cashapp/sqldelight)，[协程](https://github.com/Kotlin/kotlinx.coroutines)，[序列化](https://github.com/Kotlin/kotlinx.serialization)，[日期和时间](https://github.com/Kotlin/kotlinx-datetime)等跨平台能力。更多跨平台能力，可以去
[Github](https://github.com/search?q=Kotlin%20Multiplatform&type=repositories)上查找。

Compose Multiplatform 所支持平台的稳定性级别：

| Platform （平台）            | Stability level（稳定级别） |
| ------------------------ | --------------------- |
| Android                  | Stable                |
| iOS                      | Beta                  |
| Desktop (JVM)            | Stable                |
| Web based on Kotlin/Wasm | Alpha                 |

- Stable 表示：放心使用 Compose 编写跨平台UI
- Beta 表示：可以使用，不过还有一些问题
- Alpha 表示：使用有风险，自行决定是否要使用

根据上面的稳定性，使用 KMP 编写 **UI** 共享代码，可以稳定支持 Android、Desktop（JVM）平台，也可以在 iOS 平台使用。

---

了解了 Kotlin Multiplatform 和 Compose Multiplatform 的稳定性之后，对于个人和公司来说，可以直接做的就是：在 Android 和 iOS平台，编写 **逻辑** 共享代码。

目前可以共享的代码，主要是基础组件代码，使用 KMP 重新编写一套基础组件给 Android 和 iOS 使用。这些基础组件可以是：网络，日志，上传，下载，播放器等。

另外，如果对 KMP 非常感兴趣，也可以去做一些开创性的东西。

对于个人研发产品而言，逻辑和UI共享代码都可以尝试去做。

综上，何时入场，取决于你目前的角色。如果是移动端架构师和基础库负责人，那么可以直接入场，做基础组件逻辑共享和部分 UI 组件  UI 共享。如果是垂直业务负责人和业务架构研发，那么也可以直接入场，做业务基础组件逻辑共享。如果是业务开发，那么可以了解，不必直接入场，但如果有浓厚兴趣，以后希望做个人产品，那么还是可以先入场的。

---

参考文档：

1. [Kotlin Multiplatform Development](https://www.jetbrains.com/help/kotlin-multiplatform-dev/get-started.html)
2. [Compose Multiplatform](https://www.jetbrains.com/lp/compose-multiplatform/)
3. [Kotlin Blog](https://blog.jetbrains.com/zh-hans/kotlin/)
