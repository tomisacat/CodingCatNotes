起初，Image I/O 属于 Core Graphics 框架的一部分，后来独立出来成为一个独立的框架（但是依然依赖 Core Graphics）。无论用哪种方式访问图像数据，最终都是通过 Image I/O 完成，因为它效率高，可以访问元数据，并且提供了颜色管理。它支持处理大部分的图像格式，包括 HDR 和 raw 类型的图像。

它主要通过以下两个类型来表示处理的目的：

* 读：**CGImageSourceRef**
* 写：**CGImageDestinationRef**

如何创建：

* CFURLRef
* CFDataRef/CFMutableDataRef
* CGDataConsumerRef/CGDataProviderRef

获取 Image I/O 支持的图像类型：

* CGImageSourceCopyTypeIdentifiers：它会返回一个元素类型为 UTI(Uniform Type Identifiers) 的数组，表示支持读取的图像类型
* CGImageDestinationCopyTypeIdentifiers：类似上面

```swift
let sources = CGImageSourceCopyTypeIdentifiers()
CFShow(sources)
        
let destinations = CGImageDestinationCopyTypeIdentifiers()
CFShow(destinations)
```

具体的类型可以在 MobileCoreService 框架下找到，主要有以下这些：

| UTI | 常量值 |
| :-: | :-: |
| public.image | kUTTypeImage |
| public.png | kUTTypePNG |
| public.jpeg | kUTTypeJPEG |
| public.jpeg-2000 (OS X only) | kUTTypeJPEG2000 |
| public.tiff | kUTTypeTIFF |
| com.apple.pict (OS X only) | kUTTypePICT |
| com.compuserve.gif | kUTTypeGIF |

### CGImageSourceRef

* 获取一个 gif 动图里图片总数和第一张图片的示例代码：

    ```swift
    let path = Bundle.main.path(forResource: "IMG_5959", ofType: "GIF")
    let url = URL(fileURLWithPath: path!) as CFURL
    let options = [
        kCGImageSourceShouldCache: true,
        kCGImageSourceShouldAllowFloat: true
    ] as CFDictionary
    
    guard let imageSource = CGImageSourceCreateWithURL(url, options) else {
        return
    }
    
    let count = CGImageSourceGetCount(imageSource)
    print(count)
            
    let firstImage = CGImageSourceCreateImageAtIndex(imageSource, 0, nil)
    imageView.image = UIImage(cgImage: firstImage!)
    ```
    
    由于 Image I/O 使用到的类型大多是 CoreFoundation 类型，在 Swift 里被转换为各种UnsafePointer， 使用起来真的是很蛋疼，这里一个偷懒的技巧就是使用 Foundation 或 Swift 原生类型，然后使用 as 操作转换为 CoreFoundation 类型。比如上面的 Dictionary -> CFDictionary
    
* 缩略图
    
    类似的，可以使用 `CGImageSourceCreateThumbnailAtIndex` 方法创建缩略图。，可以在创建的 option 参数里添加 `kCGImageSourceCreateThumbnailFromImageIfAbsent` 参数键值对，它可以在原图本身没有缩略图的情况下自动创建，默认大小为原图大小，所以如果你想要控制创建出来的缩略图的大小还可以添加 `kCGImageSourceThumbnailMaxPixelSize` 来控制创建出来的缩略图的最大像素值。
    
* 属性

    可以使用 `CGImageSourceCopyProperties` 获取一个 image source 的各项属性值，比如一个 gif 图片可能的输出为：
    
    ```
    <CFBasicHash 0x1c4463080 [0x1b5085310]>{type = immutable dict, count = 2,
entries =>
	0 : <CFString 0x1adbb68d0 [0x1b5085310]>{contents = "{GIF}"} = <CFBasicHash 0x1c4464440 [0x1b5085310]>{type = mutable dict, count = 2,
entries =>
	1 : <CFString 0x1adbb8090 [0x1b5085310]>{contents = "HasGlobalColorMap"} = <CFBoolean 0x1b5085868 [0x1b5085310]>{value = true}
	2 : <CFString 0x1adbb8030 [0x1b5085310]>{contents = "LoopCount"} = <CFNumber 0xb000000000000002 [0x1b5085310]>{value = +0, type = kCFNumberSInt32Type}
}

	1 : <CFString 0x1adbb6ed0 [0x1b5085310]>{contents = "FileSize"} = <CFNumber 0xb0000000017cf803 [0x1b5085310]>{value = +1560448, type = kCFNumberSInt64Type}
}
    ```
    
    可以看出，这个 gif 动图有一个全局颜色映射表，循环次数（0 表示无限循环），以及图片文件大小。
    
* 元数据

    使用 `CGImageSourceCopyMetadataAtIndex` 获取元数据。图片的 EXIF 信息就存在里面。
    
* 渐进式加载

    主要就是不断地 feed
    
### CGImageDestinationRef

主要的几个参数：

* kCGImageDestinationBackgroundColor：当要写入的图像数据有 alpha 通道，而 destination 图像格式并不支持 alpha 通道时，可以用这个参数来混合。默认是白色

##### 创建动态 PNG 图片

```swift
let loopCount = 1
let frameCount = 60
 
var fileProperties = NSMutableDictionary()
fileProperties.setObject(kCGImagePropertyPNGDictionary, forKey: NSDictionary(dictionary: [kCGImagePropertyAPNGLoopCount: frameCount]))
 
var frameProperties = NSMutableDictionary()
frameProperties.setObject(kCGImagePropertyPNGDictionary, forKey: NSDictionary(dictionary: [kCGImagePropertyAPNGDelayTime: 1.0 / Double(frameCount)]))
 
guard let destination = CGImageDestinationCreateWithURL(fileURL, kUTTypePNG, frameCount, nil) else {
    // Provide error handling here.
}
 
CGImageDestinationSetProperties(destination, fileProperties.copy() as? NSDictionary)
 
for i in 0..<frameCount {
    autoreleasepool {
        let radians = M_PI * 2.0 * Double(i) / Double(frameCount)
        guard let image = imageForFrame(size: CGSize(width: 300, height: 300)) else {
            return
        }
        
        CGImageDestinationAddImage(destination, image, frameProperties)
    }
}
 
if !CGImageDestinationFinalize(destination) {
    // Provide error handling here.
}
```




