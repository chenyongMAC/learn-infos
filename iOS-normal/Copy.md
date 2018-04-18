## 1.copy和mutablecopy
1）copy拷贝出来的对象类型总是不可变类型(例如, NSString, NSDictionary, NSArray等等)

2）mutableCopy拷贝出来的对象类型总是可变类型(例如, NSMutableString, NSMutableDictionary, NSMutableArray等等)


## 2.深拷贝和浅拷贝

1.非集合类对象

```
[immutableObject copy] // 浅复制
[immutableObject mutableCopy] //深复制
[mutableObject copy] //深复制
[mutableObject mutableCopy] //深复制
```

2.集合类对象

```
[immutableObject copy] // 浅复制
[immutableObject mutableCopy] //单层深复制
[mutableObject copy] //单层深复制
[mutableObject mutableCopy] //单层深复制
```

## 2.property的copy
1.目的：获得不可变副本
因为父类指针可以指向子类对象,使用 copy 的目的是为了让本对象的属性不受外界影响,使用 copy 无论给我传入是一个可变对象还是不可对象,我本身持有的就是一个不可变的副本.如果我们使用是 strong ,那么这个属性就有可能指向一个可变对象,如果这个可变对象在外部被修改了,那么会影响该属性.
