## Weex 对 DOM 设计的简化和取舍
我们对 virtual-DOM 的设计很大程度上借鉴了 HTML DOM 的设计，不论从 API 还是 class，但做了一定的简化和取舍，主要包括以下几点：

1.传统的 HTML DOM 分了很多种 nodeType，比如 Element、TextNode、Comment、CDATA、Entity、Attribute、Fragment … 等， Weex 只保留了 Element 和 Comment ，一个 Element 对应着 native 的一个 View，而 Comment 通常对 native 来说是无意义的，但是它可以帮助 JS 上层的框架用作一些特殊处理时的 placeholder。

2.传统的 HTML DOM 是既有 attribute 又有 property 的，property 里还包括 style、方法调用这样的特殊 property， Weex 只保留了 attributes，没有 properties ，但支持一个特殊的维度，就是样式 style。

3.传统的 HTML DOM 是支持同一个 Element 绑定多个事件的，从 JS 和 native 通信的角度，这样做是没有必要的， 所以 Weex 只提供了 DOM Level 0 的事件模型，也就是 onxxx="fn"。 如果同一个 Element 需要在业务层绑定多个事件，可以在 virtual-DOM 上层再进行封装
传统的 HTML DOM 事件是存在捕获和冒泡阶段的， Weex 做了精简，没有支持冒泡或捕获事件 ，只有在 native 层的当前元素触发该事件才会 fireEvent 给 JS。

4.传统的 HTML DOM 针对每个页面有唯一且现成的 document、document.documentElement、document.body，但是在 Weex 中，由于每个页面需要的初始化 body 类型是有选择的，基本上分 scroller、div、list 这三种，根据页面不同的展示特征而定， 所以 Weex 页面的 document.body 是需要手动创建的，并且有机会制定其类型为 scroller、div、list 其中的一种。

5.Weex 不支持 XML 的 namespace 语法


所以综上所述，这是一个非常精简版的 virtual-DOM 设计。我们在 Weex 中所能感受到的各种视觉效果和交互效果，实际上都是通过这样的 virtual-DOM 结构进行分解和执行的。