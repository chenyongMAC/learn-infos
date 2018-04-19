## 实现原理
AutoreleasePool并没有单独的结构，而是由若干个AutoreleasePoolPage以双向链表的形式组合而成（分别对应结构中的parent指针和child指针）。

AutoreleasePool是按线程一一对应的,每一个线程都会有一个autoreleasepool。当某一个线程结束时，释放池内的东西会被自动释放。

AutoreleasePoolPage是一个C++实现的类，每个对象会开辟4096字节内存（也就是虚拟内存一页的大小），除了自身实例变量所占空间，剩下的空间全部用来储存autorelease对象的地址。一个AutoreleasePoolPage的空间被占满时，会新建一个AutoreleasePoolPage对象，连接链表


#### @autoReleasePool {}
编译器将其改写为：

```
void *context = objc_autoreleasePoolPush(); 

//some codes...

objc_autoreleasePoolPop(context);
```

#### ARC下的优化机制
假设有如下代码实现：

```
+ (instancetype)createSark {
    return [self new];
}
// caller
Sark *sark = [Sark createSark];
```

ARC优化后，实际如下：

```
+ (instancetype)createSark {
    id tmp = [self new];
    return objc_autoreleaseReturnValue(tmp); // 代替我们调用autorelease
}
// caller
id tmp = objc_retainAutoreleasedReturnValue([Sark createSark]) // 代替我们调用retain
Sark *sark = tmp;
objc_storeStrong(&sark, nil); // 相当于代替我们调用了release
```

## 需要手动创建autoreleasepool
1.写的程序不是基于UIFrameWork，例如命令行项目

2.写的循环大量创建临时对象。可以在循环中创建autoreleasepool，在池子中创建对象。这样有助于降低内存峰值。PS: 容器版本的block枚举器（enumerateObjectUsingBlock）会在内部自动添加pool。所以不需要手动添加。

3.创建了一个新的线程，线程开始执行时，需要立刻创建一个autoreleasepool


##多层嵌套的pool
pool的操作由以下2个api提供：

```
objc_autoreleasePoolPush
objc_autoreleasePoolPop
```

每当调用一次Push方法时，runtime向当前的AutoreleasePoolPage中add进一个哨兵对象，值为0（也就是个nil）。调用Pop时，会把当前next指针位置到上一个哨兵值位置之间的对象都释放掉，并重新记录next指针。
所以，多层嵌套就会重复多次上面的操作。


## autorelease对象
#### 释放方式：
向一个对象发送autorelease消息，就是将这个对象加入到当前AutoreleasePoolPage的栈顶next指针指向的位置。所以这个对象的释放就听从于autoreleasepool了。

#### 释放时机：
Autorelease对象出了作用域之后，会被添加到最近一次创建的自动释放池中，并会在当前的 runloop 迭代结束时释放。
