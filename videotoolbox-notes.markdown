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

* kVTCompressionPropertyKey_ProfileLevel
    * 现在主要用 H.264(AVC) 和 H.265(HEVC)
    * 对于 H.264 来说，主要有 Baseline/Main/High 三个 profile:
        * Baseline（最低Profile）级别支持 I/P 帧，只支持无交错（Progressive）和 CAVLC 熵编码模式，一般用于低阶或需要额外容错的应用，比如视频通话、手机视频等；
        * Main（主要Profile）级别提供 I/P/B 帧，支持无交错（Progressive）和交错（Interlaced），同样提供对于 CAVLC 和 CABAC 两种熵编码模式的支持，用于主流消费类电子产品规格如低解码（相对而言）的 mp4、便携的视频播放器、PSP 和 iPod 等；
        * High（高端Profile，也叫 FRExt）级别在 Main 的基础上增加了 8x8 内部预测、自定义量化、无损视频编码和更多的 YUV 格式（如4：4：4），用于广播及视频碟片存储（蓝光影片），高清电视的应用。

        >
        > 详细内容的可以参考[维基百科](https://en.wikipedia.org/wiki/H.264/MPEG-4_AVC#Profiles)。

* kVTCompressionPropertyKey_H264EntropyMode
    * kVTH264EntropyMode_CAVLC: Context-based Adaptive Variable Length Coding, 基于上下文的自适应可变长编码
    * kVTH264EntropyMode_CABAC: Context-based Adaptive Binary Arithmetic Coding, 基于上下文的自适应二进制算术编码
    * CABAC 压缩效果更好，但占用资源也更高
    * 需要配合 kVTCompressionPropertyKey_ProfileLevel 使用，因为不同的 profile 支持的熵编码模式不同，如果不兼容则会引起编码错误
* kVTCompressionPropertyKey_MaxH264SliceBytes
    * 表示允许的一个帧切片的最大大小，单位为字节，0 表示无限制
    * 一个视频帧可以被分为一个或多个切片，slice，这个话题可以参考 NALU
* kVTCompressionPropertyKey_ExpectedFrameRate
    * 这个值不是用来控制帧率，而只是给 compression session 一个提示，用于它的内部配置
    * 如果要控制帧率，应该设置 AVCaptureConnection 的 videoMinFrameDuration/videoMaxFrameDuration
* kVTCompressionPropertyKey_CleanAperture
* kVTCompressionPropertyKey_PixelAspectRatio
* kVTCompressionPropertyKey_FieldCount
* kVTCompressionPropertyKey_FieldDetail
* kVTCompressionPropertyKey_AspectRatio16x9
* kVTCompressionPropertyKey_ProgressiveScan
* kVTCompressionPropertyKey_ColorPrimaries
* kVTCompressionPropertyKey_TransferFunction
* kVTCompressionPropertyKey_YCbCrMatrix

##### Per-Frame Configuration

上面的这些设置都是针对 compression session 整体设置的，而下面的设置是针对一个指定的帧的：

* kVTEncodeFrameOptionKey_ForceKeyFrame
    * 强制将这一帧编码为关键帧
    * 但是这个请求 compression session 不一定总是能满足的，也就是说你设置将某一帧编码为关键帧，但是由于某些限制，这一帧无法编码为关键帧

### VTCompressionSession

VTCompressionSession 是一个 CoreFoundation 对象，通常它的生命周期如下：

1. 创建：VTCompressionSessionCreate
2. 设置：VTSessionSetProperty
3. 准备编码：VTCompressionSessionPrepareToEncodeFrames
4. 编码：VTCompressionSessionEncodeFrame
5. 完成：VTCompressionSessionCompleteFrames
6. 销毁：VTCompressionSessionInvalidate

### VTDecompressionProperties

* kVTDecompressionPropertyKey_OutputPoolRequestedMinimumBufferCount
    * 设置 decompression session 创建 CVPixelBufferPool 时的最小缓冲区个数
    * 一般来说不用设置这个值，让 decompression session 自己决定就好了
    * 置为 0 或 nil 会清除这个设置（也就是让 decompression session 自己决定）
    * 如果在 decompression session 使用过程中设置了这个值则会重新创建 CVPixelBufferPool，并且原先的 CVPixelBuffer 会被销毁
* kVTDecompressionPropertyKey_MinOutputPresentationTimeStampOfFramesBeingDecoded
* kVTDecompressionPropertyKey_MaxOutputPresentationTimeStampOfFramesBeingDecoded

> 一个 decompression session 可能在同时解码多个 frame，上面两个值表示这些 frame 中 presentation timestamp 最小的和最大的。

### VTDecompressionSession

与 VTCompressionSession 类似，VTDecompressionSession 也有下面这样的生命周期：

1. 创建：VTDecompressionSessionCreate
2. 设置：VTSessionSetProperty
3. 解码：VTDecompressionSessionDecodeFrame
4. 完成：VTDecompressionSessionFinishDelayedFrames
5. 销毁：VTDecompressionSessionInvalidate

需要注意的是，VTDecompressionSessionFinishDelayedFrames 可能会在解完所有未解码帧之前返回，所以如果想要确保在解码完所有帧后返回，应该用 VTDecompressionSessionWaitForAsynchronousFrames 代替 VTDecompressionSessionFinishDelayedFrames。

如果解码过程中，有些帧的格式发生了较小的变化，可以使用 VTDecompressionSessionCanAcceptFormatDescription 来测试原来的 decompression session 是否可以兼容新格式，这样可以避免创建新的解码会话。



