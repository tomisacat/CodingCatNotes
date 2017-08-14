### VTCompressionProperties

* kVTCompressionPropertyKey_MaxKeyFrameInterval
    * 关键帧间隔，默认值为 0，表示由编码器决定
* kVTCompressionPropertyKey_MaxKeyFrameIntervalDuration
    * 最大关键帧间隔时长，单位为秒
    * 默认为 0，表示由编码器决定

> 以上两个 property 可以同时使用，最后编码时会同时满足这两个要求

* kVTCompressionPropertyKey_AllowTemporalCompression
    * 默认为 true
    * 若设为 false，则表示指定编码器将所有帧编码为关键帧（也就是所有帧都独立存在，没有帧间预测）
* kVTCompressionPropertyKey_AllowFrameReordering
    * 默认为 true，这使得它允许编码器编码 B 帧
    * 若为 true，则某个帧的存储顺序（the decode order) 和它被传递给编码器的顺序（the display order) 不一样
* kVTCompressionPropertyKey_AverageBitRate
    * 单位为 bit
    * 不是硬性限制，峰值的时候可能会高于这个值
    * 默认为 0，编码器自己决定
* kVTCompressionPropertyKey_DataRateLimits
    * 硬性限制
    * 值为个数为偶数的数组，每个限制都由一对值表示：数据大小（单位为字节）和时长（单位为秒）表示，如 (2423947842, 1)；数值对按照优先级排列

> 以上两个 property 都有这样的限制：
>   
>   * 仅当帧的源数据提供了时间信息的时候这个设置才有效
>   * 有些编码器不一定支持指定的比特率
> 
> 事实上，kVTCompressionPropertyKey_AverageBitRate 表示整体平均比特率，是非硬性限制，可能有时候峰值会大于这个设置，而 kVTCompressionPropertyKey_DataRateLimits 里的每一对设置是硬性限制，假设这个数组里第一个数值对为 (213232213, 1)，表示每秒钟的比特率最大为 213232213，然而在1秒内仍然有可能在某个瞬间的比特率大于这个值。可以参考[官方文档](https://developer.apple.com/library/content/qa/qa1958/_index.html)。




