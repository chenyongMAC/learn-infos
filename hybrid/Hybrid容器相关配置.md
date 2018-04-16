## UserAgent管理
WebView容器其实有一个很重要的需求，就是修改WebView UA，作为区别巨有容器能力的WebView识别方式，一般情况下拿到的UA会长这样:
`Mozilla/5.0 (Linux; Android 6.0.1; XT1650-05 Build/MCC24.246-37; wv) AppleWebKit/537.36 (KHTML, like Gecko) Version/4.0 Chrome/51.0.2704.81 Mobile Safari/537.36`

#### 全局UA
iOS 8及 8 以下只能进行全局 UA 修改，可以通过 NSUserDefaults 的方式修改，一次修改每个WebView都有效（无论是常规 WebView 还是被你改造过的 Hybrid WebView）

```
NSDictionary *dictionary = [NSDictionary dictionaryWithObjectsAndKeys:UAStringXXX, @"UserAgent", nil];
[[NSUserDefaults standardUserDefaults] registerDefaults:dictionary];
```

#### 独立UA
iOS 9 有了独立UA，可以针对每一个 WKWebView 对象实例，设置专属的UA

```
if (@available(iOS 9.0, *)) {
    self.webView.customUserAgent = self.fullUserAgent;
}
```


## Cookie管理
1.UA适合传递跟设备相关的，描述设备固定不变的信息
2.Cookie适合传递任何你想要的数据，但Cookie有失效与域名限制
3.URL Query 适合传递任何你想要的数据，不过最好这个数据没什么安全敏感，因为GET请求是明文的（POST请求也可以，类比一下不多说了）

#### 传统的NSHTTPCookieStorage
通过 NSHTTPCookieStorage 设置的 Cookie ，这样设置的Cookie 无论是 UIWebView 页面请求还是 NSURLSession 网络请求，都会带上 Cookie，所以十分方便

#### WKWebView的Cookie
1.坑
WKWebView 发起的请求并不会带上 NSHTTPCookieStorage 里面的 Cookie

2.WKWebView ServerSide Cookie 设置
简单的说就是把 WKWebView 发起的 NSURLRequest 拦截，MutableCopy 一个，然后手动在RequestHeader里从NSHTTPCookieStorage读取Cookie进行添加

```
-(void)syncRequestCookie:(NSMutableURLRequest *)request
{
    if (!request.URL) {
        return;
    }
    
    NSArray *availableCookie = [[NSHTTPCookieStorage sharedHTTPCookieStorage] cookiesForURL:request.URL];
    NSMutableArray *filterCookie = [[NSMutableArray alloc]initWithArray:availableCookie];
 
    if (filterCookie.count > 0) {
        NSDictionary *reqheader = [NSHTTPCookie requestHeaderFieldsWithCookies:filterCookie];
        NSString *cookieStr = [reqheader objectForKey:@"Cookie"];
        [request setValue:cookieStr forHTTPHeaderField:@"Cookie"];
    }
    return;
}
```

当服务器发生重定向的时候，此时第一次在 RequestHeader 中写入的 Cookie 会丢失，还需要重新对重定向的 NSURLRequest 进行 RequestHeader 的 Cookie 处理 ，简单的说就是在 webView:decidePolicyForNavigationAction:decisionHandler: 的时候，判断此时 Request 是否有你要的 Cookie 没有就Cancel掉，修改Request 重新发起


3.WKWebView ClientSide Cookie 设置
上面这么写完了，当页面加载的时候，后端无论是啥语言，都能从请求里看到 Cookie 了，但是后端渲染返回页面后，在 Client Side 浏览器里运行的时候，JS 在执行的时候用 document.cookie API 是读取不到的。所以还得针对 Client Side Cookie 进行处理

```
-(void)syncClientCookieScripts:(NSMutableURLRequest *)request{
    if (!request.URL) {
        return;
    }
    
    NSArray *availableCookie = [[NSHTTPCookieStorage sharedHTTPCookieStorage] cookiesForURL:request.URL];
    NSMutableArray *filterCookie = [[NSMutableArray alloc] init];
 
    for (NSHTTPCookie * cookie in availableCookie) {
        if (self.syncCookieMode) {
            //httponly需求不得写入js cookie
            if (!cookie.HTTPOnly) {
                [filterCookie addObject:cookie];
            }
        }
    }
    
    // 拼接 JS 代码 对 Client Side 注入Cookie
    NSDictionary *reqheader = [NSHTTPCookie requestHeaderFieldsWithCookies:filterCookie];
    NSString *cookieStr = [reqheader objectForKey:@"Cookie"];
    if (filterCookie.count > 0) {
        for (NSHTTPCookie *cookie in filterCookie) {
            NSTimeInterval expiretime = [cookie.expiresDate timeIntervalSince1970];
            NSString *js = [NSString stringWithFormat:@"document.cookie ='%@=%@;expires=%f';",cookie.name,cookie.value,expiretime];
            WKUserScript *jsscript = [[WKUserScript alloc]initWithSource:js injectionTime:WKUserScriptInjectionTimeAtDocumentStart forMainFrameOnly:NO];
            [self.userContentController addUserScript:jsscript];
        }
    }
    return;
}
```


