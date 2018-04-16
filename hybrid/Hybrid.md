http://awhisper.github.io/2018/01/02/hybrid-jscomunication/
http://awhisper.github.io/2018/03/06/hybrid-webcontainer/


## 1.JS Native通信方案
#### JS调用Native
1.假跳转的请求拦截
2.弹窗拦截
	alert()
	prompt()
	confirm()
3.JS上下文注入
	JavaScriptCore注入
	scriptMessageHandler注入
	
#### Native调用JS
1.evaluatingJavaScript 直接注入执行JS代码
2.loadUrl 浏览器用’javascript:’+JS代码做跳转地址
3.WKWebView：WKUserScript WKWebView的addUserScript方法，在加载时机注入
	
	

## JS调用Native方案分析
#### 1.假跳转的请求拦截 
客户端会无差别拦截所有请求，真正的url地址应该照常放过，只有协议域名匹配的url地址才应该被客户端拦截，拦截下来的url不会导致webview继续跳转错误地址，因此无感知，相反拦截下来的url我们可以读取其中路径当做指令，读取其中参数当做数据，从而根据约定调用对应的native原生代码。

iOS的UIWebView的拦截方式webView:shouldStartLoadWithRequest:navigationType:
iOS的WKWebView的拦截方式 webView:decidePolicyForNavigationAction:decisionHandler:

```
//常规的Http地址
https://wenku.baidu.com/xxxx?xx=xx
//假的请求通信地址
wakaka://wahahalalala/action?param=paramobj

协议scheme：也就是http/https/file等，上面用了wakaka
域名host：上面的 wenku.baidu.com 和 wahahalalala
路径path：上面的 xxxx?或action？
参数query：上面的 xx=xx或param=paramobj
```

#### 2.弹窗拦截
alert() 弹出个提示框，只能点确认无回调
confirm() 弹出个确认框（确认，取消），可以回调
prompt() 弹出个输入框，让用户输入东西，可以回调

iOS的WKWebView，WKUIDelegate提供了3种弹窗的代理方法
iOS的UIWebView公有API捕获不到弹窗，私有API中，可以利用Undocumented API的方式来拦截弹框。原理是可以自行创建一个categroy，在里面实现一个未出现在任何UIWebView头文件里的delegate，就能拦截弹框了（这个Undocumented的delegate长得和WKWebView的拦截delegate一个样子）


#### 3.JS上下文注入
原理：直接将一个native对象（or函数）注入到JS里面，可以由web的js代码直接调用，直接操作。

###### 3.1 UIWebview JavaScriptCore注入
可以拿到当前WebView的JS上下文JSContext，然后就要准备往这个JSContext里面注入准备好的block，而这个准备好的block，负责解读JS传过来的数据，从而分发调用各种native函数指令。只要被注入的NSObject对象遵循JSExportOC的协议，JS都能访问到，这相当于JS可以直接调用访问OC的内存对象。

```
//拿到当前WebView的JS上下文
JSContext *context = [webview valueForKeyPath:@"documentView.webView.mainFrame.javaScriptContext"];
//给这个上下文注入callNativeFunction函数当做JS对象
context[@"callNativeFunction"] = ^( JSValue * data )
{
    //1 解读JS传过来的JSValue  data数据
    //2 取出指令参数，确认要发起的native调用的指令是什么
    //3 取出数据参数，拿到JS传过来的数据
    //4 根据指令调用对应的native方法，传递数据
    //5 此时还可以将客户端的数据同步返回！
}
```

###### 3.2 WKWebView scriptMessageHandler注入
直接调用WKWebView的API即可

```
//添加注入事件
[self.webView.configuration.userContentController addScriptMessageHandler:self name:@"nativeObject"];
//移除对象注入
[self.webView.configuration.userContentController removeScriptMessageHandlerForName:@"nativeObject"];

-(void)userContentController:(WKUserContentController *)userContentController didReceiveScriptMessage:(WKScriptMessage *)message { 
	//接收JS事件
}
```


## Native调用JS方案分析

#### 1.evaluatingJavaScript 执行JS代码

#### 2.loadUrl 执行JS代码（安卓支持）
安卓在4.4以前是不能用evaluatingJavaScript这个方法的，因此之前安卓都用的是webview直接loadUrl，但是传入的url并不是一个链接，而是以”javascript:”开头的js代码，从而达到让webview执行js代码的作用

#### 3.WKUserScript 执行JS代码
对于iOS的WKWebView，除了evaluatingJavaScript，还有WKUserScript这个方式可以执行JS代码，他们之间是有区别的：

- evaluatingJavaScript 是在客户端执行这条代码的时候立刻去执行当条JS代码
- WKUserScript 是预先准备好JS代码，当WKWebView加载Dom的时候，执行当条JS代码


## 方案对比

#### 1.假请求的通信拦截
优点：
1.版本兼容性好：iOS6及以前只有这唯一的一种方式，这也是为什么WebViewJavascriptBridge如此热门的原因

缺点：
1.丢失消息
因为假跳转的请求归根结底是一种模拟跳转，跳转这件事情上webview会有限制，当JS连续发送多条跳转的时候，webview会直接过滤掉后发的跳转请求，因此第二个消息根本收不到，想要收到怎么办？JS里将第二条消息延时一下，比如延时500ms发送第二个消息。
2.URL长度限制


#### 2.弹窗拦截
- UIWebView不支持，但没事UIWebView有更好的JS上下文注入的方式，JSContext不仅支持直接传递对象无需json序列化，还支持传递function函数给客户端呢
安卓一切正常，不会出现丢消息的情况
- WKWebView一切正常，也不会出现丢消息的情况，但其实WKWebView苹果给了更好的API，何不用那个，至少用这个是可以直接传递对象无需进行json序列化的

#### 3.JS上下文注入
优点：
1.支持JS同步返回
要知道我们看到的所有JS通信框架设计的都是异步返回，包括RN（这有设计原因，但不代表JSC不支持同步返回），都是设计了一套callback机制，一条通信消息到达Native后，如果需要返回数据，需要调用这个callback接口由Native反向通知JS

```
//同步JS调用Native  JS这边可以直接写=  !!!
var nativeNetStatus = nativeObject.getNetStatus();
//异步JS调用Native JS只能这么写
nativeObject.getNetSatus(callback(net){
    console.log(net)
})
```

2.支持直接传递对象，无需通过字符串序列化
一个JS对象在JS代码中如果想通过假跳转/弹窗拦截等方式，那么必须把JS对象搞成json，然后才能传递给端，端拿到后还要反解成字典对象，然后才能识别，但是JS上下文注入不需要（其实他本质上是框架层帮你做了这件事情，就是JSValue这个iOS类的能力）

缺点：
1.只有UIWebView可以用KVC取到JSContext，取到了JSContext才能发挥JavaScriptCore的牛逼能力。UIWebView的JSContext是通过iOS的kvc方法拿到，而非UIWebView的直接接口API，因此UIWebView-JSContext注入使用上要非常注意注入时机。因为WebView每次加载一个新地址都会启用一个新的JSContext，在loadUrl之前注入，会因为旧的JSContext已被舍弃导致注入无效，若在WebView触发FinishLoad事件的时候注入，又会导致在FinishLoad之前执行的JS代码，是无法调用native通信的

- UIWebView-JSContext 在loadUrl之前注入无效
- UIWebView-JSContext 在FinishLoad之后注入有效但有延迟


#### 4.WKWebView的scriptMessageHandler注入
优点：
1.无需json化传递数据
webkit.messageHandlers.xxx.postMessage()是支持直接传递json数据，无需前端客户端字符串处理的
2.不会丢消息

缺点：
1.版本要求iOS8
2.不支持JSContext那样的同步返回




