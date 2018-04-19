1.configuration配置，生成webview
2.监听title，estimatedProgress，提供标题，进度条
3.navigationItem: back, close, actions
4.UIDelegate：弹框
5.NavigationDelegate
6.userContentController注入js，接收scriptMessage
7.didReceiveScriptMessage
8.NSMethodSignature，用方法名生成方法调用（方法名由web端提供）
9.提供全局字典，存放注册的组件。key = function name,  value = widgets‘ object






wkwebview’s problem
来源：https://mp.weixin.qq.com/s/rhYKLIbXOsUJC_n6dt9UfA
1.白屏：
在 UIWebView 上当内存占用太大的时候，App Process 会 crash；而在 WKWebView 上当总体的内存占用比较大的时候，WebContent Process 会 crash，从而出现白屏现象。

解决：
1）webView.title为空时，reload
2）webViewWebContentProcessDidTerminate，reload


2.cookie
WKWebView Cookie 问题在于 WKWebView 发起的请求不会自动带上存储于 NSHTTPCookieStorage 容器中的 Cookie。

解决：
1）iOS11提供了cookie的API
2）WKWebView loadRequest 前，在 request header 中设置 Cookie, 解决首个请求 Cookie 带不上的问题

```
WKWebView * webView = [WKWebView new]; 
NSMutableURLRequest * request = [NSMutableURLRequest requestWithURL:[NSURL URLWithString:@"http://h5.qzone.qq.com/mqzone/index"]]; 

[request addValue:@"skey=skeyValue" forHTTPHeaderField:@"Cookie"]; 
[webView loadRequest:request];
```

对于一些页面的后续页面（同域）ajax，iframe请求，上面的方案还存在cookie丢失问题。所以可以通过注入js的方式设置cookie：

```
WKUserContentController* userContentController = [WKUserContentController new]; 
WKUserScript * cookieScript = [[WKUserScript alloc] initWithSource: @"document.cookie = 'skey=skeyValue';" injectionTime:WKUserScriptInjectionTimeAtDocumentStart forMainFrameOnly:NO]; 

[userContentController addUserScript:cookieScript];
```

如果这种情况下还有丢失cookie，如302请求。可以在 decidePolicyForNavigationAction 方法中拦截请求，并手动写入cookie。不过这种方法依然解决不了页面 iframe 跨域请求的 Cookie 问题，毕竟-[WKWebView loadRequest:]只适合加载 mainFrame 请求。



3.NSURLProtocol问题
WKWebView 在独立于 app 进程之外的进程中执行网络请求，请求数据不经过主进程，因此，在 WKWebView 上直接使用 NSURLProtocol 无法拦截请求。

通过注册 http(s) scheme 后 WKWebView 将可以使用 NSURLProtocol 拦截 http(s) 请求。但是这种方案有2个严重的问题：

1）post请求的body数据被清空
由于 WKWebView 在独立进程里执行网络请求。一旦注册 http(s) scheme 后，网络请求将从 Network Process 发送到 App Process，这样 NSURLProtocol 才能拦截网络请求。在 webkit2 的设计里使用 MessageQueue 进行进程之间的通信，Network Process 会将请求 encode 成一个 Message,然后通过 IPC 发送给 App Process。出于性能的原因，encode 的时候 HTTPBody 和 HTTPBodyStream 这两个字段被丢弃掉了

2）对ATS支持不足
测试发现一旦打开ATS开关：Allow Arbitrary Loads 选项设置为NO，同时通过 registerSchemeForCustomProtocol 注册了 http(s) scheme，WKWebView 发起的所有 http 网络请求将被阻塞（即便将Allow Arbitrary Loads in Web Content 选项设置为YES）


4.loadRequest问题
在 WKWebView 上通过 loadRequest 发起的 post 请求 body 数据会丢失：

```
//同样是由于进程间通信性能问题，HTTPBody字段被丢弃
[request setHTTPMethod:@"POST"];
[request setHTTPBody:[@"bodyData" dataUsingEncoding:NSUTF8StringEncoding]];
[wkwebview loadRequest: request];
```

这个问题是有解决方案的：
1）替换请求 scheme，生成新的 post 请求 request2: post://h5.qzone.qq.com/mqzone/index, 同时将 request1 的 body 字段复制到 request2 的 header 中（WebKit 不会丢弃 header 字段）;

2）通过-[WKWebView loadRequest:]加载新的 post 请求 request2;

3）通过 +[WKBrowsingContextController registerSchemeForCustomProtocol:]注册 scheme: post://;

4）注册 NSURLProtocol 拦截请求post://h5.qzone.qq.com/mqzone/index ,替换请求 scheme, 生成新的请求 request3: http://h5.qzone.qq.com/mqzone/index，将 request2 header的body 字段复制到 request3 的 body 中，并使用 NSURLConnection 加载 request3，最后通过 NSURLProtocolClient 将加载结果返回 WKWebView;


