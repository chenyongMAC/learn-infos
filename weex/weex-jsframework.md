https://yq.aliyun.com/articles/543724?spm=a2c4e.11155435.0.0.40983e17S4Mx7N#19

## 在 Weex 框架中的位置
JS Framework 介于前端框架和原生渲染引擎之间，处于承上启下的位置，也是跨框架跨平台的关键。
像 Vue 和 Rax 这类前端框架虽然内部的渲染机制、Virtual DOM 的结构都是不同的，但是都是用来描述页面结构以及开发范式的，对 Weex 而言只属于语法层，或者称之为 DSL (Domain Specific Language)。无论前端框架里数据管理和组件管理的策略是什么样的，它们最终都将调用 JS Framework 提供的接口来调用原生功能并且渲染真实 UI。底层渲染引擎中也不必关心上层框架中组件化的语法和更新策略是怎样的，只需要处理 JS Framework 中统一定义的节点结构和渲染指令。多了这么一层抽象，有利于标准的统一，也使得跨框架和跨平台成为了可能。


## js framework主要功能
1.适配前端框架
2.构建渲染指令树
4.JS-Native 通信
5.JS Service
6.准备环境接口

#### 1.适配前端框架
前端框架在 Weex 和浏览器中的执行过程不一样，这个应该不难理解。如何让一个前端框架运行在 Weex 平台上，是 JS Framework 的一个关键功能。

以 Vue.js 为例，在浏览器上运行一个页面大概分这么几个步骤：首先要准备好页面容器，可以是浏览器或者是 WebView，容器里提供了标准的 Web API。然后给页面容器传入一个地址，通过这个地址最终获取到一个 HTML 文件，然后解析这个 HTML 文件，得到DOM Tree。然后解析CSS得到CSS OM(object model)。将前两者合并得到render Tree。layout根据render tree计算每个节点的信息。然后根据计算好的信息绘制组件，并挂在到相应的挂载点上。

在 Weex 里的执行过程也比较类似，不过 Weex 页面对应的是一个 js 文件，不是 HTML 文件，而且不需要自行引入 Vue.js 框架的代码，也不需要设置挂载点。过程大概是这样的：首先初始化好 Weex 容器，这个过程中会初始化 JS Framework，Vue.js 的代码也包含在了其中。然后给 Weex 容器传入页面地址，通过这个地址最终获取到一个 js 文件，客户端会调用 createInstance 来创建页面，也提供了刷新页面和销毁页面的接口。大致的渲染行为和浏览器一致，但是和浏览器的调用方式不一样，前端框架中至少要适配客户端打开页面、销毁页面（push、pop）的行为才可以在 Weex 中运行。

即：
1）浏览器运行一个页面
浏览器或webView(提供了标准的web api) ---> 传入一个地址 ---> 获取到HTML文件，并解析，执行其中的脚本文件 ---> (开始渲染)先加载执行框架代码，如vue、react ---> 创建挂在的DOM元素 ---> 执行页面代码 ---> 渲染组件，并挂载到配置的挂载点上。

2）初始化weex容器(包括初始化jsframework，加载DSL代码) ---> 传入url ---> 客户端调用createInstance创建页面，并提供API ---> 渲染(后文介绍)

#### 2.构建渲染指令树
不同的前端框架里 Virtual DOM 的结构、patch 的方式都是不同的，这也反应了它们开发理念和优化策略的不同，**但是最终，在浏览器上它们都使用一致的 DOM API 把 Virtual DOM 转换成真实的 HTMLElement。**在 Weex 里的逻辑也是类似的，只是在最后一步生成真实元素的过程中，不使用原生 DOM API，而是使用 JS Framework 里定义的一套 Weex DOM API 将操作转化成渲染指令发给客户端。

此外 DOM 接口的设计相当复杂，背负了大量的历史包袱，也不是所有特性都适合移动端。JS Framework 里将这些接口做了大量简化，借鉴了 W3C 的标准，只保留了其中最常用到的一部分。目前的状态是够用、精简高效、和 W3C 标准有很多差异，但是已经成为 Vue 和 Rax 渲染原生 UI 的事实标准，后续还会重新设计这些接口，使其变得更标准一些。

前端框架调用这些接口会在 JS Framework 中构建一颗树，这颗树中的节点不包含复杂的状态和绑定信息，能够序列化转换成 JSON 格式的渲染指令发送给客户端。可以称之为“Render Directive Tree”。这颗树的层次结构和原生 UI 的层次结构是一致的，当前端的节点有更新时，这棵树也会跟着更新，然后把更新结果以渲染指令的形式发送给客户端。这棵树并不计算布局，也没有什么副作用，操作也都是很高效的，基本都是 O(1) 级别，偶尔有些 O(n) 的操作会遍历同层兄弟节点或者上溯找到根节点，不会遍历整棵树。

整个过程中，JSFramework将整个页面的渲染分拆成一个个渲染指令，然后通过JS Bridge发送给各个平台的RenderEngine进行Native渲染。因此，尽管在开发时写的是 HTML / CSS / JS，但最后在各个移动端（在iOS上对应的是iOS的Native UI、在Android上对应的是Android的Native UI）渲染后产生的结果是纯Native页面。 由于JSFramework在本地SDK中，只用在初始化的时候初始化一次，之后每个页面都无须再初始化了。也进一步的提高了与Native的通信效率。

#### 3.JS-Native通信
在开发页面过程中，除了节点的渲染以外，还有原生模块的调用、事件绑定、回调等功能，这些功能都依赖于 js 和 native 之间的通信来实现。

首先，页面的 js 代码是运行在 js 线程上的，然而原生组件的绘制、事件的捕获都发生在 UI 线程。在这两个线程之间的通信用的是 callNative 和 callJS 这两个底层接口（现在已经扩展到了很多个），它们默认都是异步的，在 JS Framework 和原生渲染器内部都基于这两个方法做了各种封装。

callNative 是由客户端向 JS 执行环境中注入的接口，提供给 JS Framework 调用，界面的节点（上文提到的渲染指令树）、模块调用的方法和参数都是通过这个接口发送给客户端的。为了减少调用接口时的开销，其实现在已经开了更多更直接的通信接口，其中有些接口还支持同步调用（支持返回值），它们在原理上都和 callNative 是一样的。

callJS 是由 JS Framework 实现的，并且也注入到了执行环境中，提供给客户端调用。事件的派发、模块的回调函数都是通过这个接口通知到 JS Framework，然后再将其传递给上层前端框架。

#### 4.JS Service
Weex 是一个多页面的框架，每个页面的 js bundle 都在一个独立的环境里运行，不同的 Weex 页面对应到浏览器上就相当于不同的“标签页”，普通的 js 库没办法实现在多个页面之间实现状态共享，也很难实现跨页通信。

在 JS Framework 中实现了 JS Service 的功能，主要就是用来解决跨页面复用和状态共享的问题的，例如 BroadcastChannel 就是基于 JS Service 实现的，它可以在多个 Weex 页面之间通信。

#### 5.环境接口
由于 Weex 运行环境和浏览器环境有很大差异，在 JS Framework 里还对一些环境变量做了封装，主要是为了解决解决原生环境里的兼容问题，底层使用渲染引擎提供的接口。这一层里的东西可以说都是用来“填坑”的，也是与环境有关 Bug 的高发地带，如果你只看代码的话会觉得莫名奇妙，但是它很可能解决了某些版本某个环境中的某个神奇的问题，也有可能触发了一个更神奇的问题。随着对 JS 引擎本身的优化和定制越来越多，这一层代码可以越来越少，最终会全部移除掉。

1）console: 原生提供了 nativeLog 接口，将其封装成前端熟悉的 console.xxx 并可以控制日志的输出级别。
2）timer: 原生环境里 timer 接口不全，名称和参数不一致。目前来看有了原生 C/C++ 实现的 timer 后，这一层可以移除。
3）freeze: 冻结当前环境里全局变量的原型链（如 Array.prototype）。

## jsframework的执行过程

#### 1.框架初始化
JS Framework 以及 Vue 和 Rax 的代码都是内置在了 Weex SDK 里的，随着 Weex SDK 一起初始化。SDK 的初始化一般在 App 启动时就已经完成了，只会执行一次。初始化过程中与 JS Framework 有关的是如下这三个操作：

**1）初始化 JS 引擎**
准备好 JS 执行环境，向其中注册一些变量和接口，如 WXEnvironment、callNative。

**2）执行 JS Framework 的代码。**
a) 注册上层 DSL 框架
如 Vue 和 Rax。这个过程只是告诉 JS Framework 有哪些 DSL 可用，适配它们提供的接口，如 init、createInstance，但是不会执行前端框架里的逻辑。

b) 初始化环境变量
并且会将原生对象的原型链冻结，此时也会注册内置的 JS Service，如 BroadcastChannel。

c) 如果 DSL 框架里实现了 init 接口，会在此时调用。

d)向全局环境中注入可供客户端调用的接口，如 callJS、createInstance、registerComponents，调用这些接口会同时触发 DSL 中相应的接口。

**3）注册原生组件和原生模块。**

再回顾看这两个过程，可以发现原生的组件和模块是注册进来的，DSL 也是注册进来的，Weex 做的比较灵活，组件模块是可插拔的，DSL 框架也是可插拔的，有很强的扩展能力。

#### 2.js bundle 执行过程
通常 Weex 的一个页面对应了一个 js bundle 文件，页面的渲染过程也是加载并执行 js bundle 的过程。

首先是调用原生渲染引擎里提供的接口来加载执行 js bundle，在 Android 上是 renderByUrl，在 iOS 上是 renderWithURL。在得到了 js bundle 的代码之后，会继续执行 SDK 里的原生 createInstance 方法，给当前页面生成一个唯一 id，并且把代码和一些配置项传递给 JS Framework 提供的 createInstance 方法。在 JS Framework 接收到页面代码之后，会判断其中使用的 DSL 的类型（Vue 或者 Rax），然后找到相应的框架，执行 createInstanceContext 创建页面所需要的环境变量。
在旧的方案中，JS Framework 会调用 runInContex 函数在特定的环境中执行 js 代码，内部基于 new Function 实现。在新的 Sandbox 方案中，js bundle 的代码不再发给 JS Framework，也不再使用 new Function，而是由客户端直接执行 js 代码。


#### 3.weex渲染
Weex 里页面的渲染过程和浏览器的渲染过程类似，整体可以分为5个步骤：
【创建前端组件】-> 【构建 Virtual DOM】->【生成“真实” DOM】->【发送渲染指令】->【绘制原生 UI】

前两个步骤发生在前端框架中，第三和第四个步骤在 JS Framework 中处理，最后一步是由原生渲染引擎实现的。

**具体分析渲染行为**
1.创建前端组件
以 Vue.js 为例，页面都是以组件化的形式开发的，整个页面可以划分成多个层层嵌套和平铺的组件。Vue 框架在执行渲染前，会先根据开发时编写的模板创建相应的组件实例，可以称为 Vue Component，它包含了组件的内部数据、生命周期以及 render 函数等。

如果给同一个模板传入多条数据，就会生成多个组件实例，这可以算是组件的复用。如上图所示，假如有一个组件模板和两条数据，渲染时会创建两个 Vue Component 的实例，每个组件实例的内部状态是不一样的。

2.构建virtual Dom
Vue Component 的渲染过程，可以简单理解为组件实例执行 render 函数生成 VNode 节点树的过程，也就是构建 Virtual DOM 的生成过程。自定义的组件在这个过程中被展开成了平台支持的节点，例如图中的 VNode 节点都是和平台提供的原生节点一一对应的，它的类型必须在 Weex 支持的原生组件范围内。

3.生成“真实”Dom
以上过程在 Weex 和浏览器里都是完全一样的，从生成真实 DOM 这一步开始，Weex 使用了不同的渲染方式。前面提到过 JS Framework 中提供了和 DOM 接口类似的 Weex DOM API，在 Vue 里会使用这些接口将 VNode 渲染生成适用于 Weex 平台的 Element 对象，和 DOM 很像，但并不是“真实”的 DOM。

4.发送渲染指令
在 JS Framework 内部和客户端渲染引擎约定了一系列的指令接口，对应了一个原子的 DOM 操作，如 addElement removeElement updateAttrs updateStyle 等。JS Framework 使用这些接口将自己内部构建的 Element 节点树以渲染指令的形式发给客户端。

5.绘制原生UI
客户端接收 JS Framework 发送的渲染指令，创建相应的原生组件，最终调用系统提供的接口绘制原生 UI。具体细节这里就不展开了。

## 事件响应过程
无论是在浏览器还是 Weex 里，事件都是由原生 UI 捕获的，然而事件处理函数都是写在前端里的，所以会有一个传递的过程。

如果在 Vue.js 里某个标签上绑定了事件，会在内部执行 addEventListener 给节点绑定事件，这个接口在 Weex 平台下调用的是 JS Framework 提供的 addEvent 方法向元素上添加事件，传递了事件类型和处理函数。JS Framework 不会立即向客户端发送添加事件的指令，而是把事件类型和处理函数记录下来，节点构建好以后再一起发给客户端，发送的节点中只包含了事件类型，不含事件处理函数。客户端在渲染节点时，如果发现节点上包含事件，就监听原生 UI 上的指定事件。

当原生 UI 监听到用户触发的事件以后，会派发 fireEvent 命令把节点的 ref、事件类型以及事件对象发给 JS Framework。JS Framework 根据 ref 和事件类型找到相应的事件处理函数，并且以事件对象 event 为参数执行事件处理函数。目前 Weex 里的事件模型相对比较简单，并不区分捕获阶段和冒泡阶段，而是只派发给触发了事件的节点，并不向上冒泡，类似 DOM 模型里 level 0 级别的事件。

上述过程里，事件只会绑定一次，但是很可能会触发多次，例如 touchmove 事件，在手指移动过程中，每秒可能会派发几十次，每次事件都对应了一次 fireEvent -> invokeHandler 的处理过程，很容易损伤性能，浏览器也是如此。针对这种情况，可以使用用 expression binding 来将事件处理函数转成表达式，在绑定事件时一起发给客户端，这样客户端在监听到原生事件以后可以直接解析并执行绑定的表达式，而不需要把事件再派发给前端。


## 演进方向
其实在 Weex 里，能跨多个渲染引擎通用的不止 JS Framework，还有 Weex Core，它们要解决的问题差不多，然而 JS Framework 是用 javascript 写的，Weex Core 是用 C/C++ 写的，实现的功能更底层一些，其实你可以将 JS Framework 理解为 js 版本的 Weex Core。不过 Weex Core 目前的功能还比较少，和 JS Framework 没有重叠，只包含了对 JS 引擎的优化和新的 CSS 布局引擎。

具体来讲，首先要做的就是在 Weex Core 中实现一份 DOM 接口，将会设计得更加标准、更加符合规范，有了原生的 document 、Element 这些对象以后，前端框架就可以直接调用原生接口不必经过 JS Framework，渲染性能会有提升，JS Framework 里的那颗不知道怎么称呼的树也就可以拿掉了，也不需要将节点发送客户端了，这样通信的逻辑也可以大幅简化。等把原生模块的调用、回调、事件响应这些问题解决了之后，JS-Native 之间的通信也可以拿掉了。