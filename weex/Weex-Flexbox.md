## Weex布局算法

#### 1.布局算法
1）weex中的布局算法来源
打开Weex的源码的Layout文件夹，就会看到两个c的文件，这两个文件就是今天要谈的Weex的布局引擎。
Layout.h和Layout.c最开始是来自于React-Native里面的代码。也就是说Weex和React-Native的布局引擎都是同一套代码。当前React-Native的代码里面已经没有这两个文件了，而是换成了Yoga。

Yoga本是Facebook在React Native里引入的一种跨平台的基于CSS的布局引擎，它实现了Flexbox规范，完全遵守W3C的规范。随着该系统不断完善，Facebook对其进行重新发布，于是就成了现在的Yoga

Weex的这种FlexBox的实现其实只是W3C标准的一个实现的子集，因为FlexBox的官方标准里面还有一些并没有实现出来。Weex的布局都在子线程，刷新渲染回到主线程。

#### 2.flexbox中的布局算法
1）盒模型
Weex 盒模型基于 CSS 盒模型，每个 Weex 元素都可视作一个盒子。我们一般在讨论设计或布局时，会提到「盒模型」这个概念。盒模型描述了一个元素所占用的空间。每一个盒子有四条边界：外边距边界 margin edge, 边框边界 border edge, 内边距边界 padding edge 与内容边界 content edge。这四层边界，形成一层层的盒子包裹起来，这就是盒模型大体上的含义。

2）Yoga布局算法流程 （选修）


2.布局算法性能分析 
flexbox优于autolayout

3.如何布局原生界面
1）createRoot:
当JSFramework把JS文件转换类似JSON的文件之后，就开始调用Native的callNative方法。
callNative方法会最终执行WXBridgeContext类里面的[weakSelf invokeNative:instance tasks:tasks callback:callback]方法。这里会把JS从发送过来的callNative方法转换成Native的组件component的方法调用或者模块module的方法调用。

那么createRoot到底做了什么？
a) 创建WXComponent
b) 初始化css_node_t
c) 添加UI任务到uiTaskQueue数组中




