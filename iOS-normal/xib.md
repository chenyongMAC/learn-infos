## view上的weak property

我们平时定义控件属性的时候一般都会用strong修饰符，而我们在用xib，sb拖控件的时候会发现，这时属性都是用的weak修饰符。

原因：
代码创建的控件，被view所持有，所以要用strong。
xib、sb创建的控件，被他们自己持有，而不是view。所以在view中要用weak。但是使用strong也不一定会发生内存泄漏。viewController释放时，它持有的xib和view通常都会被释放，则控件也会被释放。但是多层view的场景中，控件是可能存在不被释放的情况的。