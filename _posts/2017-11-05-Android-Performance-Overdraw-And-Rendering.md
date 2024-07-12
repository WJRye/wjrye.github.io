---
layout: post
title: Android 关于过度绘制和渲染的介绍
categories: [Android]
description: 过度绘制：屏幕上的某个像素在同一帧的时间内被绘制了多次。过度渲染：操作在 16ms 内未完成。
keywords: Android, Performance
mermaid: false
sequence: false
flow: false
mathjax: false
mindmap: false
mindmap2: false
---

## 1.引起卡顿的原因

在Android中，过度绘制和过度渲染都是造成应用程序卡顿的主要原因。

关于过度绘制：

> 屏幕上的某个像素在同一帧的时间内被绘制了多次。

如果一个布局的视图之间层级嵌套太多，或者每个视图都有自己的背景就会导致“屏幕上的某个像素在同一帧的时间内被绘制了多次“。

关于过度渲染：

> Android系统每隔16ms发出VSYNC信号，触发对UI进行渲染，如果每次渲染都成功，这样就能够达到流畅的画面所需要的60fps，为了能够实现60fps，这意味着程序的大多数操作都必须在16ms内完成。

那么，如果每一帧的渲染时间超过了16ms，就会产生丢帧，导致卡顿现象。在Android中，执行复杂动画、绘制或裁剪过大图片 、频繁刷新UI等操作都是发生这些现象的原因。

---

## 2.分析优化

## 过度绘制分析

**调试GPU过度绘制**主要通过在屏幕上显示不同的颜色来分析，它的开关在设置的开发者选项中，如下图所示：

<img src="../../../../images/posts/2017-11-05-Android-Performance-Overdraw-And-Rendering/p1.png" width="50%" height="50%" />

这个功能主要是通过在UI上显示不同的颜色来识别过度绘制。过度绘制发生在同一帧内绘制在同一像素上超过了一次。因此，这个可视化显示了应用程序可能做了很多额外的渲染工作，由于GPU努力渲染的并不是对用户可见的像素，所以这可能会成为一个性能问题。因此，应该尽可能的[修复过度绘制事件](https://developer.android.com/topic/performance/rendering/overdraw.html#fixing)。

显示在UI上的不同颜色元素分别表示：

<p>
    <div>
<table>
  <tr>
    <th>颜色</th>
    <th>过度绘制</th>
  </tr>
  <tr>
    <td>真实颜色</td>
    <td>没有过度绘制</td>
  </tr>
  <tr>
    <td bgcolor="#D0D1FE">蓝色</td>
    <td>过度绘制一次</td>
  </tr>
   <tr>
    <td bgcolor="#DDFAD3">绿色</td>
    <td>过度绘制两次</td>
  </tr>
  <tr>
    <td bgcolor="#F1BEC4">粉红色</td>
    <td>过度绘制三次</td>
  </tr>
   <tr>
    <td bgcolor="#E78087">红色</td>
    <td>过度绘制四次或者更多次</td>
  </tr>
</table>
</div>
<p/>

虽然过度绘制是不可避免的，但在优化应用程序的时候，应该尽量使显示的可视化颜色为真实颜色或者蓝色（过度绘制一次）。 

<h4> 修复过度绘制</h4>

这里有几种策略可以减少或者消除过度绘制： 

<div>
<ul>
<li>移除布局中不必要的背景</li>
<li>使视图层级扁平化</li>
<li>降低透明度</li>
</ul>
</div>

<h4>移除布局中不必要的背景</h4>

<p>默认情况下，一个布局没有背景，这意味着它自己不会直接地渲染任何东西。当布局有背景时，就有可能过度绘制。</p>

<p>移除不必要的背景是一种提升性能的快速方式。不必要的背景可能是从不可见的，因为它的背景完全会被应用程序绘制在最顶层的视图所覆盖。例如，当绘制的子视图在父视图之上时，系统可能完全覆盖父视图的背景。</p>

<p>想找到为什么过度绘制，可以使用<a herf="https://developer.android.com/studio/debug/layout-inspector.html">Layout Inspector</a> 工具来遍历视图层级。当这样做时，可消除任何对用户不可见的背景。例如，许多容器共享一个背景，就可以消除不必要的背景（在应用程序中，设置窗口背景为主要的颜色背景，让在上面的其它所有容器都不定义背景值）。</p>

<p>使用 Layout Inspector 工具简单介绍：</p>

<p>Layout Inspector 在Android Studio IDE中，可以在运行时检测应用程序的视图层级。它在菜单Tools的Android下面，如下图：</p>

<div><img title="" src="../../../../images/posts/2017-11-05-Android-Performance-Overdraw-And-Rendering/p2.png" alt=""></div>

<p>然后，再选择要检测的应用程序的进程。</p>

<p>Layout Inspector会捕获一张屏幕快照，在AndroidStudio中会呈现：<p>

<div>
<ul><li>View树：在布局中的视图层级</li>

<li>快照：有每一个View边界的屏幕快照</li>
 <li>属性表：选中View的布局属性</li>
</ul>
</div>

<div><img title="" src="../../../../images/posts/2017-11-05-Android-Performance-Overdraw-And-Rendering/p3.png" alt=""></div>

<h4>使视图层级扁平化</h4>

<p>现代布局使堆叠和分层视图更加容易。然而，这样做却会过度绘制，从而导致性能降低。特别在每一个堆叠视图对象是不透明的场景中，可见和不可见的像素都需要绘制在屏幕上。<p/>

<p>如果遇到这类问题，可以通过<a herf="https://developer.android.com/topic/performance/rendering/optimizing-view-hierarchies.html">优化视图层次结构</a>来减少重叠的UI对象数量，从而提升性能。</p>

<p>要想扁平化布局效果，可以使用约束性布局<a herf="https://constraintlayout.com/">ConstraintLayout</a>。通过它，无论多么复杂的层级嵌套布局，最后都可以优化为下面这种结构：</p>

<table>
<td>
<p>ViewGroup</p>
  <p>View</p>
  <p>View</p>
  <p>View</p>
  <p>... </p>
  <p>... </p>
<p>ViewGroup</p>

</td>
</table>



<h4> 降低透明度</h4>

<p>在屏幕上渲染透明像素，被称为alpha渲染，是过度绘制的一个重要因素。不像标准的过度绘制，通过在它们上面绘制不透明像素，系统会完全隐藏现有的绘制像素，为了出现混合平衡，透明对象需要现有的像素被首先绘制。像透明动画，淡入淡出，水滴阴影视觉效果，都涉及某种透明度，这会促成严重的过度绘制。可以通过减少渲染透明对象的数量来改善过度绘制。例如，可以通过在TextView中绘制有透明度alpha值的黑色文本来得到灰色文本。但是也可以通过绘制灰色文本以更好的性能达到相同效果。</p>

<h3>过度渲染分析</h3>

<p>精确定为到导致卡顿现象的代码特别困难，但可以通过一些工具来分析。</p>



<p><a herf="https://developer.android.com/studio/profile/inspect-gpu-rendering.html#profile_rendering">GPU呈现模式分析</a>可以通过视觉化看到渲染UI时遇到的问题，如执行了更多的渲染工作，或执行了时间较长的线程和GPU操作。</p>

<p>它主要是在屏幕上显示滚动的柱状图，从视觉上可以看到渲染UI窗口的帧相对于每帧16ms所花费的时间。</p>

<p>它的开关在设置的开发者选项中，如下图所示：</p>

<div>

<table>
  <tr>
    <td><img src="../../../../images/posts/2017-11-05-Android-Performance-Overdraw-And-Rendering/p4.png"/></td>
    <td><img src="../../../../images/posts/2017-11-05-Android-Performance-Overdraw-And-Rendering/p5.png"/></td>
  </tr>
</table>

</div>

</p>

<p>上面图中呈现的彩色部分需要系统6.0以上才能显示，它说明了：</p>

<ul>

<li>水平绿色横线代表16ms。要达到每秒60帧，每帧的条形图应该保持在16ms以下。条形图在任何时候超过了这条线，都有可能暂停动画。</li>

<li>每个条形图都有着色的组件，且这些元素映射到渲染管道中的一个阶段。</li>

<li>每个条形图表示每一帧渲染的时间，条形图越高表示花费的渲染时间越长。</li>

</ul>

不同颜色条形图表示的意思是：

<table border="1" cellpadding="2">
  <tr>
    <th>条形图组件</th>
    <th>渲染阶段</th>
    <th>描述</th>
  </tr>
  <tr>
    <td bgcolor="#FF9900"></td>
    <td>交换缓冲区阶段(Swap Buffers)</td>
    <td>表示 CPU 等待 GPU 完成其工作的时间。 如果此竖条升高，则表示应用在 GPU 上执行太多工作。</td>
  </tr>
   <tr>
    <td bgcolor="#F7412D"></td>
    <td>命令问题阶段(Command Issue)</td>
    <td>表示 Android 的 2D 渲染器向 OpenGL 发起绘制和重新绘制显示列表的命令所花的时间。 此竖条的高度与它执行每个显示列表所花的时间的总和成正比—显示列表越多，红色条就越高。</td>
  </tr>
  <tr>
    <td bgcolor="#48C2F9"></td>
    <td>同步和上传阶段(Sync& Upload)</td>
    <td>表示将位图信息上传到 GPU 所花的时间。 大区段表示应用花费大量的时间加载大量图形。</td>
  </tr>
    <tr>
    <td bgcolor="#1194F6"></td>
    <td>绘制阶段(Draw)</td>
    <td>表示用于创建和更新视图显示列表的时间。 如果竖条的此部分很高，则表明这里可能有许多自定义视图绘制，或 onDraw 函数执行的工作很多。</td>
  </tr>
 <tr>
    <td bgcolor="#1AB3AB"></td>
    <td>测量/布局阶段(Measure / Layout)</td>
    <td>表示在视图层次结构中的 onLayout 和 onMeasure 回调上所花的时间。 大区段表示此视图层次结构正在花很长时间进行处理。</td>
  </tr>
 <tr>
    <td bgcolor="#00A59A"></td>
    <td>动画阶段(Animation)</td>
    <td>表示评估运行该帧的所有动画程序所花的时间。 如果此区段很大，则表示您的应用可能在使用性能欠佳的自定义动画程序，或因更新属性而导致一些意料之外的工作。</td>
  </tr>
  <tr>
    <td bgcolor="#00998E"></td>
    <td>输入处理阶段(Input Handling)</td>
    <td>表示应用执行输入 Event 回调中的代码所花的时间。 如果此区段很大，则表示此应用花太多时间处理用户输入。 考虑将此处理任务分流到另一个线程。</td>
  </tr>
<tr>
    <td bgcolor="#007B6F"></td>
    <td>其他时间/VSync 延迟阶段(Misc Time / VSync Delay)</td>
    <td>表示应用执行两个连续帧之间的操作所花的时间。 它可能表示界面线程中进行的处理太多，而这些处理任务本可以分流到其他线程。</td>
  </tr>
</table>
<p></p>

渲染阶段的意思是：

<h3>输入处理阶段：</h3>

管道的输入处理阶段测量了应用程序花费在处理输入事件需要的时间。这个度量标准表明了应用程序执行输入事件回调方法中的代码所花费的时长。

<h4>当这部分很大时</h4>

<p>在这个区域的值很高，通常是由于太多的工作或太复杂的工作发生在输入处理事件回调方法中。因为这些回调方法总发生在主线程里面，要解决这些问题，可以直接优化这些工作，或分放这些工作到不同的线程中。</p>

<p>值得注意的是，RecyclerView在滚动的时候会出现在这个阶段。RecyclerView滚动时会立即消耗触摸事件。因此，它会填充或更新新的条目视图。所以，让这个操作尽可能的快是非常重要的。可以使用TraceView或Systrace来做进一步分析。</p>

<h3>动画阶段</h3>

动画阶段显示了评估所有运行在帧的动画所花费的时间。最通常用的动画是ObjectAnimator, ViewPropertyAnimator, 和Transitions.

<h4>当这部分很大时</h4>

在这个区域的值很高，通常是由于动画的一些属性改变而执行的工作导致的。

<h3>测量/布局阶段</h3>

<p>为了让Android绘制视图条目到屏幕上，在层级中从布局到视图会执行两个特别的操作。</p>

<p>首先，系统会测量视图条目。每一个视图和布局在屏幕上有描述对象大小的特殊数据。有些视图有特定的大小，其它一些有一个适应父布局容器大小的大小。<p>

<p>其次，系统会布局视图条目。一但系统计算完子View的大小，系统就会对屏幕上的视图进行布局、分级和定位。</p>

<p>系统不仅对要被绘制的视图进行布局和测量，而且也会对这些视图的父层级进行布局和测量，最后一直布局和测量到根视图。</p>

<h4>当这部分很大时</h4>

<p>如果应用程序在这个区域的每一帧花费了大量时间，通常是因为大量的视图需要被布局，或者在层级的错误地方出现了像执行多个布局和测量迭代的问题。在这两种情况下，解决性能需要<a herf="https://developer.android.com/topic/performance/rendering/optimizing-view-hierarchies.html">改进视图层次结构的性能</a></p>

<p>添加在onLayout(boolean, int, int, int, int) 和 onMeasure(int, int)方法中的代码也会造成性能问题。TraceView和Systrace工具可以用来帮助检查这些回调方法，从而识别出代码中可能存在的问题。</p>

<h3>绘制阶段</h3>

<p>绘制阶段将视图的绘制操作（如背景和文本绘制）转换为一系列的本地绘制命令。系统将这些命令捕获到显示列表中。</p>

<p>绘制条形图记录了完成捕获命令到display list中所花费的时间，以及这帧内在屏幕需要更新的所有视图。测量时间用于在应用程序中添加到UI对象里的所有代码。这些代码可能在 onDraw()方法、dispatchDraw()方法和属于Drawable类子类的所有draw ()方法里。</p>

<h4>当这部分很大时</h4>

简单的说，这个衡量标准可以理解为每次刷新View运行onDraw()方法所花费的时间。这个测量包括分发绘制命令到子View和drawable（当有drawable）所花费的时间。出于这个原因，当看到这个条形图很突出时，可能是刷新大量的View造成的。刷新需要重新生成View显示列表。另外，在自定义View的onDraw()方法中有一些极其复杂的逻辑也会导致时间变长。

<h3>同步和上传阶段</h3>

<p>Sync & Upload的衡量标准表示在当前帧中将bitmap对象从CPU内存转移到GPU内存所花费的时间。</p>

<p>对于不同的处理器，CPU和GPU有不同的RAM区域来专门处理。当在Android上绘制bitmap时，在GPU渲染它到屏幕上之前，系统需要先将bitmap转移到GPU内存中。</p>

> 注意：在Lollipop设备上，这个阶段是紫色的。

<h4> 当这部分很大时</h4>

<p>所有的资源在被绘制到帧中之前，需要先驻留到GPU内存中。这个衡量标准的值很高意味着加载了大量的小资源或者小量的大资源。一个常见的情况是，应用程序展示一张接近屏幕大小的bitmap。另一种情况是，展示大量的缩略图。</p>

<p>为了缩小这个条形图，可采取下面的方式：</p>

<ul><li>确保bitmap的分辨率不会比显示它的大小大很多。例如，避免在1024*1024大小上显示48*48的图片。</li><li>在下一个同步阶段之前，利用 prepareToDraw()方法来异步预加载bitmap。</li></ul>

<h3>命令问题阶段</h3>

<p>发布命令部分表示在屏幕上绘制display list发布所有命令所花费的时间。</p>

<p>对于屏幕绘制display list，它会发送必要的命令到GPU。通常，通过OpenGL ES API来执行此操作。</p>

<p>这个过程需要一些时间，在发送命令到GPU之前，系统会对每一个命令执行最后的转换和裁剪。在GPU中计算最后的命令还会引起额外的开销。这些命令包括最终转换和附加裁剪。</p>

<h4>当这部分很大时</h4>

<p>这个阶段所花费的时间是，给定帧内系统渲染的display list的复杂性和数量的直接测量。例如，有很多draw操作，尤其是在每一个原始draw中还有小的固定的开销的情况。例如：</p>

<table>
<td>
for (int i = 0; i < 1000; i++)
<p></p>
canvas.drawPoint()    
</td>
</table>

<p>比下面有更多昂贵的开销：</p>

<table>
<td>
canvas.drawPoints(mThousandPointArray);
</td>
</table>

<p>在发布命令和实际绘制display list之间并不总是是1:1的相关性。发布命令与捕获发送绘制命令到GPU所花费的时间与不一样的。Draw 衡量标准表示捕获发布命令到display list所花费的时间。</p>

<p>之所以出现这种差异，是因为display list会被系统尽可能的缓存。因此，在滚动、转换或动画情况下，需要系统重新发送display list，但不必实际创建它-从头开始重新捕获绘制命令。因此，看到的是很高的"Issue commands"条形图，而不是Draw命令条形图。     </p>                                                                                                                                                                           

<h3>交换缓冲区阶段</h3>

一旦Android完成提交所有的display list到GPU，系统就发送最后一条命令告诉图形驱动器，当前帧已经完成。此时，系统最终可以显示更新后的图片到屏幕上。

<h4>当这部分很大时</h4>

<p>理解GPU执行工作与CPU是线性的是非常重要的。Android系统发布绘制命令到GPU后，就会移到下一个任务。GPU从队列中读取这些绘制命令然后处理它们。</p>

<p>在CPU发布命令比GPU消耗命令快的情况下，在处理器之间的通讯队列会被填满。当发生这种情况，CPU会阻塞，然后等待一直到队列中有空间来放置下一个命令。填满队列通常发生在Swap Buffers阶段，因为此时，整个帧的值命令已经被提交。</p>

<p>减轻这个问题的关键在于减少发生在GPU上的工作复杂性，在"Issue Commands"所要做的工作也一样。</p>

<h3>其他时间/VSync 延迟阶段</h3>

除了渲染系统执行它的工作要花费时间外，与渲染毫无关系发生在主线程中执行的一系列工作也需要花费时间。这个工作消耗的时间被称为Misc时间。Misc时间一般表示在两个连续渲染帧之间可能发生在UI线程中的工作时间。

<h4>当这部分很大时</h4>

<p>如果这个值很高，可能是应用程序中有回调，意图或发生在其它线程中其它工作。Method Tracing 或 Systrace 工具可以提供运行在主线程中任务的可见性。这些信息可以帮助实现性能改进目标。</p>

<p>从上面的解释，当看到如下示例图时：</p>

<img src="../../../../images/posts/2017-11-05-Android-Performance-Overdraw-And-Rendering/p6.png" width="50%" height="50%" />

<p>可以发现在条形图较高的帧中是Animation和Misc Time / VSync Delay渲染阶段花费了大量时间，所以可以优化性能不是很好的动画和把同一时刻的复杂操作放到不同的线程中处理。</p>

<h2>总结</h2>

<p>关于UI渲染和过度绘制就介绍和分析到这里。加快UI渲染需要在每一个渲染阶段进行优化，所涉及的方面较广，尤其在开发时要多注意和思考。另外，使用<a herf="https://developer.android.com/studio/profile/traceview.html">Method Tracing</a>或 <a herf="https://developer.android.com/studio/command-line/systrace.html">Systrace</a>工具可以帮助分析UI渲染问题。降低过度绘制 可以从移除布局中不必要的背景、使视图层级扁平化、降低透明度三方面来进行，编写应用程序应尽量使显示的可视化颜色为真实颜色或者蓝色（过度绘制一次）。</p>

参考：

1. <a herf="https://developer.android.com/studio/profile/dev-options-rendering.html">Profile GPU Rendering Walkthrough</a>

2. <a herf="https://developer.android.com/topic/performance/rendering/overdraw.html#fixing">Reducing Overdraw</a>
