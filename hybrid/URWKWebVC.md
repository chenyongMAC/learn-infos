
## Web Container方案
1.纯浏览器方案：
Native 除了扔给内嵌浏览器一个 URL 地址之外，就不做任何事情了，剩余的事情都由 Web 完成。这和用 Safari 或 Chrome 等普通浏览器打开一个网页并没有太多区别。只是我们固定了访问的地址。

2.前端模板渲染方案：
在客户端存储了一个 HTML 作为 UI 模板。Native 代码负责获取数据，向 HTML 文件模板中填入动态数据，得到一个可以在内嵌浏览器渲染的 HTML 文件。

3.Hybrid方案：
提供web容器和webView，并对webView提供扩展

## Hybrid方案的功能

1.提供route 一个url对应一个web page，本地路由表用于存放对应关系
2.封装发出的API请求
3.缓存web前端所需的静态文件和资源文件，如HTML、CSS、JavaScript、Image等
4.提供原生支持的功能

## 页面执行过程
1.打开一个URL
2.根据URL查询本地缓存的路由表。如果没记录，请求全量更新本地路由
3.拿到html后，查看本地是否有缓存。如果没有，请求远端html，并缓存
4.web容器正常使用流程，如下

#### web容器使用概览
1.configuration配置，注入本地js，生成webview
2.监听title，estimatedProgress，提供标题，进度条
3.navigationItem: back, close, actions
4.UIDelegate：弹框
5.NavigationDelegate
6.userContentController接收scriptMessage，message中包括以下内容：
target name, action name, callbackId, params

7.didReceiveScriptMessage
8.NSMethodSignature，用方法名生成方法调用（方法名由web端提供）
9.使用target-action匹配前端的方法调用
前端提供target name和action name，本地用methodSignatureForSelector方法生成NSMethodSignature对象。如果成功，则使用NSInvocation转发消息；如果失败，则结束此次调用。

10.action调用完后，如果有callbackId，将callbackId和params统一回传给前端。使用的是_handleMessageFromApp方法，该方法会判断message type字段，如果是回调内容，会去方法缓存表中找到回调函数并执行。（即，回调函数的处理，其实是在注入的js中的）
11.RU项目中没有使用通常概念上的session+cookie来存放信息。RU中登录时，服务器会下发一个openID；以后每次打开webView时，服务器会优先获取本地存储的openID。



## wkwebview’s problem
来源：https://mp.weixin.qq.com/s/rhYKLIbXOsUJC_n6dt9UfA

#### 1.白屏：
在 UIWebView 上当内存占用太大的时候，App Process 会 crash；而在 WKWebView 上当总体的内存占用比较大的时候，WebContent Process 会 crash，从而出现白屏现象。

解决：
1）webView.title为空时，reload
2）webViewWebContentProcessDidTerminate，reload


#### 2.cookie
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



#### 3.NSURLProtocol问题
WKWebView 在独立于 app 进程之外的进程中执行网络请求，请求数据不经过主进程，因此，在 WKWebView 上直接使用 NSURLProtocol 无法拦截请求。

通过注册 http(s) scheme 后 WKWebView 将可以使用 NSURLProtocol 拦截 http(s) 请求。但是这种方案有2个严重的问题：

1）post请求的body数据被清空
WebKit 进程是独立于 app 进程之外的，两个进程之间使用消息队列的方式进行进程间通信。比如 app 想使用 WKWebView 加载一个请求的话，就要把请求的参数打包成一个 Message，然后通过 IPC 把 Message 交给 WebKit 去加载，反过来 WebKit 的请求想传到 app 进程的话（比如 URLProtocol ），也要打包成 Message 走 IPC。出于性能的原因，打包的时候 HTTPBody 和 HTTPBodyStream 这两个字段被丢弃掉了

2）对ATS支持不足
测试发现一旦打开ATS开关：Allow Arbitrary Loads 选项设置为NO，同时通过 registerSchemeForCustomProtocol 注册了 http(s) scheme，WKWebView 发起的所有 http 网络请求将被阻塞（即便将Allow Arbitrary Loads in Web Content 选项设置为YES）


#### 4.loadRequest问题
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


