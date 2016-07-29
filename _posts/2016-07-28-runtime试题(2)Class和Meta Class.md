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

### id和Class

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
