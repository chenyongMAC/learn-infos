Digest
1.4种事件类型
2.事件冒泡的使用和停止
3.添加事件
4.事件实现原理
5.事件传递
6.编码EventModule


## 1.事件类型
#### 1.通用事件
1.click
当组件上发生点击手势时被触发。

2.longpress
如果一个组件被绑定了 longpress 事件，那么当用户长按这个组件时，该事件将会被触发。

#### 2.Appear
如果一个位于某个可滚动区域内的组件被绑定了 appear 事件，那么当这个组件的状态变为在屏幕上可见时，该事件将被触发。 

#### 3.Disappear
如果一个位于某个可滚动区域内的组件被绑定了 disappear 事件，那么当这个组件被滑出屏幕变为不可见状态时，该事件将被触发。

#### 4.Page 事件
Weex 通过 viewappear 和 viewdisappear 事件提供了简单的页面状态管理能力。
viewappear 事件会在页面就要显示或配置的任何页面动画被执行前触发,viewdisappear 事件会在页面就要关闭时被触发。
与组件的 appear 和 disappear 事件不同的是，viewappear 和 viewdisappear 事件关注的是整个页面的状态，所以它们必须绑定到页面的根元素上。



## 2.事件冒泡
1.概念
以点击事件为例，比如一个点击事件发生在某个 <video> 元素上，这个元素有一个父元素（比如是个 div 元素），浏览器会执行两个处理阶段 - 捕获（capturing）阶段和冒泡（bubbling）阶段。在 web 开发中冒泡阶段会用的比较多，而捕获处理用的比较少。

在捕获阶段，浏览器检查当前元素的最外层父节点（在 web 上，比如，<html> 元素），如果它上面绑定了一个 click 事件处理器，那么先执行这个处理器。然后检查下一个元素，<html> 的子元素里是 <video> 祖先元素的那个元素，做同样的检测。一步步直到遇到当前点击的这个元素本身。

接下来是冒泡阶段，方向和捕获阶段相反：浏览器先检测当前被点击的元素上是否注册了点击事件处理器，如果有则执行它。接下来检测它的父元素，一步步向上，直到最外层 <html> 元素。

2.Weex冒泡事件
我们一般使用默认的事件注册机制，将事件处理注册在冒泡阶段，所以处理冒泡阶段的情况比较多。当我们想要停止事件冒泡，只需要调用事件对象的 stopPropagation 方法。标准事件对象包含 stopPropagation 方法，当执行事件处理器时调用该方法，会立即停止事件冒泡，这样事件冒泡处理链上的后续处理器就不会再执行下去。

Weex 在 0.13 版本 SDK 里实现了事件冒泡机制。注意事件冒泡默认是不开启的，你需要在模板根元素上加上属性 bubble=true 才能开启冒泡。

```
<template>
  <!-- 开启事件冒泡机制. -->
  <div bubble="true">
    ...
  </div>
</template>
```

```
{
  handleClick (event) {
    // 阻止继续冒泡.
    event.stopPropagation()
  }
}
```


## 3.源码分析

#### 1.添加事件
WXComponentManager 中 _addEventOnMainThread 添加事件。
WX_ADD_EVENT 是一个宏：

```
#define WX_ADD_EVENT(eventName, addSelector) \
if ([addEventName isEqualToString:@#eventName]) {\
    [self addSelector];\
}
```

即是判断待添加的事件 addEventName 的名字和默认支持的事件名字 eventName 是否一致，如果一致，就执行 addSelector 方法。最后会执行一个 addEvent: 方法，每个组件里面会可以重写这个方法。在这个方法里面做的就是对组件的状态的标识。

在 WXGlobalEventModule 模块中，有一个 fireGlobalEvent: 方法。开发者可以通过 WXGlobalEventModule 进行全局的通知。


#### 2. 通用事件
WXComponent中定义了5个手势识别器。Weex的通用事件里面就包含这5种，点击事件，轻扫事件，长按事件，拖动事件，通用触摸事件。由上文得知，5个事件对应了5个“addSelector”。

例：
1）点击事件

```
#define WX_ADD_EVENT(click, addClickEvent) \
if ([addEventName isEqualToString:@“click”]) {\
    [self addClickEvent];\
}

- (void)addClickEvent
{
    if (!_tapGesture) {
        _tapGesture = [[UITapGestureRecognizer alloc] initWithTarget:self action:@selector(onClick:)];
        _tapGesture.delegate = self;
        [self.view addGestureRecognizer:_tapGesture];
    }
}

//Weex 会计算点击出点击到的视图的坐标以及宽高尺寸。
- (void)onClick:(__unused UITapGestureRecognizer *)recognizer
{
    NSMutableDictionary *position = [[NSMutableDictionary alloc] initWithCapacity:4];
    CGFloat scaleFactor = self.weexInstance.pixelScaleFactor;
    
    if (!CGRectEqualToRect(self.calculatedFrame, CGRectZero)) {
        CGRect frame = [self.view.superview convertRect:self.calculatedFrame toView:self.view.window];
        position[@"x"] = @(frame.origin.x/scaleFactor);
        position[@"y"] = @(frame.origin.y/scaleFactor);
        position[@"width"] = @(frame.size.width/scaleFactor);
        position[@"height"] = @(frame.size.height/scaleFactor);
    }

    [self fireEvent:@"click" params:@{@"position":position}];
}
```

解析：
在日常 iOS 开发中，开发者使用的计算单位是 pt。由于每个屏幕的 ppi 不同(ppi:Pixels Per Inch，即每英寸所拥有的像素数目，屏幕像素密度。)，最终会导致分辨率的不同。而Weex 的开发中，目前都是用的 px，而且 Weex 对于长度值目前只支持像素 px 值，还不支持相对单位（em、rem）。那么就需要 pt 和 px 的换算了。


2）轻扫事件，长按事件，拖动事件和前者大体相同，就不具体展开了。

3）触摸事件
Weex 里面对每个 Component 都新建了一个手势识别器。当用户开始触摸屏幕，在屏幕上移动，手指从屏幕上结束触摸，取消触摸，分别都会触发touchesBegan:，touchesMoved:，touchesEnded:，touchesCancelled:方法。


#### 3.Appear事件
由于 Appear 事件和 Disappear 事件都必须要求是滚动视图，所以这里会遍历当前视图的 supercomponent，直到找到一个遵循 WXScrollerProtocol 的 supercomponent。

在滚动视图里面包含有一个 listenerArray，数组里面装的都是被监听的对象。添加进这个数组会先判断当前是否有相同的 WXScrollToTarget，避免重复添加，如果没有重复的就新建一个 WXScrollToTarget，再添加进 listenerArray中。WXScrollToTarget 是一个普通的对象，里面弱引用了当前需要监听的 WXComponent，以及一个 BOOL 变量记录当前是否 Appear 了。

```
@interface WXScrollToTarget : NSObject
@property (nonatomic, weak)   WXComponent *target;
@property (nonatomic, assign) BOOL hasAppear;
@end
```


#### 4.Page 事件
Weex 通过 viewappear 和 viewdisappear 事件提供了简单的页面状态管理能力。

比如在 WXBaseViewController 里面，有这样一个更新当前 Instance 状态的方法，这个方法里面就会触发 viewappear 和 viewdisappear 事件。


## 4.事件传递
在 Weex 中，iOS Native 把事件传递给 JS 目前只有2种方式，一是 Module 模块的 callback，二是通过 Component 组件自定义的通知事件。

1）callback
在 WXModuleProtocol 中定义了2种可以 callback 给 JS 的闭包。两个闭包都可以 callback 把 data 传递回给 JS，data 可以是字符串或者字典。

```
typedef void (^WXModuleCallback)(id result);
typedef void (^WXModuleKeepAliveCallback)(id result, BOOL keepAlive);
```

2）自定义通知事件方法 --- fireEvent:params:domChanges:
Weex 事件的4种类型，通用事件，Appear 事件，Disappear 事件，Page 事件，全部都是通过 fireEvent:params:domChanges: 这种方式，Native 触发事件之后，Native 把参数传递给 JS 的。

例如：假设一个场景，用户点击了一张图片，于是就会改变 label 上的一段文字。

a）通过以上方法，发送给js的参数字典大概如下：

```
args:(
    0,
        (
                {
            args =             (
                3,
                click,
                                {
                    position =                     {
                        height = "199.8792270531401";
                        width = "199.8792270531401";
                        x = "274.7584541062802";
                        y = "115.9420289855072";
                    };
                    timestamp = "1489932655404.133";
                },
                                {
                }
            );
            method = fireEvent;
            module = "";
        }
    )
)
```

b）JSFramework 收到了 fireEvent 方法调用以后，处理完，知道 label 需要更新，于是又会开始 call Native，调用 Native 的方法。调用 Native 的 callNative 方法，发过来的参数如下：

```
(
        {
        args =         (
            4,
                        {
                value = "\U56fe\U7247\U88ab\U70b9\U51fb";
            }
        );
        method = updateAttrs;
        module = dom;
    }
)
```

c）最终会调用 Dom 的 updateAttrs 方法，会去更新 id 为4的 value，id 为4对应的就是 label，更新它的值就是刷新 label。接着 JSFramework 还会继续调用 Native 的 callNative 方法，发过来的参数如下：

```
(
        {
        args =         (
        );
        method = updateFinish;
        module = dom;
    }
)
```

调用 Dom 的 updateFinish 方法，即页面刷新完毕。