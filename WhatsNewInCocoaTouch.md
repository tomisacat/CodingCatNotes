## WWDC 2018: What's New in Cocoa Touch

### Scrolling

* 优化了 UITableView 的 prefetch 功能，更智能地安排 prefetch 时间，确保不在滑动的同时prefetch 后面 cell 的数据
* 由于我们掌握着从上到下的软件栈，所以可以从 UIKit 获取所有想要的信息，比如说滑动时正在发生哪些事情，将这个信息发送给底层 CPU 性能控制器，这样就可以更加智能的计算出所需要的 CPU 资源

### Memory

* Automatic Backing Store
    * 默认启用
    * UIView.draw()
    * UIGraphicsImageRenderer（Offscreen rendering)
    * 如果在使用 UIGraphicsImageRenderer 的时候确定不想要这个功能，可以通过设置 UIGraphicsImageRenderer.Range 来修改

### Auto Layout

* 对于下面几种情况都优化了算法，使得添加 View 所需要的时间呈线性增长
    * Independent Sibling Views：层级相同且相互独立的 Views
    * Dependent Sibling Views：层级相同且相互依赖的 Views
    * Nested Views：互相嵌套的 Views

### Framework Updates

Swiftication
* Nesting
    * Types
        
        ```swift
        // Swift 4
        enum UIApplicationState {
            case active
            case inactive
            case background
        }
        ```
        
        现在是：
        
        ```swift
        // Swift 4.2
        class UIApplication: UIResponder {
            enum State {
                case active
                case inactive
                case background
            }
        }
        ```
        
    * Constants
    * Functions
* NSSecureCoding

### API Enhancements

* Group Notification
* Message
* Automatic Passwords/Security Code AutoFill
* Siri Shortcut

