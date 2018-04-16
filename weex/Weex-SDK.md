


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