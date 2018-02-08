在基于 Auto Layout 布局的情况下，UI 主要受 External/Internal Changes 影响：

### External Changes

也就是外部变化，主要指父视图变化(size，shape)引起的变化，包括：

* 用户改变了窗口的大小（macOS）
* iPad 上使用 Split View
* iDevice 旋转
* 打电话和录音时显示在顶部的条
* 支持不同的 size classes
* 不同的屏幕尺寸

iPad 上支持 Slide Over 和 Split View 也主要靠 Auto Layout

### Internal Changes

也就是控件本身尺寸的变化，主要有：

* 显示的内容发生了变化 （text，image）
* 国际化
* Dynamic Types（iOS，主要指字体）

## UIStackView 

使用 UIStackView 主要注意 distribution 和 alignment 两个属性：

* distribution：规定沿 axis 轴线时的布局
* alignment：规定与 distribution 垂直轴线时的布局

remove 的时候要同时给 subview 发送 removeFromSuperview 消息，否则这个 subview 只是从 arrangedSubviews 里移除但还在 UIStackView 的 subviews 里，也就是还会渲染在视图里，这时候它的大小就是它的 intrinsicContentSize。

### Attribute

* top
* bottom
* left/leading
* right/trailing
* width
* height
* centerX/centerY(within margins)
* base line(first/last)
* margins(left/right/top/bottom/leading/trailing)
* notAnAttribute

公式：
> AView.Leading = BView.trailing * multiplier + constant

对于这些 attributes 来说有以下规则：

* 不能用 location attributes（left/right etc.）来设置 size attributes（width/height）。比如说，你不能设置 subview 的 width 约束值等于 subview 的 left 约束值
* 不能用定值来设置 location attributes，也就是说 location attributes 必须依赖另一个 attributes
* 使用 location attributes 的时候，multiplier 只能为 1.0，例如 AView.Leading = BView.trailing * 1.0 + 8.0
* 对于 location attributes 来说，不能用 vertical attributes 来约束 horizontal attributes（反之也一样）。也就是说你不能设置 AView.leading = BView.top * 1.0
* 对于 location attributes 来说，不能用 leading/trailing 来设置 left/right（两种不同的体系）

### Priority

* low: 250
* medium: 500
* high: 750
* required: 1000

### Intrinsic Content Size

| View | Intrinsic Content Size |
| :-: | :-: |
| UIView/NSView | NO |
| Slider | iOS: only width; macOS: width/height or both |
| label/button/switch/text field | width and height |
| text view/image view | vary |


Label/Button: based on text and font
Image View: based on image size
Text View: based on content/scrolling enabled/other constraints

### Content Hugging/Compression Resistance

Content Hugging: 向内收缩，使视图紧贴内容
Compression Resistance： 向外撑开，避免内容被裁剪

![](/images/intrinsic_content_size_2x.png)

对于 view 来说，默认的 content hugging 是 250，compression resistance 是 750，也就是 view 更易于拉伸而不是收缩。

### Layout Margins/Layout Guide

UIView 有一个 layoutMargins 属性，是 UIEdgeInsets 类型，同时它还有一个 layoutMarginsGuide 属性，是 UILayoutGuide 类型，这个类型主要是用来辅助我们进行布局的，它本身并不渲染在视图的内容里，它主要有以下属性：

* bottomAnchor
* centerXAnchor
* centerYAnchor
* heightAnchor
* widthAnchor
* leadingAnchor
* leftAnchor
* rightAnchor
* topAnchor
* trailingAnchor

iOS 11 修改了很多 api，其中 UIViewController 的 topLayoutGuide 和 bottomLayoutGuide 就被 view 的 safeAreaLayoutGuide 取代了。

### Readable Content Guides

view 的 readableContentGuide 属性定义了这个 view 里显示文本内容的最优宽度。理想情况下，它会尽量压缩使你一次看到尽可能多的内容。它的大小主要由系统字体的 dynamic type 来定。

> 大多数设备上这个属性和 layoutMargins 没区别，主要在 iPad Landscape 模式下有区别。

### Semantic Content

view 有一个 semanticContentAttribute 属性，它决定了当语言从 left-to-right 变成 right-to left 时视图内容是否翻转。

### NSLayoutAnchor

view 的属性，主要有：

* topAnchor
* bottomAnchor
* ...

其实很类似 UILayoutGuide，因此，UILayoutGuide 就像是不渲染到视图里的 view 一样。

### ScrollView

* scroll view 通过修改 bounds.origin 来实现内容的滚动
* 当 scroll view 使用 auto layout 的时候，它的 top/bottom/left/right 实际上是指它内部一个 ‘content view’ 的边界
* 它的 contentSize 由 subview 决定，并且只有当 subviews 的约束能够确定一个具体的 size 时 scroll view 的约束才能是合法的
* 要想使用 auto layout 来调整 scroll view 的 frame，那么它的约束必须满足：要么明确定义 scroll view 的 width/height，要么 scroll view 的边界（edges）必须和 subview 以外的视图产生约束

[官方文档](https://developer.apple.com/library/content/technotes/tn2154/_index.html)
[里脊串](http://adad184.com/2015/12/01/scrollview-under-autolayout/)

### Self-Sizing Table View Cells

```swift
tableView.estimatedRowHeight = 85.0
tableView.rowHeight = UITableViewAutomaticDimension
```

### LayoutGuide

对于 iOS 11 来说:

> safeAreaLayoutGuide 的 top 在状态栏下面，bottom 就是 view 的 bottom

对于 iOS 10 以及之前来说：

> 1. topLayoutGuide 的 top 就是屏幕顶部，bottom 在状态栏下面
> 2. bottomLayoutGuide 的 top 和 bottom 一样，都是屏幕底部

