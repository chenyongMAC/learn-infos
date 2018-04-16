1.Weex概述
2.Weex工作原理
3.Weex在iOS端如何运行
4.Weex、RN、JSPatch


## 1.概述
write once， run everywhere

## 2.工作原理
服务器下发js bundle（可用vue，react开发），客户端内置jsframework解析js bundle，输出json格式的virtual dom，并使用flexbox盒模型解析和布局、渲染。

#### 1.WXMonitor监视器记录状态
WXMonitor负责记录下各个操作的tag值和记录成功和失败的原因，它提供了各种方便的宏。

#### 2.weexSDK初始化
1）加载本地jsframework（js资源文件）
2）注册Component、Module、Handler

①registerComponent:withClass:方法和registerComponent:withClass:withProperties:方法的区别
在于最后一个入参是否传@{@"append":@"tree"}，如果被标记成了@"tree"，那么在syncQueue堆积了很多任务的时候，会被强制执行一次layout。

②注册组件全部都是通过WXComponentFactory完成注册的。这是一个单例，它又一个_componentConfigs字典，存储组件所有配置。注册的过程即生成_componentConfigs的过程。

```
_componentConfigs
key: 组件名
value: WXComponentConfig
```
    
WXComponentConfig继承自WXInvocationConfig，在WXInvocationConfig中存储了组件名name，类名clazz，类里面的同步方法字典syncMethods和异步方法字典asyncMethods

WX_EXPORT_METHOD宏用NSStringFromSelector()方法把Component提供给Factory，它的原理是将前缀和方法在文件中的行数拼接，例如：

`wx_export_method_52`

代表Component中第52行的方法被暴露给了WeexSDK。

③最后一步，在JSFrame中注册组件。
在WXSDKManager里面会强持有一个WXBridgeManager。这个WXBridgeManager就是用来和JS交互的Bridge。
WXBridgeManager中会弱引用WXSDKInstance实例，是为了能调用WXSDKInstance的一些属性和方法。WXBridgeManager里面最重要的一个属性就是WXBridgeContext。
即，持有关系是：
WXSDKManager --->  (strong) WXBridgeManager  ---> (weak) WXSDKInstance

最终是WXBridgeManager里面的WXBridgeContext调用registerComponents，进行组件的注册。该过程在一个子线程中。

PS: 
这里有一个需要注意的一点，由于是在子线程上注册组件，那么JSFramework如果没有加载完成，native去调用js的方法，必定调用失败。所以需要在JSFramework加载完成之前，把native调用JS的方法都缓存起来，一旦JSFramework加载完成，把缓存里面的方法都丢给JSFramework去加载。(事实也是这样的，这时候jsframework并没有完成初始化)
所以在WXBridgeContext中需要一个NSMutableArray，用来缓存在JSFramework加载完成之前，调用JS的方法。这里是保存在_methodQueue里面。如果JSFramework加载完成，那么就会调用callJSMethod:args:方法。


#### 2.


JSFramework在本地注册4个重要的回调函数：

```
1.typedef NSInteger(^WXJSCallNative)(NSString *instance, NSArray *tasks, NSString *callback);
2.typedef NSInteger(^WXJSCallAddElement)(NSString *instanceId,  NSString *parentRef, NSDictionary *elementData, NSInteger index);
3.typedef NSInvocation *(^WXJSCallNativeModule)(NSString *instanceId, NSString *moduleName, NSString *methodName, NSArray *args, NSDictionary *options);
4.typedef void (^WXJSCallNativeComponent)(NSString *instanceId, NSString *componentRef, NSString *methodName, NSArray *args, NSDictionary *options);

//这4个闭包函数被传入4个对应的函数中
registerCallNative			//JS用来调用任意一个Native方法的
registerCallAddElement		//JS用来给当前页面添加视图元素的
registerCallNativeModule	//JS用来调用模块里面暴露出来的方法
registerCallNativeComponent//JS用来调用组件里面暴露出来的方法
```