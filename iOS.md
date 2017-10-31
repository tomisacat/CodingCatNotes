### Haptic Engine 不工作

当 `Haptic Engine` 与音频录制一同工作时会失效，比如，当 `AVCaptureSession` 添加了一个 `.audio` 类型的 `AVCaptureDeviceInput`，一个可行的 workaround 就是在视频预览的时候不添加 `.audio` 输入，这使得你可以正常使用 `Haptic Engine`；当开始录制的时候再添加 `.audio` 输入。

