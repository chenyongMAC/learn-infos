## runtime动态添加实例变量

#### 1.不能向编译后得到的类中增加实例变量；
因为编译后的类已经注册在runtime中，类结构体中的objc_ivar_list 实例变量的链表和instance_size实例变量的内存大小已经确定，同时runtime 会调用class_setIvarLayout 或 class_setWeakIvarLayout来处理strong weak引用，所以不能向存在的类中添加实例变量

#### 2.能向运行时创建的类中添加实例变量
运行时创建的类是可以添加实例变量的。即使用runtime的api去创建一个类的过程中，有可用的api用于添加实例变量。

在objc_class中，所有的成员变量、属性的信息是放在链表ivars中的。ivars是一个数组，数组中每个元素是指向Ivar(变量信息)的指针。runtime提供了丰富的函数来操作这一字段。大体上可以分为以下几类：

```
// 获取类中指定名称实例成员变量的信息
Ivar class_getInstanceVariable ( Class cls, const char *name );
 
// 获取类成员变量的信息
Ivar class_getClassVariable ( Class cls, const char *name );
 
// 添加成员变量
BOOL class_addIvar ( Class cls, const char *name, size_t size, uint8_t alignment, const char *types );
 
// 获取整个成员变量列表
Ivar * class_copyIvarList ( Class cls, unsigned int *outCount );
```

#### 3.runtime提供的部分api参考

1）动态创建类：

```
// 创建一个新类和元类
Class objc_allocateClassPair ( Class superclass, const char *name, size_t extraBytes );
 
// 销毁一个类及其相关联的类
void objc_disposeClassPair ( Class cls );
 
// 在应用中注册由objc_allocateClassPair创建的类
void objc_registerClassPair ( Class cls );
```

2）动态创建对象

```
// 创建类实例
id class_createInstance ( Class cls, size_t extraBytes );
 
// 在指定位置创建类实例
id objc_constructInstance ( Class cls, void *bytes );
 
// 销毁类实例
void * objc_destructInstance ( id obj );
```
