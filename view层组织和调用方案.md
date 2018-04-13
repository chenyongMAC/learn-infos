## 要做一个View层架构，主要就是从以下三方面入手：

1.制定良好的规范
2.选择好合适的模式（MVC、MVCS、MVVM、VIPER）
3.根据业务情况针对ViewController做好拆分，提供一些小工具方便开发




## View代码结构的规定
例如：在viewDidload里面只做addSubview的事情，然后在viewWillAppear里面做布局的事情，最后在viewDidAppear里面做Notification的监听之类的事情。至于属性的初始化，则交给getter去做。

其实针对View层的架构设计，还是要做好三点：代码规范，架构模式，工具集。
1.代码规范对于View层来说意义重大，毕竟View层非常重业务，如果代码布局混乱，后来者很难接手，也很难维护。
2.架构模式具体如何选择，完全取决于业务复杂度。如果业务相当相当复杂，那就可以使用VIPER，如果相对简单，那就直接MVC稍微改改就好了。
3.View层的工具集主要还是集中在如何对View进行布局，以及一些特定的View



## 关于在哪儿写Constraints？
苹果在文档中指出，updateViewConstraints是用来做add constraints的地方。
总：在viewDidLoad里面开一个layoutPageSubviews的方法，然后在这个里面创建Constraints并添加，会比较好。

```
- (void)viewDidLoad
{
    [super viewDidLoad];

    [self.view addSubview:self.firstView];
    [self.view addSubview:self.secondView];
    [self.view addSubview:self.thirdView];

    [self layoutPageSubviews];
}

- (void)layoutPageSubviews
{
    [self.view addConstraints:xxxConstraints];
    [self.view addConstraints:yyyConstraints];
    [self.view addConstraints:zzzConstraints];
}
```

PS：
其实在viewWillAppear这里改变UI元素不是很可靠，Autolayout发生在viewWillAppear之后，严格来说这里通常不做视图位置的修改，而用来更新Form数据。改变位置可以放在viewWilllayoutSubview或者didLayoutSubview里，而且在viewDidLayoutSubview确定UI位置关系之后设置autoLayout比较稳妥。另外，viewWillAppear在每次页面即将显示都会调用，viewWillLayoutSubviews虽然在lifeCycle里调用顺序在viewWillAppear之后，但是只有在页面元素需要调整时才会调用，避免了Constraints的重复添加。





## 业务方如何统一viewController
1.使用一个统一的派生类viewController
2.使用AOP



## MVC、MVVM、MVCS、VIPER
1）MVC
①M应该做的事：
给ViewController提供数据
给ViewController存储数据提供接口
提供经过抽象的业务基本组件，供Controller调度
②C应该做的事：
管理View Container的生命周期
负责生成所有的View实例，并放入View Container
监听来自View与业务有关的事件，通过与Model的合作，来完成对应事件的业务。
③V应该做的事：
响应与业务无关的事件，并因此引发动画效果，点击反馈（如果合适的话，尽量还是放在View去做）等。
界面元素表达


2）MVCS
苹果自身就采用的是这种架构思路，从名字也能看出，也是基于MVC衍生出来的一套架构。从概念上来说，它拆分的部分是Model部分，拆出来一个Store。这个Store专门负责数据存取。但从实际操作的角度上讲，它拆开的是Controller。
MVCS是基于瘦Model的一种架构思路，把原本Model要做的很多事情中的其中一部分关于数据存储的代码抽象成了Store，在一定程度上降低了Controller的压力。


3）MVVM
MVVM是基于胖Model的架构思路建立的，然后在胖Model中拆出两部分：Model和ViewModel。胖Model做的事情是先为Controller减负，然后由于Model变胖，再在此基础上拆出ViewModel，跟业界普遍认知的MVVM本质上是为Controller减负这个说法并不矛盾，因为胖Model做的事情也是为Controller减负。
ViewModel做什么事情？就是把RawData变成直接能被View使用的对象的一种Model。

RAC和MVVM
不用ReactiveCocoa也能MVVM，用ReactiveCocoa能更好地体现MVVM的精髓。
在MVVM中使用ReactiveCocoa的第一个目的就是如上所说，View并不适合直接持有ViewModel。第二个目的就在于，ViewModel有可能并不是只服务于特定的一个View，使用更加松散的绑定关系能够降低ViewModel和View之间的耦合度。

其实归根结底就是一句话：在MVC的基础上，把C拆出一个ViewModel专门负责数据处理的事情，就是MVVM。然后，为了让View和ViewModel之间能够有比较松散的绑定关系，于是我们使用ReactiveCocoa，因为苹果本身并没有提供一个比较适合这种情况的绑定方法。iOS领域里KVO，Notification，block，delegate和target-action都可以用来做数据通信，从而来实现绑定，但都不如ReactiveCocoa提供的RACSignal来的优雅，如果不用ReactiveCocoa，绑定关系可能就做不到那么松散那么好，但并不影响它还是MVVM。


4）VIPER
除了View层没拆分，其他都拆了。




## 关于胖Model和瘦Model
什么叫胖Model？
胖Model包含了部分弱业务逻辑。胖Model要达到的目的是，Controller从胖Model这里拿到数据之后，不用额外做操作或者只要做非常少的操作，就能够将数据直接应用在View上。
然而其缺点就在于，胖Model相对比较难移植，虽然只是包含弱业务，但好歹也是业务，迁移的时候很容易拔出萝卜带出泥。另外一点，MVC的架构思想更加倾向于Model是一个Layer，而不是一个Object，不应该把一个Layer应该做的事情交给一个Object去做。最后一点，软件是会成长的，FatModel很有可能随着软件的成长越来越Fat，最终难以维护。

什么叫瘦Model？
瘦Model只负责业务数据的表达，所有业务无论强弱一律扔到Controller。瘦Model要达到的目的是，尽一切可能去编写细粒度Model，然后配套各种helper类或方法来对弱业务做抽象，强业务依旧交给Controller。
由于Model的操作会出现在各种地方，瘦model在一定程度上违背了DRY（Don't Repeat Yourself）的思路，Controller仍然不可避免在一定程度上出现代码膨胀。




## 跨业务页面调用方案的设计
跨业务页面调用是指，当一个App中存在A业务，B业务等多个业务时，B业务有可能会需要展示A业务的某个页面，A业务也有可能会调用其他业务的某个页面。在小规模的App中，我们直接import其他业务的某个ViewController然后或者push或者present，是不会产生特别大的问题的。但是如果App的规模非常大，涉及业务数量非常多，再这么直接import就会出现问题。

如何处理？
让依赖关系下沉。

怎么让依赖关系下沉？引入Mediator模式。
所谓引入Mediator模式来让依赖关系下沉，实质上就是每次呼唤页面的时候，通过一个中间人来召唤另外一个页面，这样只要每个业务依赖这个中间人就可以了，中间人的角色就可以放在业务层的下面一层，这就是依赖关系下沉。




留给评论区各种补

总结