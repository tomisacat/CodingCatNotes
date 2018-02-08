# 性能/省电/卡顿优化

### 分析

1. 使用 Xcode 调试面板看 CPU/Network/GPU/Overhead/Disk/Radio(WiFi/Bluetooth)/Location 等方面的消耗，找到消耗较大的部分
2. 在手机上打开开发者选项里的 Logging -> Energy/Network，然后在手机不插电的情况下运行 app 几分钟，然后用 Instruments 分析刚才记录的日志

### 优化

主要从 CPU/Network/GPU/Location/Bluetooth/IO 方面来下手。

1. CPU
    * 尽量减少刷新，能刷新一行就不要用 reloadData
    * 使用合适的数据结构或者容器，比如说数组适合按 index 查找，字典适合按 key 查找，Set 适合无序但 Item 唯一的场景
    * lazy initialization
    * 大量相同对象尽量重用，比如内存池，线程池
    * 缓存，比如图片缓存，或者 Cell 高度缓存等
    * 线程或队列选用合适的 QoS（以前是 queue priority)
    * 少使用定时器，尽量使用事件通知而不是定时器轮询，比如文件修改可以用 dispatch source
    * 减少后台任务，如果有，执行完后通知系统
2. GPU
    * 少用圆角。比如 maskToBounds/clipToBounds，因为这会引起 off-screen rendering
    * 少用透明或半透明，原因同上
3. Location/Bluetooth
    * 用完就关掉
    * 减小精度
4. Network
    * HTTP/2. 复用连接
    * DNS 解析
    * 传输数据压缩
    * Cache
5. IO
    * 使用缓冲区（buffer），批量读写
    * 小数据读写考虑使用数据库代替文件


### 参考

* [Energy Efficiency Guide for iOS Apps](https://developer.apple.com/library/content/documentation/Performance/Conceptual/EnergyGuide-iOS/WorkLessInTheBackground.html#//apple_ref/doc/uid/TP40015243-CH22-SW1)
* [微信读书 iOS 性能优化总结](http://wereadteam.github.io/2016/05/03/WeRead-Performance/)
* [iOS实时卡顿监控](http://www.tanhao.me/code/151113.html/)
* [iOS监控：卡顿检测](http://ios.jobbole.com/93085/)
* [iOS应用UI线程卡顿监控](https://mp.weixin.qq.com/s?__biz=MzI5MjEzNzA1MA==&mid=2650264136&idx=1&sn=052c1db8131d4bed8458b98e1ec0d5b0&chksm=f406837dc3710a6b49e76ce3639f671373b553e8a91b544e82bb8747e9adc7985fea1093a394#rd)
* [深入剖析 iOS 性能优化](https://ming1016.github.io/2017/06/20/deeply-ios-performance-optimization/)

