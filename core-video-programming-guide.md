> 译注：
> 
> * 由于文档过于古老（上一次更新是 2007 年），第二部分的例子使用的是已经废弃且移除的 `OS X` 框架，因此第二部分没有翻译
> 
> * 尽管该文档只讲解了 `Core Video` 在 `OS X`(macOS) 系统上的特性，但大部分内容也同样适用于 `iOS` 系统。
> 
> * 原文地址：[https://developer.apple.com/library/content/documentation/GraphicsImaging/Conceptual/CoreVideo/CVProg_Intro/CVProg_Intro.html#//apple_ref/doc/uid/TP40001536-CH201-DontLinkElementID_27](https://developer.apple.com/library/content/documentation/GraphicsImaging/Conceptual/CoreVideo/CVProg_Intro/CVProg_Intro.html#//apple_ref/doc/uid/TP40001536-CH201-DontLinkElementID_27)

## 简介

本文档解释了 `Core Video` 的相关概念并阐述了如何使用 `Core Video` 编程接口来获取和处理视频帧。

### 什么是 `Core Video`

`Core Video` 是 `OS X` 上一种新的数字视频处理的管道模型。通过将处理过程分成数个独立的部分，使得开发者更加容易地获取和处理单个视频帧，并且不需要担心帧数据在不同的数据类型之间的转换（`QuickTime`，`OpenGL` 等等）或者显示同步问题。

`Core Video` 堪比 Core Image 和 Core Audio 技术。

`Core Video` 适用于：

* `OS X` v10.4 或以后版本
* `OS X` v10.3 且安装了 v7.0 或以后版本的 `QuickTime` 

为了获得最好的结果，你应该只在支持硬件图像加速(也就是 `Quartz Extreme`)的电脑上使用 `Core Video`.

### 目标读者

本文档的目标用户是想要深入处理视频图像的 `Carbon` 或 `Cocoa` 开发者。开发者应当熟悉数字视频系统、`OpenGL` 以及多线程编程。

仅当你想要处理单个视频帧的时候才需要使用 `Core Video`. 比如以下几种视频处理需要用到 `Core Video`:

* 颜色修正或添加滤镜，比如 `Core Image` 提供的滤镜
* 对视频图像添加物理形变（比如扭曲，或者将图像映射到物体表面）
* 将视频添加到 `OpenGL` 场景
* 为视频帧添加额外信息，例如可视时间码
* 合成多个视频流

如果你不需要如此复杂的处理（例如你只是想要在应用里播放视频），你应该使用简化了的视频播放器，比如 `Carbon` 里的 `HIMovieView` 或者 `Cocoa` 里的 `QTKit`. 你也可以使用 `Quartz Composer` 对视频添加特效。

### 组织结构

本文档被分为以下章节：

* `Core Video` 概念：阐述 `Core Video` 管道模型并解释了使用 `Core Video` API 时一些必要的关键概念
* `Core Video` 操作：展示如何使用 `Core Video` 来获取和处理视频帧

### 参见

作为补充，苹果开发者中心提供了以下额外资源：

* *Core Video Reference* 提供了一份关于 `Core Video` API 更详细的描述
* *OpenGL Programming Guide for Mac* 提供了 GL 纹理和 CGL 绘图上下文的信息
* *Cocoa OpenGL* 包含了 Cocoa 里可用的 `OpenGL` 类的信息（比如 NSOpenGLView）
* *Core Image Programming Guide* 包含了如何创建 `Core Image` 滤镜的信息，你可以把它应用到视频帧上

比外，`OpenGL` 官网([http://www.opengl.org/](http://www.opengl.org/))是获取 `OpenGL` API 信息的主要来源。

## `Core Video` 概念

Core Vide 是一个 `OS X` 系统处理数字视频的新模型。它提供两个主要功能来简化视频处理：

* 一个标准缓冲模型使得它更容易的在未压缩视频帧（比如说从 `QuickTime` 里获取的）和 `OpenGL` 之间切换
* 一个显示同步解决方案

本章节介绍这些功能背后的概念。

### `Core Video` 管道

`Core Video` 提出了一种从输入视频数据到将视频帧实际显示到屏幕上的离散管道模型，这种管道模型使得添加自定义处理变得更加容易。

图 1-1 `Core Video` Pipeline

![](/images/corevideo_pipeline.png)

从视频源（比如 `QuickTime`）得到的帧数据被分配给一个可视上下文（visual context）。这个可视上下文仅简单地指定你想要渲染的视频的绘制目标。比如说，这个上下文可以是 `Core Graphic` 也可以是 `OpenGL`. 大多数情况下，可视上下文与窗口内的一个视图绑定在一起，但一个离屏渲染上下文也是可能存在的。

> 注意：在 `QuickTime` 7.0 或以后的版本中，当你准备播放一个 `QuickTime` 影片的时候可以指定一个可视上下文，它取代了古老的 `GWorld` 或 `GrafPort` 渲染空间。

当指定了一个绘图上下文后，你可以自由地操控帧数据。比如说，你可以使用 `Core Image` 的滤镜来处理视频帧数据，或者用 `OpenGL` 给视频帧设置扭曲效果。这样做的话，你就将视频帧数据提交给 `OpenGL`，随后它会执行你的渲染指令（如果有的话）并将结果发送到显示设备上。

对开发者来说，`Core Video` 管道模型最重要的方面就是处理显示同步的 display link 和用来简化在不同缓冲类型之间移动帧数据的内存管理的通用缓冲模型（common buffering model）。大多数应用在处理视频时仅需要用到 display link。仅当生成或压缩视频帧的时候你才需要考虑使用缓冲。

### 显示连接 （display link）

为了简化视频和显示器刷新率的同步问题，`Core Video` 提供了一个特殊的定时器叫 *display link*，它工作在一个独立的高优先级线程，不会受到应用内交互的影响。

在过去，同步视频帧和显示器刷新率经常是一个问题，尤其是还需要处理音频的时候。你能做的仅仅是简单猜测何时输出一个视频帧（比如说使用一个定时器），这样做没有考虑到用户交互、CPU 负载、窗口混合等事件导致的延迟。`Core Video` display link 可以根据显示器类型和延迟智能地预估某一帧何时输出到显示器上去。

图 1-2 展示了 display link 在处理视频帧的时候如何与你的应用交互

图 1-2

![](/images/obtaining_frames.png)

* display link 周期性的调用你的回调来获取帧数据
* 然后你的回调需要根据请求事件获取帧数据，根据这个帧数据你会得到一个 `OpenGL` 纹理（这个例子假设你从 `QuickTime` 中获取帧数据，但你可以使用任意视频源来提供帧数据）。
* 现在你可以使用任意关于纹理的 `OpenGL` 指令来处理这个纹理数据

如果因为某些原因处理时长超过预期（也就是 display link 时间预估失败），如果有必要的话显卡依然可以放弃一些帧数据或者对时间错误进行补偿。

### 缓冲管理

如果你的应用是用来生成显示的帧数据，或者压缩得到的原始视频数据，你可能需要存储处理过程中的图像数据。`Core Video` 提供了不同的缓冲类型来简化这个操作。

首先，视频处理需要很大的开支。比如说你想要使用 `OpenGL` 处理 `QuickTime` 帧数据。在不同的缓冲类型之间转换并且处理内部内存开支是一件很繁琐的事情。现在，使用 `Core Video` 的话，缓冲就是 `Core Foundation` 风格的对象，它易于创建和销毁以及在不同的类型之间转换。

`Core Video` 定义了一个抽象缓冲类型：`CVBuffer`. 所有其他缓冲类型都继承 `CVBuffer`. `CVBuffer` 可以持有视频、音频或者可能的其他数据。对任意的 `Core Video` 缓冲类型都可以使用 `CVBuffer` 的 API.

* 图像缓冲（image buffer）是一种抽象缓冲类型，它被特定的用于保存视频图像（或视频帧）。像素缓冲（pixel buffer）和 `OpenGL` 缓冲（OpenGL buffer）继承自图像缓冲
* pixel buffer 用于在主存（main memory，内存) 存储一幅图像
* Core Video OpenGL buffer 是对一个标准的 OpenGL buffer （或 pbuffer）的包装。它将数据保存在显卡内存里
* Core Video OpenGL texture 是对一个标准的 OpenGL texture 的包装，它是保存在显卡内存里的一幅不可变的图像数据。它继承自 pixel buffer 或 OpenGL buffer（实际上存储帧数据的地方）。一个纹理必须被包装到一个原始类型（矩形或球体）上才可用于显示。

通常来说，当使用缓冲的时候，使用缓冲池来管理缓冲是很有益的。缓冲池分配若干缓冲以便后续需要的时候重用。它的优势在于系统不用花费额外的时间用于分配和回收内存，当你释放一个缓冲的时候，它会被放回缓冲池里。你可以在主内存里分配像素缓冲池（pixel buffer pools），在显卡内存里分配 `OpenGL` 缓冲池。

你可以把缓冲池看成是一个公司的车队，员工在需要的时候可以从车队里取走一辆车并在使用完之后归还回车队。这样做的话相比每次买车卖车可以有更小的开销。为了最大化利用资源，车的数量可以根据需求动态调整。

类似的，你应该使用一个纹理缓存（texture cache）来分配 `OpenGL` 纹理缓冲，它持有若干个可重用的纹理缓冲。

图 1-3 展示了处理 `QuickTime` 影片数据时一种可能的实现：使用若干缓冲和缓冲池来存储视频数据，用于存储压缩的文件数据以及最终显示到显示器上的图像数据。

图 1-3 解压和处理 `QuickTime` 视频帧

![](/images/recording_frames.png)

视频帧处理步骤为：

* `QuickTime` 提供将要被转换为独立视频帧的视频数据流
* 使用指定的解码器解压帧数据。像素缓冲池（pixel buffer pool）用来持有用于渲染成独立帧数据的关键帧和 B 帧等数据。
* 独立的帧数据会被作为 `OpenGL` 纹理存储在显卡内存里。对这个帧额外的图像处理（比如反交错）会在这里完成，并将处理的结果存储在一个 `OpenGL` 缓冲（buffer）里。
* 当你向 `Core Video` 请求一个帧数据的时候（响应 display link 的回调），这个 `OpenGL` 缓冲会被转换成一个 `OpenGL` 纹理传给你

### 一个帧数据都有什么

通常一个视频帧会包含一些便于系统显示的信息。在 `Core Video` 里，这些信息被作为附件关联到一个视频帧上。附件是用于表示不同类型数据的 Core Foundation 对象，比如下面几种常见的视频属性：

* 清除光圈（clean aperture）和 优先清除光圈（preferred clean aperture）。视频处理（例如添加滤镜）会在视频帧的边缘产生 artifacts 效果。为了避免这种影响，多数视频图像数据会保存比用于显示更多的屏幕数据以便简单的裁剪掉这些边缘。压缩视频时，优先清除光圈就是被推荐的裁剪数据被保存到视频里。清除光圈是显示时的实际裁剪信息。
* 颜色空间（color space）。它用于表示一张图像的模型，比如 RGB 或 YCbCr. 由于大多数模型都是使用若干个可以映射到一个空间上的点的参数，故而得名颜色空间。比如说， RGB 颜色空间使用红绿蓝三个参数，这三个参数的每种可能的组合都可以映射到一个三位空间里。
* Square Pixel VS Rectangular Pixel. 电脑上数字视频一般使用 Square Pixel 而电视上使用 Rectangular Pixel，所以如果你要创建用于广播的视频时需要为这个差异添加补偿
* 伽马值。伽马值是一种“软糖因子”，用于匹配输出硬件的输出到我们人眼期望看到的。比如说，显示器的电压与颜色强度比通常是非线性的，给“蓝色”信号电压加倍并不一定能使图片看起来有两倍蓝。伽马值就是这种用来匹配输入输出的曲线指数
* 时间戳。通常表示成小时，分钟，秒以及小数部分。时间戳用于表示一个特定的视频帧在影片里出现的时间。小数部分的大小通常依赖于影片使用的时间基准。时间戳的使用使得区分特定的视频帧更加容易，并简化了音视频同步问题。

附件使用键值对来表示。你既可以使用预定义的键，比如 *Core Video Reference* 里阐述的，也可以使用自定义的键。如果你指明这个附件是可以被传播的，那么就可以很容易地将这些附件信息传递给接替的缓冲，比如说，从 pixel buffer 创建一个 `OpenGL` 纹理的时候。



