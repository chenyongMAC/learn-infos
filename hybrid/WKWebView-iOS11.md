## 1.Cookie

iOS11推出了新的 API WKHTTPCookieStore 可以用来拦截 WKWebView 的 Cookie 信息。用法如下：

```
WKHTTPCookieStore *cookieStroe = self.webView.configuration.websiteDataStore.httpCookieStore;
//get cookies
[cookieStroe getAllCookies:^(NSArray<NSHTTPCookie *> * _Nonnull cookies) {
    NSLog(@"All cookies %@",cookies);
}];
//set cookie
NSMutableDictionary *dict = [NSMutableDictionary dictionary];
dict[NSHTTPCookieName] = @"userid";
dict[NSHTTPCookieValue] = @"123";
dict[NSHTTPCookieDomain] = @"xxxx.com";
dict[NSHTTPCookiePath] = @"/";
NSHTTPCookie *cookie = [NSHTTPCookie cookieWithProperties:dict];
[cookieStroe setCookie:cookie completionHandler:^{
    NSLog(@"set cookie");
}];
//delete cookie
[cookieStroe deleteCookie:cookie completionHandler:^{
    NSLog(@"delete cookie");
}];
```

#### 利用 iOS11 API WKHTTPCookieStore 解决 WKWebView 首次请求不携带 Cookie 的问题

问题说明：
由于许多 H5 业务都依赖于 Cookie 作登录态校验，而 WKWebView 上请求不会自动携带 Cookie。比如，如果你在Native层面做了登陆操作，获取了Cookie信息，也使用 NSHTTPCookieStorage 存到了本地，但是使用 WKWebView 打开对应网页时，网页依然处于未登陆状态。如果是登陆也在 WebView 里做的，就不会有这个问题。
iOS11 的 API 可以解决该问题，只要是存在 WKHTTPCookieStore 里的 cookie，WKWebView 每次请求都会携带，存在 NSHTTPCookieStorage 的cookie，并不会每次都携带。于是会发生首次 WKWebView 请求不携带 Cookie 的问题。

解决方法：
在执行 -[WKWebView loadReques:] 前将 NSHTTPCookieStorage 中的内容复制到 WKHTTPCookieStore 中，以此来达到 WKWebView Cookie 注入的目的。


#### 利用 iOS11 之前的 API 解决 WKWebView 首次请求不携带 Cookie 的问题

通过让所有 WKWebView 共享同一个 WKProcessPool 实例，可以实现多个 WKWebView 之间共享 Cookie（session Cookie and persistent Cookie）数据。不过 WKWebView WKProcessPool 实例在 app 杀进程重启后会被重置，导致 WKProcessPool 中的 Cookie、session Cookie 数据丢失，目前也无法实现 WKProcessPool 实例本地化保存。可以采取 cookie 放入 Header 的方法来做。

```
 WKWebView * webView = [WKWebView new]; 
 NSMutableURLRequest * request = [NSMutableURLRequest requestWithURL:[NSURL URLWithString:@"http://xxx.com/login"]]; 
 [request addValue:@"skey=skeyValue" forHTTPHeaderField:@"Cookie"]; 
 [webView loadRequest:request];
```

其中，“skey=skeyValue”的获取可以统一通过domain获取，参考：[文档](https://help.aliyun.com/knowledge_detail/60199.html?spm=a2c4g.11186623.6.570.6eG4AZ)

通过 document.cookie 设置 Cookie 解决后续页面(同域)Ajax、iframe 请求的 Cookie 问题：

```
WKUserContentController* userContentController = [WKUserContentController new]; 
 WKUserScript * cookieScript = [[WKUserScript alloc] initWithSource: @"document.cookie = 'skey=skeyValue';" injectionTime:WKUserScriptInjectionTimeAtDocumentStart forMainFrameOnly:NO]; 
 [userContentController addUserScript:cookieScript];
```


