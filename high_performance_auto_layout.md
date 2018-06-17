## WWDC 2018: High Performance Auto Layout

### 优化建议

#### 不要删除和重新添加同样的约束

```swift
override func updateConstraints() {
    NSLayoutConstraint.deactivate(myConstraints) // 1.
    myConstraints.removeAll()
    
    let views = ["text1": text1, "text2": text2]
    myConstraints += NSLayoutConstraint.constraints(withVisualFormat: "H:|-[text1]-[text2]", options: [.alignAllFirstBaseline,], metrics: nil, views: views)
    myConstraints += NSLayoutConstraint.constraints(withVisualFormat: "V:|-[text1]-", options: [], metrics: nil, views: views)
    NSLayoutConstraints.activate(myConstraints) // 2.
    super.updateConstraints()
}
```

上面代码中的 1 和 2 相当于：

```swift
override func layoutSubviews() {
    text1.removeFromSuperview() // 1.1
    text1 = nil
    text1 = UILabel(frame: CGRect(x: 20, y: 20, width: 300, height: 30))
    self.addSubview(text1) // 1.2
    
    text2.removeFromSuperview() // 2.1
    text2 = nil
    text2 = UILabel(frame: CGRect(x: 340, y: 20, width: 300, height: 30))
    self.addSubview(text2) // 2.2
    
    super.layoutSubviews()
}
```

也就是说每次调整布局的时候都重复了删除、初始化、添加的工作，显然很浪费性能。那应该如何修改？

```swift
override func updateConstraints() {
    if self.myConstraints == nil {
        NSLayoutConstraint.deactivate(myConstraints) // 1.
        myConstraints.removeAll()
        
        let views = ["text1": text1, "text2": text2]
        myConstraints += NSLayoutConstraint.constraints(withVisualFormat: "H:|-[text1]-[text2]", options: [.alignAllFirstBaseline,], metrics: nil, views: views)
        myConstraints += NSLayoutConstraint.constraints(withVisualFormat: "V:|-[text1]-", options: [], metrics: nil, views: views)
        NSLayoutConstraints.activate(myConstraints) // 2.
        super.updateConstraints()
    }
}
```

也就是说，当约束为 nil 的情况下才进行创建添加的工作。

### Avoid constraint churn

* 避免删除所有的约束
* 固定的约束应该在开始添加，并确保只添加一次
* 只修改那些需要修改的约束
* 隐藏 views 而不是将他们从父视图删除

Auto Layout 内部使用一个布局引擎来计算每个视图以及子视图的 frame 信息，并且将这些布局信息缓存下来，另外它还跟踪视图间的依赖关系，当某个约束发生改变时，所有依赖这个这个约束的约束都会相应改变，而不相关的约束不会变化。

### 不要拒绝使用约束

很多用户印象里认为使用约束的代价是高昂的（因为以前 Auto Layout 性能真的不行啊。。。），因此会手动计算大量的布局信息，添加删除视图，其实这样做往往性能更差，尤其是在 iOS 12 上，所以我们应该尽量使用约束。

### Costs of other auto layout features

* Inequality：代价很小，跟 equality 没啥区别
* NSLayoutConstraint set constant：由于布局引擎有缓存功能，它将仅更新修改的这个约束值，因此代价也很小
* Priority

Xcode 10 将会新增了一个用于调试 Auto Layout 的 Instrument，举的例子跟上面代码类似，也是大量重复（churn）添加删除约束引起的性能问题。

## 延伸

StackOverflow 上有一个关于 Auto Layout 很经典的[提问](https://stackoverflow.com/questions/20609206/setneedslayout-vs-setneedsupdateconstraints-and-layoutifneeded-vs-updateconstra)可以学习参考，大概总结如下：

1. `setNeedsUpdateConstraints` 会确保下次调用 `updateConstraintsIfNeeded` 的时候会调用 `updateConstraints` 方法
2. `setNeedsLayout` 会确保下次调用 `layoutIfNeeded` 的时候会调用 `layoutSubviews`
3. 调用 `layoutSubviews` 时它会调用 `updateConstraintsIfNeeded`，结合上面两点可知，如果之前调用了 `setNeedsUpdateConstraints`，那么 `updateConstraints` 将会被调用，因此多数情况下你不需要手动调用 `updateConstraintsIfNeeded`
4. `updateConstraintsIfNeeded` 仅更新约束，它不会强制 UI 开始进行布局相关的工作，因此 view 的 frame 不会发生变化
5. 如果手动修改约束，调用 `setNeedsLayout` 就行了
6. 如果想要布局修改立刻产生效果，调用 `layoutIfNeeded`

![](/images/layout_cycle.png)


