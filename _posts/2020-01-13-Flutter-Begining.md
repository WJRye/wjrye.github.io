---
layout: post
title: Flutter 分享会总结
categories: [Flutter]
description: 参加 Flutter 分享会
keywords: Flutter
mermaid: false
sequence: false
flow: false
mathjax: false
mindmap: false
mindmap2: false
---

## 前言

谷歌1月7日举办了 Flutter Interact 活动，因为之前领导说我们有名额可以去参加，很高兴就报了名。

在参加这次活动之前，也了解过Flutter，写过Demo，看过网上一些项目，加入过一些论坛和群，总的感觉就是将Flutter接入项目还为时过早。不过，Flutter在国内的欢呼声一直很高，并持续在增长，时不时听到某某公司项目接入了Flutter，谷歌发布了Flutter最新xx版本，谁谁开源了Flutter项目等。所以，看到这次活动，就踊跃报名了参加，想去看看过了一年半载，Flutter在实际应用中到底如何。

这次活动，主要由谷歌的Flutter工程师和字节跳动的工程师分享。分享会干货很多，了解到了Flutter的现状，以及Flutter在国内公司中的一些实际应用情况。下面就从用Flutter开发应用程序的质量、成本和收益以及项目接入Flutter方面谈谈自己的总结。

谈总结前，先发一些活动图感受下（稍微发两张）：

![](/images/posts/2020-01-13-Flutter-Begining/p1.jpeg)

![](/images/posts/2020-01-13-Flutter-Begining/p2.jpeg)

![](/images/posts/2020-01-13-Flutter-Begining/p3.jpeg)

![](/images/posts/2020-01-13-Flutter-Begining/p4.jpeg) 

## Flutter开发应用程序的质量、成本和收益

Flutter开发应用程序的质量：稳定性、性能、视觉方面

- 稳定性：iOS稳定性较好，Android还有些问题，比如在三星、华为等设备中可能出现黑屏，不是很兼容，不过谷歌官方正在投入人力解决，保证稳定性。
- 性能：帧率可以媲美原生应用，并且在低端机型上表现出的帧率要比原生还要好，也比H5的要好。
- 视觉：多端一致，文本、图片、动画等显示都能适配多端各种机型。

Flutter开发应用程序的成本：短期和长期

- 短期：开发每一个业务，每一端需要配备至少一个开发，而且研发产出效率会比较低，因为每一端的开发都需要了解在各个端上的开发。
- 长期：等开发熟练之后，效率可以提升一倍，一个业务只需要一个开发。

Flutter开发应用程序的收益：

- 提升UI开发效率：双端一致，并且热重载可以帮助快速的修改和调试UI。
- 保证多端可同时发版：业务需求可以同一个截止日期完成。
- 提升用户体验：在Andorid低端设备的上的流畅度更好，比之前H5页面的流畅度也更好。
- 人力节省：产品/设计/测试只需要对接一个开发。

## 项目接入Flutter

初级阶段：

- 与项目的融合：搭建Flutter架构，让Flutter在项目中跑起来。
- 可以接入的业务：新业务或者更新频次低的老业务，或H5页面，可以优先替换H5页面。

后续阶段：

- 混合工程自动集成
- 封装广告、付费、分享，推送等Native基础能力
- 沉淀图片、埋点等常用的库

接入Flutter先需要做的：

- 学习Flutter，熟悉Flutter开发
- 了解混合工程的开发
- 如何封装基础库，以及封装基础库的成本，如：网络库，图片库等

## Flutter学习资料

4. 学习 Dart 语言地址：[Dart](https://www.dartcn.com/guides/get-started)

5. 学习 Flutter 地址：[Flutter](https://flutterchina.club/get-started/install/)

6. Flutter 开发者帮助 APP：[flutter-go](https://github.com/alibaba/flutter-go)

7. 闲鱼[Fish Redux](https://yq.aliyun.com/articles/692482)
