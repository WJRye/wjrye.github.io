---
layout: post
title: Android Jetpack LifeCycle 实现原理分析
categories: [Android]
description: LifeCycle 是 Jetpack 提供的一个可感知 Activity 或 Fragment 的生命周期 变化的组件。
keywords: Android, Jetpack, LifeCycle
mermaid: false
sequence: false
flow: false
mathjax: false
mindmap: false
mindmap2: false
---

## LifeCycle

LifeCycle 是 Jetpack 提供的一个可感知 Activity 或 Fragment 的生命周期 变化的组件。这个组件方便业务针对生命周期的变化做出相应管理和改变，也可以防止业务内存泄漏。

首先了解下与 LifeCycle 相关的概念：

- `LifeCycle`：抽象类，定义了添加和删除观察者（`LifecycleObserver`）的抽象方法，也定义了生命周期事件`Lifecycle.Event` ，以及生命周期状态`Lifecycle.State`；
- `Lifecycle.Event` ：`LifeCycle`的内部枚举类，定义了 `LifecycleOwner`（也就是Activity 或 Fragment ）的生命周期事件，从 `ON_CREATE` 到 `ON_DESTROY`；
- `Lifecycle.State`：`LifeCycle`的内部枚举类，定义了生命周期状态。
- `LifecycleRegistry`： `LifeCycle` 的实现类，管理`LifecycleObserver`，维护生命周期状态`Lifecycle.State`，并负责分发 Activity 或 Fragment 的生命周期事件`Lifecycle.Event`；
- `LifecycleObserver`：接口，生命周期观察者，它没有任何方法，依赖注解 `OnLifecycleEvent`；
- `FullLifecycleObserver`：接口，继承自`LifecycleObserver`接口，以生命周期方法（onCreate...onDestroy）的形式观察 Activity 或 Fragment 的生命周期；
- `LifecycleEventObserver`：接口，继承自`LifecycleObserver`接口，以生命周期事件（ON_CREATE...ON_DESTROY）的形式观察 Activity 或 Fragment 的生命周期；
  - `LifecycleOwner`：接口，关联 Activity 或 Fragment 相关生命周期事件；
- `OnLifecycleEvent`：注解，主要是声明方法监听  `Lifecycle.Event` 事件；
- `Lifecycling`：辅助类，将 `LifecycleObserver` 转换为适配器 `LifecycleEventObserver`（`FullLifecycleObserverAdapter`、`SingleGeneratedAdapterObserver`、`CompositeGeneratedAdaptersObserver`、`ReflectiveGenericLifecycleObserver`），其中`ReflectiveGenericLifecycleObserver` 处理 以注解`OnLifecycleEvent` 声明的`LifecycleObserver`。

了解了 与 LifeCycle 的相关概念后，再了解下它的简要类关系图：
![在这里插入图片描述](/images/posts/2021-08-24-Android-Jetpack-LifeCycle/p1.jpg)
通常情况下，业务定义的 `LifecycleObserver`，一般是通过注解 `OnLifecycleEvent` 来观察 Activity 或 Fragment 相关生命周期，当 ReportFragment 或 Fragment 的生命周期发生变化时候，`ReflectiveGenericLifecycleObserver`  通过反射的方式调用 `LifecycleObserver` 的相关含有`OnLifecycleEvent` 注解的方法。

这里 Activity 的生命周期的监听是通过 `ReportFragment` 实现的，Fragment 的生命周期的监听是通过 它自身生命周期回调实现的。另外，定义在 `LifecycleObserver` 中声明的生命周期事件，都是在 Activity 或 Fragment 的自身生命周期方法调用之后再调用的。由于 `LifecycleRegistry` 在 `addObserver` 的时候，会以一个链表结构的 Map  来存储 `LifecycleObserver`，所以注册的相关 `LifecycleObserver`，最后接受处理生命周期事件时，也是按照添加顺序依次触发的。

流程图：
![loading-ag-205](/images/posts/2021-08-24-Android-Jetpack-LifeCycle/p2.png)
Activity 中的 LifeCycle 的流程是：

1. 在 Activity 中实现 `LifecycleOwner` 接口，并创建 `LifecycleRegistry` 对象；
2. 在 `ReportFragment` 中，当前版本SDK 大于等于 29（也就是安卓10）时，会创建 Activity 的生命周期监听器，否则就使用 ReportFragment 的生命周期作为监听；
3. 当 `ReportFragment`的生命周期发生变化时，通过 `ReportFragment`  中的 activity 对象，获取 `LifecycleOwner`对象 ，然后获取 `Lifecycle` 的实现类 `LifecycleRegistry` 对象，然后调用它的 `handleLifecycleEvent` 方法；
4. 添加生命周期观察者`LifecycleObserver`，通过 Activity 获取  `Lifecycle`的实现类 `LifecycleRegistry` 对象，调用 `addObserver` 方法，移除生命周期观察者`LifecycleObserver`，调用 `removeObserver` ；
5. `addObserver` 方法会创建一个 `ObserverWithState` 对象，并将`LifecycleObserver`对象传递给  `ObserverWithState` 的构造函数，在这个方法中，通过 `Lifecycling` 创建一个 实现了 `LifecycleEventObserver` 的适配器对象，`LifecycleObserver`对象被包装在`LifecycleEventObserver` 的适配器对象中；
6. `LifecycleRegistry`  会用一个 链表的 Map 去维护 `LifecycleObserver`对象 和 `ObserverWithState` 对象，也会维护 生命周期状态 `Lifecycle.State`；
7. `LifecycleRegistry`  对象 接受 `handleLifecycleEvent` 时，就会触发添加的 `LifecycleEventObserver` 的 `onStateChanged`方法；然后适配者（`FullLifecycleObserverAdapter`、`SingleGeneratedAdapterObserver`、`CompositeGeneratedAdaptersObserver`、`ReflectiveGenericLifecycleObserver`）通知`LifecycleObserver`观察者。

Fragment 中的 LifeCycle 的流程，除了 生命周期的监听方式不一样以外，其它的和 Activity 一样。Fragment 是在自身的生命周期方法回调中做的处理。

## 例子

定义生命周期观察者，使用注解 `OnLifecycleEvent` 定义生命周期事件：

```
open class BizXXX : LifecycleObserver {

    @OnLifecycleEvent(value = Lifecycle.Event.ON_CREATE)
    fun onCreate() {

    }

    @OnLifecycleEvent(value = Lifecycle.Event.ON_START)
    fun onStart() {

    }

    @OnLifecycleEvent(value = Lifecycle.Event.ON_RESUME)
    fun onResume() {

    }

    @OnLifecycleEvent(value = Lifecycle.Event.ON_PAUSE)
    fun onPause() {

    }

    @OnLifecycleEvent(value = Lifecycle.Event.ON_STOP)
    fun onStop() {

    }

    @OnLifecycleEvent(value = Lifecycle.Event.ON_DESTROY)
    fun onDestroy() {

    }

}
```

或者 使用 `DefaultLifecycleObserver` 观察生命周期：

```
open class BizXXX : DefaultLifecycleObserver {

    override fun onCreate(owner: LifecycleOwner) {

    }

    override fun onStart(owner: LifecycleOwner) {

    }

    override fun onResume(owner: LifecycleOwner) {

    }

    override fun onPause(owner: LifecycleOwner) {

    }

    override fun onStop(owner: LifecycleOwner) {

    }

    override fun onDestroy(owner: LifecycleOwner) {

    }
}
```

在 Activity 中添加：

```
class YourActivity : BaseAppCompatActivity() {

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        this.lifecycle.addObserver(BizXXX())
    }
}
```

在 Fragment 中添加：

```
class YourFragment : Fragment() {

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        this.lifecycle.addObserver(BizXXX())
    }
}
```

当不能直接使用 LifeCycle 的时候，也可以间接使用 LifeCycle，比如 数据层也要监听生命周期的变化，那么可以将 Presenter 层定义 为一个 LifeOwner：

```
open class BizPresenter : LifecycleObserver, LifecycleOwner {

    private val lifecycleRegistry: LifecycleRegistry = LifecycleRegistry(this)

    override fun getLifecycle(): Lifecycle {
        return lifecycleRegistry
    }

    @OnLifecycleEvent(value = Lifecycle.Event.ON_CREATE)
    fun onCreate() {
        lifecycle.addObserver(BizModel())
        lifecycleRegistry.handleLifecycleEvent(Lifecycle.Event.ON_CREATE)
    }

    @OnLifecycleEvent(value = Lifecycle.Event.ON_START)
    fun onStart() {
        lifecycleRegistry.handleLifecycleEvent(Lifecycle.Event.ON_START)
    }

    @OnLifecycleEvent(value = Lifecycle.Event.ON_RESUME)
    fun onResume() {
        lifecycleRegistry.handleLifecycleEvent(Lifecycle.Event.ON_RESUME)
    }

    @OnLifecycleEvent(value = Lifecycle.Event.ON_PAUSE)
    fun onPause() {
        lifecycleRegistry.handleLifecycleEvent(Lifecycle.Event.ON_PAUSE)
    }

    @OnLifecycleEvent(value = Lifecycle.Event.ON_STOP)
    fun onStop() {
        lifecycleRegistry.handleLifecycleEvent(Lifecycle.Event.ON_STOP)
    }

    @OnLifecycleEvent(value = Lifecycle.Event.ON_DESTROY)
    fun onDestroy() {
        lifecycleRegistry.handleLifecycleEvent(Lifecycle.Event.ON_DESTROY)
    }

}
```

```
class BizModel : LifecycleObserver {

    @OnLifecycleEvent(value = Lifecycle.Event.ON_CREATE)
    fun onCreate() {

    }

    @OnLifecycleEvent(value = Lifecycle.Event.ON_DESTROY)
    fun onDestroy() {

    }
}
```

这种嵌套也是很实用的。
