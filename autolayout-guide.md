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
* base line(first/last)
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


