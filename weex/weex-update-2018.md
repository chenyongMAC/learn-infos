## Weex内核演进
#### 1.JS引擎
1）JavaScriptCore替换V8
2）JS Runtime运行在独立的进程
3）重新考虑Weex业务隔离的安全问题
4）瘦包

详解：
1）JavaScriptCore引擎是apple维护的，很少在Android平台商业化过。兼容性问题挑战大

2）使用JSCore之后内存的消耗增大了，Weex的内存问题导致的Crash也明显上升，所以考虑JS使用独立线程。Weex 原有的三个线程架构，JS 线程任务被并行成两个，JS独立的进程和原来JS线程，整体增强了JS任务执行的并发性，抵消了方案本身的IPC通信的性能损耗。
独立进程后，我们加入了自修复的能力，增强了JS Runtime进程的自动重置功能，解决JS Runtime 异常后的无法自动恢复的问题，极大地保障业务的稳定性

3）Weex 原有的设计，所有Weex页面包括JSFrameWork 都是运行在同一个JS Context里，并在JS层面，对全局的JS 变量做了freeze操作，防止被其他业务页面污染。这个设计主要还是出于高性能和安全性的综合考量。
经过新的讨论后，Weex团队决定针对每个Weex页面都创建一个JS Context 作为独立的运行环境，页面依赖的全局对象由Global Contex统一生层并传递给每个页面的Contex。传输过程中有2个重要的操作：
①对传输的对象做freeze操作
②对JavaScript引擎层面，使用native的setPrototype接口切断原型链的关系。这样就彻底解决了页面互相污染的问题


#### 2.Layout引擎
1）Layout引擎作为核心模块，Weex团队希望主导它的演进路线
2）Facebook的证书风险不能回避

详解：
全新的Layout引擎参考了 Google的FlexLayout的算法流程，重新实现了Weex的 Flex布局，目前性能和功能方面基本和yoga保持一致，后续会做一些性能优化。

未来的演进方向：
①自主开发全新的高性能的跨平台Layout引擎，统一由C++实现，IOS／android 两端复用同一套代码；
②扩展更多的布局方式，比如Gird布局、Absolute布局等
③编译器或服务端做预布局，提升端测的布局效率等


#### 3.Weex Core
1）从DSL的跨平台到内核的跨平台

详解：
抽象统一Weex 核心解析渲染流程，通过C + + 语言实现，实现 ios、android 平台核心逻辑统一，不仅可以增强平台间的一致性，降低维护成本，以及扩展平台的成本。通过C+ +代码执行效率要高于JAVA，同时还可提高Android端的代码执行性能。

提供标准的Weex Dom API Layer，简化DSL的接入成本，将大部分JSFramework native化

