## 1.理解“类对象”的用意
“在运行期检查对象类型”这一操作也叫“类型信息查询”（introspection，内省）。在OC中不要直接比较对象所属的类，明智的做法是调用“类型信息查询”方法。为什么呢？从这一点，我们展开对OC对象本质的探讨。

## 2.什么是OC对象

首先，我们要定义对象是什么，这样我们才能更好的区分什么样的“东西”是对象。

**OC中的对象，具有类型信息，能接受消息。**

从objc源码看，描述OC对象的数据结构定义如下。简单的理解，我们可以把objc_object理解为对象，把objc_class理解为类。我们可以得出，每一个对象都是一个类的实例。

```
typedef struct objc_class *Class;
typedef struct objc_object {
  Class isa;  //该变量定义了对象所属的类
} *id;

typedef stuct objc_class *Class;
struct objc_class {
  Class isa;
  Class super_class;
  const char *name;
  struct objc_method_list **methodLists;
  // ...略
};
```
不难看出，类（objc_class）的实现也包含了isa指针，它也能接受消息（即我们使用的类方法）。所以，我们又得出，每一个类也是一个对象。

#### 总结：
OC中的类和对象都是对象，他们都含有一个isa指针，指向他们的类。并且能接受消息。

## 3.OC的类继承关系
因为OC中的类和对象都是“对象”，所以他们拥有一个相似的继承关系，即：
**对象是一个类的实例, 类是元类（metaclass）的实例。**

#### 小结
实例对象：当我们new一个实例对象时，它只会拷贝类的成员变量，实例方法列表保存在类中。当调用实例方法时，会根据isa指针，找到实例对应的类，并在该类的实例方法列表中寻找对应的方法。

类对象：同理，类对象是元类的一个实例，类对象的方法保存在元类的类方法列表中。


#### 元类
元类也是一个OC对象，包含了isa指针，保存了类方法的列表。

https://upload-images.jianshu.io/upload_images/3088401-4d8fa334a29fdf63.png?imageMogr2/auto-orient/


## 4.类型信息查询

原理：它的原理是：使用isa指针获取对象所属的类，然后通过super_class指针在继承体系中游走

- isKindOfClass //判断出对象是否为某个特定类的实例
- isMemberOfClass //判断出对象是否为某类或其派生类的实例

源码实现如下：
```
+ (BOOL)isKindOfClass:(Class)cls {
    for (Class tcls = object_getClass((id)self); tcls; tcls = tcls->superclass) {
        if (tcls == cls) return YES;
    }
    return NO;
}

+ (BOOL)isMemberOfClass:(Class)cls {
    return object_getClass((id)self) == cls;
}

Class object_getClass(id obj)
{
    if (obj) return obj->getIsa();
    else return Nil;
}

inline Class 
objc_object::getIsa() 
{
    if (isTaggedPointer()) {
        uintptr_t slot = ((uintptr_t)this >> TAG_SLOT_SHIFT) & TAG_SLOT_MASK;
        return objc_tag_classes[slot];
    }
    return ISA();
}

inline Class 
objc_object::ISA() 
{
    assert(!isTaggedPointer()); 
    return (Class)(isa.bits & ISA_MASK);
}
```

 
