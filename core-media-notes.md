### CMTimebase

这个数据类型的主要用途是同步。比如说分别解码并播放音频和视频，由于硬件的延时（latency）不可控，因此音视频的同步就比较麻烦，而使用 `CMTimebase` 就可以根据系统的 `host time`，也就是一个 CMClock 类型的数据，然后创建音频 `timebase`，并根据这个 `timebase` 同步视频。

这里有几个关于这个数据类型的使用问题：

1. [Building a streaming video player for iOS WITHOUT AVPlayer](https://stackoverflow.com/questions/30518297/building-a-streaming-video-player-for-ios-without-avplayer)
2. [iOS correct way for update recording time since beginning of the recording](https://stackoverflow.com/questions/40384771/ios-correct-way-for-update-recording-time-since-beginning-of-the-recording)


