### Haptic Engine 不工作

当 `Haptic Engine` 与音频录制一同工作时会失效，比如，当 `AVCaptureSession` 添加了一个 `.audio` 类型的 `AVCaptureDeviceInput`，一个可行的 workaround 就是在视频预览的时候不添加 `.audio` 输入，这使得你可以正常使用 `Haptic Engine`；当开始录制的时候再添加 `.audio` 输入。

### UIView 的 opaque 属性

`opaque`属性，`UIView`的默认值是`true`，但`UIButton`等子类的默认值都是`false`。`opaque`表示当前`UIView`是否不透明，不过搞笑的是事实上它却决定不了当前`UIView`是不是不透明，比如你将`opaque`设为`false`，该`UIView`照样是可见的（是否可见是由`alpha`或`hidden`属性决定的），照理说为`false`就表示透明，那就应该是不可见的呀？

显示器中的每个像素点都可以显示一个由RGBA颜色空间组成的色值，比如上图中有红色和绿色两个图层色块，对于没有交叉的部分，即纯红色和绿色部分来说，对应位置的像素点只需要简单的显示红或绿，对应的RGBA为（1，0，0，1）和（0，1，0，1）就行了，负责图形显示的GPU需要很小的计算量就可以确定像素点对应的显示内容。
问题是红色和绿色还有相交的一块，其相交的颜色为黄色。这里的黄色是怎么来的呢？原来，GPU会通过图层一和图层二的颜色进行图层混合，计算出混合部分的颜色，最理想情况的计算公式如下：
R = S + D * ( 1 – Sa )
其中，R表示混合结果的颜色，S是源颜色(位于上层的红色图层一)，D是目标颜色(位于下层的绿色图层二)，Sa是源颜色的alpha值，即透明度。公式中所有的S和D颜色都假定已经预先乘以了他们的透明度。
知道图层混合的基本原理以后，再回到正题说说opaque属性的作用。当UIView的opaque属性被设为YES以后，按照上面的公式，也就是Sa的值为1，这个时候公式就变成了：
R = S
即不管D为什么，结果都一样。因此GPU将不会做任何的计算合成，不需要考虑它下方的任何东西(因为都被它遮挡住了)，而是简单从这个层拷贝。这节省了GPU相当大的工作量。由此看来，opaque属性的真实用处是给绘图系统提供一个性能优化开关！

按照前面的逻辑，当`opaque`属性被设为`true`时，GPU就不会再利用图层颜色合成公式去合成真正的色值。因此，如果`opaque`被设置成`true`，而对应`UIView`的`alpha`属性不为1.0的时候，就会有不可预料的情况发生，这一点苹果在官方文档中有明确的说明：
An opaque view is expected to fill its bounds with entirely opaque content—that is, the content should have an alpha value of 1.0. If the view is opaque and either does not fill its bounds or contains wholly or partially transparent content,the results are unpredictable. You should always set the value of this property to NO if the view is fully or partially transparent.

### position/anchorPoint/center/frame/bounds

position = center 也就是说 layer.position 就是 view.center

position 是 anchorPoint 那个点在 superView(center)/superLayer 的位置
position 和 anchorPoint 互相不影响，他们只影响 frame：

> position.x = frame.origin.x + anchorPoint.x * bounds.size.width  
position.y = frame.origin.y + anchorPoint.y * bounds.size.height

> frame.origin.x = position.x - anchorPoint.x * bounds.size.width  
frame.origin.y = position.y - anchorPoint.y * bounds.size.height

> position是layer中的anchorPoint点在superLayer中的位置坐标。position点是相对superLayer的，anchorPoint点是相对layer的，两者是相对不同的坐标空间的一个重合点。

> anchorPoint和position互不影响，受影响的只有frame

frame 相当于一个 computed property, setFrame 的时候改变了 center 和 bounds.size, 获取 frame 的时候就从 center 和 bounds 来得到结果。

在使用 autolayout 的情况下，如果 affineTransform != .identity，那么 view.frame 是 undefined 的。

