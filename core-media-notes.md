### CMTimebase

这个数据类型的主要用途是同步。比如说分别解码并播放音频和视频，由于硬件的延时（latency）不可控，因此音视频的同步就比较麻烦，而使用 `CMTimebase` 就可以根据同一个 `host time`，也就是一个 CMClock 类型的数据，然后分别创建音频和视频的 `control timebase`，通过同步这两个 `control timebase` 就可以简单的同步音视频了。

