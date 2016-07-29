---
layout: post
title:  "runtime试题(2)Class和Meta Class"
date:   2016-07-28 14:59:30
categories: runtime
---

神经病院objc runtime入院考试,第二题关于Class和Meta Class

### 第二题

(2) 下面代码的结果？

```
BOOL res1 = [(id)[NSObject class] isKindOfClass:[NSObject class]];
BOOL res2 = [(id)[NSObject class] isMemberOfClass:[NSObject class]];
BOOL res3 = [(id)[Sark class] isKindOfClass:[Sark class]];
BOOL res4 = [(id)[Sark class] isMemberOfClass:[Sark class]];

```

答案: 
打印的结果为：

```
1 0 0 0

```

### 一, id和Class

再解决此问题前,首先要明白id和Class这2个概念

#### 1 Class
在objc.h文件中,Class的定义为指向objc_class结构体的指针,它代表了一个类。

```
/// An opaque type that represents an Objective-C class.
/// 代表一个Objective-C的类
typedef struct objc_class *Class;

```

关于结构体objc_class,它的定义在runtime.h中

```
struct objc_class {
    Class isa  OBJC_ISA_AVAILABILITY;  //meta class

#if !__OBJC2__
    Class super_class      //父类                                   
    const char *name       //类名                                  
    long version           //版本                                 
    long info               //类信息                                 
    long instance_size      //实例变量的大小                                
    struct objc_ivar_list *ivars    //成员变量列表                          
    struct objc_method_list **methodLists     //方法列表                 
    struct objc_cache *cache                  //方法缓存列表                     
    struct objc_protocol_list *protocols      //声明的协议列表                    
    
    OBJC2_UNAVAILABLE;
#endif

} OBJC2_UNAVAILABLE;

```


#### 2  id

在objc.h文件中,我们能看到id的定义,它是一个指向objc_object结构体的指针

```
/// A pointer to an instance of a class.
/// 指向一个类实例对象的指针
typedef struct objc_object *id;

```
而关于objc_object这个结构体,它代表一个类的实例.

```
/// Represents an instance of a class.
/// 代表一个类的实例
struct objc_object {
    Class isa  OBJC_ISA_AVAILABILITY;  //指向类
};

```

类本事也是一个对象,称为类对象,它的isa指针指向该类的meta class.而实例变量的isa指针指向它所属的类.

关于类的isa指针,superclass等关系如下图:

<span><img src="\images\id和class\class.png"></span>


### 二, isKindOfClass:和isMemberOfClass:

要解答这个问题,还需要了解的是isKindOfClass:和isMemberOfClass:这2个方法的实现

#### 1 isKindOfClass:

在Object.mm中,能找到这两个方法的实现

```
- (BOOL)isKindOf:aClass
{
	Class cls;
	for (cls = isa; cls; cls = cls->superclass) 
		if (cls == (Class)aClass)
			return YES;
	return NO;
}

```

通过这个实现，我们发现这里是有一个遍历,首先初始赋值为isa指针所指的对象,然后一层一层的父类,直到为nil。如果发现与aClass相等,则返回YES。若整个for循环之后都找不到相等,则返回NO.

懂了这个之后,然后参照上图再来看这个题目:

```
BOOL res1 = [(id)[NSObject class] isKindOfClass:[NSObject class]];
BOOL res3 = [(id)[Sark class] isKindOfClass:[Sark class]];

```

第一个 [NSObject class]的isa指针指向NSObject的meta Class,与[NSObject class]不相等,继续遍历到NSObject的meta Class的super class为[NSObject class].与[NSObject class]相等,所以返回1.

第二个 [Sark class]的isa指针指向Sark的meta Class,与[Sark class]不相等,一层层像上遍历superclass为NSObject的meta Class,[NSObject class].都与[Sark class]不相等,所以最后返回0.

#### 2 isMemberOfClass:

```
- (BOOL)isMemberOf:aClass
{
	return isa == (Class)aClass;
}
```

isMemberOfClass这个方法就是直接比较isa指针所指的类与aClass是否相等.

在来看题目:

```
BOOL res2 = [(id)[NSObject class] isMemberOfClass:[NSObject class]];
BOOL res4 = [(id)[Sark class] isMemberOfClass:[Sark class]];

```
第一个 [NSObject class]的isa指针指向NSObject的meta Class,与[NSObject class]不相等,所以返回0.

第二个 [Sark class]的isa指针指向Sark的meta Class,与[Sark class]不相等,所以也返回0.