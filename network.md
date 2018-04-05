# Network

### URLSession

1. 默认是**线程安全的**
2. 如果没有指定 delegate，它默认会使用一个系统提供的 delegate 来处理各种事件
3. URLSession 主要会创建以下 task：
    * URLSessionTask
    * URLSessionDataTask -> URLSession:dataTask:didReceiveData:
    * URLSessionUploadTask -> file/data or URLSession:task:needNewBodyStream:
    * URLSessionDownloadTask -> URLSession:downloadTask:didFinishDownloadingToURL: (after finishing writing to a temp file)
    * URLSessionStreamTask: This allows for **direct** TCP/IP connection to a given host and port with optional secure handshaking and navigation of proxies
4. shared: 共享使用全局的 URLCache/HTTPCookieStorage/URLCredentialStorage
5. delegate: 如果初始化 session 的时候指定了 delegate，那么 session 将持有（retain）这个delegate 对象直到收到 `URLSession:didBecomeInvalidWithError:` 消息
6. invalidateAndCancel 与 finishTasksAndInvalidate 类似，但它会向所有未完成的 task 对象发送 `cancel` 消息
7. `getTasksWithCompletionHandler` 获取当前未完成的 `data tasks`、`upload tasks` 和 `download tasks`
8. iOS 9 之后可以 `getAllTasksWithCompletionHandler`


### URLSessionTask

1. 如果发生服务器重定向，`originalRequest` 和 `currentRequest` 不一样
2. `progress`：iOS 11 开始支持
3. count:
    * countOfBytesClientExpectsTo(Send/Receive)：预估值，默认为 NSURLSessionTransferSizeUnknown
    * countOfBytes(Received/Sent/ExpectedToSend/ExpectedToReceive)。预计发送的大小一般由请求头（request header) 的 `Content-Length` 得到，而预估接收大小则由响应头（response header）里的 `Content-Length` 得到
4. suspend/resume: 挂起的时候，超时计时器也会暂停
5. task、data task 和 upload task 没有区别，只是名字不同
6. stream task 既可以用 URLSession 直接创建，也可能由 `URLSession:dataTask:didReceiveResponse:` 方法在处理响应（response）时创建
    * 异步串行读写（asynchronous, enqueued and executed serially
    * 向 `URLSessionTask` 发送 `captureStreams` 消息可以创建 InputStream/OutputStream，在创建之前会先完成所有未完成的读写任务，创建完之后这个 stream task 就被认为已经完成任务，不再接收任何消息（改由 session delegate 接收）。
    * 读请求一旦错误，后续读请求也会立刻抛出错误
    * 写请求执行完成回调时并不表示远端（服务器）已经收到全部数据，它仅表示所有数据已经写入内核（kernel space）

### URLSessionConfiguration

1. 系统提供了三个预置的 configuration：
    * default
    * ephemeral
    * background
2. 缓存设置
    *
3. **超时设置**
    * timeoutIntervalForRequest 当有数据传输时会重置
    * timeoutIntervalForResource
4. discretionary：对于 background 配置，如果为 true 表示由系统安排请求的发起，用于优化性能
5. sessionSendsLaunchEvents：对于 background 配置，如果为 true 则表示当请求完成或者需要验证（authentication）时唤醒 app
6. SSL
    * tlsMinimumSupportedProtocol
    * tlsMaximumSupportedProtocol
7. HTTP 设置
    * httpShouldUsePipelining
    * httpShouldSetCookies
    * httpCookieAcceptPolicy
    * httpAdditionalHeaders
    * httpMaximumConnectionsPerHost
    * httpCookieStorage
8. URL 设置
    * urlCredentialStorage
    * urlCache

### URLSessionDelegate

* urlSession(_ session: URLSession, didBecomeInvalidWithError error: Error?)
* urlSession(_ session: URLSession, didReceive challenge: URLAuthenticationChallenge, completionHandler: @escaping (URLSession.AuthChallengeDisposition, URLCredential?) -> Swift.Void)

    > 有些鉴权类型可以被应用到多个请求上，例如 SSL
* urlSessionDidFinishEvents(forBackgroundURLSession session: URLSession)

##### URLSessionTaskDelegate

* urlSession(_ session: URLSession, task: URLSessionTask, willBeginDelayedRequest request: URLRequest, completionHandler: @escaping (URLSession.DelayedRequestDisposition, URLRequest?) -> Swift.Void)
    
    > iOS 11 新加入的，当一个被延迟执行的 task 开始执行的时候会先调用这个代理方法。如果修改了 request，那么新的 `request.allowCellularAccess` 属性会被忽略，继续使用原来请求的这个属性值
* urlSession(_ session: URLSession, taskIsWaitingForConnectivity task: URLSessionTask)

    > 如果一个 task 的 `waitsForConnectivity` 属性置为 true 且当前网络不好时会调用这个方法；但如果是一个 background session 则会忽略这个属性（也就不会执行这个代理方法）
* urlSession(_ session: URLSession, task: URLSessionTask, willPerformHTTPRedirection response: HTTPURLResponse, newRequest request: URLRequest, completionHandler: @escaping (URLRequest?) -> Swift.Void)

    > 如果传一个 nil 给 completionHandler，那么新请求 newRequest 将会把 response 的 body 部分作为负载（payload）
    > background session 不会调用这个方法，直接执行默认的重定向操作
* urlSession(_ session: URLSession, task: URLSessionTask, didReceive challenge: URLAuthenticationChallenge, completionHandler: @escaping (URLSession.AuthChallengeDisposition, URLCredential?) -> Swift.Void)

    > 如果没有实现这个方法，那么也不会调用 session 的鉴权方法，直接使用默认的系统处理方式
* urlSession(_ session: URLSession, task: URLSessionTask, needNewBodyStream completionHandler: @escaping (InputStream?) -> Swift.Void)
* urlSession(_ session: URLSession, task: URLSessionTask, didSendBodyData bytesSent: Int64, totalBytesSent: Int64, totalBytesExpectedToSend: Int64)

    > 定期调用这个方法来通知上传进度
    > task 类也有相应的属性获取这些值
* urlSession(_ session: URLSession, task: URLSessionTask, didFinishCollecting metrics: URLSessionTaskMetrics)
* urlSession(_ session: URLSession, task: URLSessionTask, didCompleteWithError error: Error?)

##### URLSessionDataDelegate

* urlSession(_ session: URLSession, dataTask: URLSessionDataTask, didReceive response: URLResponse, completionHandler: @escaping (URLSession.ResponseDisposition) -> Swift.Void)

    > 收到 response
    > background 上传任务不会调用这个方法
* urlSession(_ session: URLSession, dataTask: URLSessionDataTask, didBecome downloadTask: URLSessionDownloadTask)
* urlSession(_ session: URLSession, dataTask: URLSessionDataTask, didBecome streamTask: URLSessionStreamTask)
* urlSession(_ session: URLSession, dataTask: URLSessionDataTask, didReceive data: Data)

    > delegate 应该强引用而不是拷贝 data 
    > 由于数据可能是不连续的，应该使用 Data 的 `enumerateBytes` 方法访问数据
* urlSession(_ session: URLSession, dataTask: URLSessionDataTask, willCacheResponse proposedResponse: CachedURLResponse, completionHandler: @escaping (CachedURLResponse?) -> Swift.Void)

##### URLSessionDownloadDelegate

* urlSession(_ session: URLSession, downloadTask: URLSessionDownloadTask, didFinishDownloadingTo location: URL)

    > 当这个方法执行完，文件将会被删除，所以你应该在这个方法里将文件拷贝到一个合适的地方
* urlSession(_ session: URLSession, downloadTask: URLSessionDownloadTask, didWriteData bytesWritten: Int64, totalBytesWritten: Int64, totalBytesExpectedToWrite: Int64)
* urlSession(_ session: URLSession, downloadTask: URLSessionDownloadTask, didResumeAtOffset fileOffset: Int64, expectedTotalBytes: Int64)

##### URLSessionStreamDelegate

* urlSession(_ session: URLSession, readClosedFor streamTask: URLSessionStreamTask)
* urlSession(_ session: URLSession, writeClosedFor streamTask: URLSessionStreamTask)
* urlSession(_ session: URLSession, betterRouteDiscoveredFor streamTask: URLSessionStreamTask)
* urlSession(_ session: URLSession, streamTask: URLSessionStreamTask, didBecome inputStream: InputStream, outputStream: OutputStream)

##### 几个枚举

1. DelayedRequestDisposition
    * continueLoading
    * useNewRequest
    * cancel
2. AuthChallengeDisposition
    * useCredential
    * performDefaultHandling
    * cancelAuthenticationChallenge
    * rejectProtectionSpace
3. ResponseDisposition
    * cancel: 跟 task.cancel() 相同
    * allow
    * becomeDownload
    * becomeStream

### URL

1. 属性
    * absoluteString/relativeString/baseURL/absoluteURL/relativePath
    * scheme
    * resourceSpecifier
    * host
    * port
    * user/password
    * fragment
    * parameterString
    * query
2. 一条标准的 URL 应该是这样的：scheme:resourceSpecifier

### URLRequest

1. 有几个只读属性（跟 URLSessionConfiguration 配置相关）
    * timeoutInterval
    * url
    * cachePolicy
    * networkServiceType
    * allowCellularAccess： 是否允许使用蜂窝网络
    * httpMethod
    * allHttpHeaderFields： 请求头
    * httpBody：Data 类型
    * httpBodyStream：InputStream 类型，仅用于检查，直接操作流数据是不安全的
    * httpShouldHandleCookies
    * httpShouldUsePipelining：如果是 true 表示请求不必等待前一个响应接收完再传输后面的数据
    
2. 有对应的 mutable 类型，上面的属性也是 mutable

### URLResponse

初始化
> init(url URL: URL, mimeType MIMEType: String?, expectedContentLength length: Int, textEncodingName name: String?)

1. 只包含请求响应的元数据，不包含代表 URL 数据内容的实际数据
2. 与 protocol/scheme 无关。对于特性的协议，比如 HTTP，有对应的 HTTPURLResponse 类型
3. 属性
    * mimeType
    * url
    * expectedContentLength
    * textEncodingName: encoding，编码
    * 对于 HTTPURLResponse 来说还有：
        * statusCode
        * allHeaderFields：响应头

### CachedURLResponse

初始化
> init(response: URLResponse, data: Data)

1. policy:
    * allowed
    * allowedInMemoryOnly
    * notAllowed

### URLCache

1. shared：获取或者设置（var）全局共享的实例
2. 默认：
    * Memory: 4MB
    * Disk: 20MB
3. get/set/remove cached response for request/dataTask

### URLCredentialStorage

1. 单例（singleton）
2. get/set/remove/default credential for protection space
3. 第2条还有适用于特定 URLSessionTask 的版本

### URLProtectionSpace

初始化：(域名、端口、协议)
> init(host: String, port: Int, protocol: String?, realm: String?, authenticationMethod: String?)
>
> init(proxyHost host: String, port: Int, type: String?, realm: String?, authenticationMethod: String?)

1. 有以下几种类型：
    * NSURLProtectionSpaceHTTP
    * NSURLProtectionSpaceHTTPS
    * NSURLProtectionSpaceFTP
    * NSURLProtectionSpaceHTTPProxy
    * NSURLProtectionSpaceHTTPSProxy
    * NSURLProtectionSpaceFTPProxy
    * NSURLProtectionSpaceSOCKSProxy
2. 认证方法：
    * NSURLAuthenticationMethodDefault
    * NSURLAuthenticationMethodHTTPBasic
    * NSURLAuthenticationMethodHTTPDigest
    * NSURLAuthenticationMethodHTMLForm
    * NSURLAuthenticationMethodNTLM
    * NSURLAuthenticationMethodNegotiate
    * NSURLAuthenticationMethodClientCertificate （客户端证书）
    * NSURLAuthenticationMethodServerTrust （服务端）
3. 属性
    * realm：认证范围。通常只适用于 http 请求
    * host/port/protocol
    * authenticationMethod
    * serverTrust：服务端 SSL 事务状态

### URLCredential

1. persistence:
    * none
    * forSession
    * permanent
    * synchronizable
2. 初始化方法：
    * init(user: String, password: String, persistence: URLCredential.Persistence)
    * init(identity: SecIdentity, certificates certArray: [Any]?, persistence: URLCredential.Persistence)
    * init(trust: SecTrust)

### URLAuthenticationChallenge

初始化
> init(protectionSpace space: URLProtectionSpace, proposedCredential credential: URLCredential?, previousFailureCount: Int, failureResponse response: URLResponse?, error: Error?, sender: URLAuthenticationChallengeSender)
> 
> init(authenticationChallenge challenge: URLAuthenticationChallenge, sender: URLAuthenticationChallengeSender)

1. 属性
    * protectionSpace
    * propsoedCredential
    * previousFailureCount
    * failureResponse
    * error

### URLProtocol

**只能处理 Cocoa（Touch） 层的网络连接**

初始化
> init(request: URLRequest, cachedResponse: CachedURLResponse?, client: URLProtocolClient?)

1. 抽象类，需要自定义类来继承并实现它的几个方法才可以使用
2. 属性
    * client：URLProtocolClient 协议类型
    * request：URLRequest
    * cachedResponse
3. 重点需要实现的方法：
    * `class func canInit(with request: URLRequest) -> Bool`：必须实现
    * `class func canonicalRequest(for request: URLRequest) -> URLRequest`：应该保证相同的输入 request 都能得到相同的输出，因为缓存（URLCache）会检查 request
    * `class func requestIsCacheEquivalent(_ a: URLRequest, to b: URLRequest) -> Bool`：也是为缓存服务的。通常调用父类实现即可
    * `start/stop loading`
4. 实践
    * 重定向
    * 忽略请求，使用本地缓存
    * 自定义请求返回的结果
    * 全局网络设置
    * DNS 解析
5. 参考
    * [iOS开发之--- NSURLProtocol](https://www.jianshu.com/p/7c89b8c5482a)
    * [iOS 开发中使用 NSURLProtocol 拦截 HTTP 请求](https://github.com/Draveness/analyze/blob/master/contents/OHHTTPStubs/iOS%20开发中使用%20NSURLProtocol%20拦截%20HTTP%20请求.md)
    * [WKWebView 那些坑](https://mp.weixin.qq.com/s?__biz=MzA3NTYzODYzMg==&mid=2653578513&idx=1&sn=961bf5394eecde40a43060550b81b0bb&chksm=84b3b716b3c43e00ee39de8cf12ff3f8d475096ffaa05de9c00ff65df62cd73aa1cff606057d&mpshare=1&scene=1&srcid=0214nkrYxApaVTQcGw3U9Ryp)
    * [NSURLProtocol无法截获NSURLSession解决方案](https://zhongwuzw.github.io/2016/08/31/NSURLProtocol无法截获NSURLSession解决方案/)
    * [让 WKWebView 支持 NSURLProtocol](https://blog.yeatse.com/2016/10/26/support-nsurlprotocol-in-wkwebview/)

未完待续

