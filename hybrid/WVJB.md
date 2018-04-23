WebViewJavaScriptBridge是一个iOS/OSX上，用于WKWebView和UIWebView的让Obj-C和JavaScript相互发送消息(交互)的桥接库。
它主要是用拦截假跳转scheme的方式实现通信的。以下分三部分讲解它的实现方式。

## 1.初始化webView，注册事件

OC端的使用没有特别多内容，如下：

```
- (void)viewWillAppear:(BOOL)animated {
    if (_bridge) { return; }
    
    UIWebView* webView = [[UIWebView alloc] initWithFrame:self.view.bounds];
    [self.view addSubview:webView];
    
    [WebViewJavascriptBridge enableLogging];
    
    _bridge = [WebViewJavascriptBridge bridgeForWebView:webView];
    [_bridge setWebViewDelegate:self];
    
    [_bridge registerHandler:@"testObjcCallback" handler:^(id data, WVJBResponseCallback responseCallback) {
        NSLog(@"testObjcCallback called: %@", data);
        responseCallback(@"Response from testObjcCallback");
    }];
    
    [_bridge callHandler:@"testJavascriptHandler" data:@{ @"foo":@"before ready" }];
    
    [self renderButtons:webView];
    [self loadExamplePage:webView];
}
```

Web端调用，需要先生成一个iframe，提供特定的src。客户端识别这个src后，会执行注入js文件操作。注入的js即桥接文件，用于适配2端的方法调用。实例代码如下：

```
function setupWebViewJavascriptBridge(callback) {
        if (window.WebViewJavascriptBridge) { return callback(WebViewJavascriptBridge); }
        if (window.WVJBCallbacks) { return window.WVJBCallbacks.push(callback); }
        window.WVJBCallbacks = [callback];
        var WVJBIframe = document.createElement('iframe');
        WVJBIframe.style.display = 'none';
        WVJBIframe.src = 'https://__bridge_loaded__';
        document.documentElement.appendChild(WVJBIframe);
        setTimeout(function() { document.documentElement.removeChild(WVJBIframe) }, 0)
    }
```

## 2.原理
首先要明确，js和native是2个不同的执行环境，是不可能相互调用到对方的方法的。桥接库的本质是通过通信手段传递数据，告知对方调用某些API。在js上下文中，我们能拿到window、document对象，并修改它们的内容，这是通信方式可行的前提条件。

## 3.js -> native
当native注册了方法到js上下文。桥接对象中生成了mssageHandlers(可变字典)，key=handler name，value=register callback。

如果web端调用了一个已注册的事件，会触发以下流程：
1.web端调用注入js文件中的callHandler方法，校验参数，并调用doSend方法
2.doSend中，如果传参需要callback，则生成callbackId，加入到message中。并且，会把callbackId和callback以键值对的形式存入responseCallbacks
3.修改iframe，触发webView代理方法
4.native代理中识别scheme，调用flushMessageQueue
5.根据有无callbackId，判断是否要生成responseCallback
6.执行注册事件对应的方法
7.如果web端需要回调，即5中存在callbackId，调用id对应的callback，即触发web端回调。并且，传参中会附带responseId和responseData。web端会识别这个responseId，用来区分是回调还是客户端调用的callHandler方法。

## 4.native -> js

原理和3一模一样，无非是一个是native端实现，一个是js实现。
