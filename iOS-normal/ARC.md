Objective C的引用计数理解起来很容易，当一个对象被持有的时候计数加一，不再被持有的时候引用计数减一，当引用计数为零的时候，说明这个对象已经无用了，则将其释放。

引用计数分为两种：
1)手动引用计数（MRC）
2)自动引用计数（ARC）


## MRC

```
// MRC代码
NSObject * obj = [[NSObject alloc] init]; //引用计数为1
//不需要的时候
[obj release] //引用计数减1
//持有这个对象
[obj retain] //引用计数加1
//放到AutoReleasePool
[obj autorelease]//在auto release pool释放的时候，引用计数减1
```

## ARC
在Objective C中，有三种类型是ARC适用的
1)block
2)objective 对象，id, Class, NSError*等
3)由attribute((NSObject))标记的类型。

像double *,CFStringRef等不是ARC适用的，仍然需要手动管理内存。
Tips： 以CF开头的（Core Foundation）的对象往往需要手动管理内存。


## 内存管理方法

1.retain
通过SideTable这个数据结构来存储引用计数。这个数据结构就是存储了一个自旋锁，一个引用计数map。这个引用计数的map以对象的地址作为key，引用计数作为value。

2.release
查找map，对引用计数减1，如果引用计数小于阈值，则调用SEL_dealloc

3.autorelease
把对象存储到AutoreleasePoolPage的链表里。等到auto release pool被释放的时候，把链表内存储的对象删除
