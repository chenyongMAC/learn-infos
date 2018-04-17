## 1.用途
1.添加私有属性，用于更好地实现细节（声明在实现文件中）
2.添加公有属性，增强category的功能（声明在头文件中）
3.创建一个用于KVO的关联观察者


## 2.实现

```
//1.添加关联对象，传入nil时，相当于清空该关联对象的值
void objc_setAssociatedObject(id object, const void *key, id value, objc_AssociationPolicy policy);
//2.获取关联对象
id objc_getAssociatedObject(id object, const void *key);
//3.移除所有关联对象，一般不使用，这样会把所有的关联对象都移除。
void objc_removeAssociatedObjects(id object);
```

#### 在分类中到底能否实现属性？
1）如果你把属性理解为通过方法访问的实例变量，我相信这个问题的答案是不能，因为分类不能为类增加额外的实例变量。
2）如果属性只是一个存取方法以及存储值的容器的集合，那么分类是可以实现属性的。分类中对属性的实现其实只是实现了一个看起来像属性的接口而已。

#### key值定义
key值的获取：key值必须是一个对象级别的唯一常量。通常有三种写法：
1）声明static char kAssociatedObejctKey；使用&kAssociatedObejctKey作为Key
2）声明static void *kAssociatedObejctKey = &kAssociatedObejctKey；使用kAssociatedObejctKey 作为Key
3）使用selector，用getter方法作为Key。例如@selector(函数名称)，更简单就是使用_cmd（代表当前方法的选择子）。这样可以去为Key命名的烦恼，推荐！


## 3.原理
（选修）hash map


## 4.销毁
无论在MRC下还是ARC下均不需要。被关联的对象在生命周期内要比对象本身释放的晚很多，它们会在被 NSObject -dealloc 调用的object_dispose()方法中释放。

#### 对象的内存销毁时间表：
1.release
	调用[self dealloc]
	
2.[super dealloc]
	释放MRC的实例变量们（iVars）
	[super dealloc]
	
3.[NSObject dealloc]
	这一步只会去调用OC runtime中的object_dispose()
	
4.object_dispose()
	为C++的实例变量调用destructors
	为ARC的实例变量们（iVars）调用release
	解除所有associated object
	解除所有__weak
	调用free()



