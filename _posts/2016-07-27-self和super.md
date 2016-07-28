---
layout: post
title:  "self和super"
date:   2016-07-27 23:41:37
categories: runtime
---

有一组关于runtime的题目,神经病院objc runtime入院考试:http://blog.sunnyxx.com/2014/11/06/runtime-nuts/

网上有对这些问题的一些答案:http://chun.tips/blog/2014/11/05/bao-gen-wen-di-objective%5Bnil%5Dc-runtime(1)%5Bnil%5D-self-and-super/

同时,通过查阅资料和实验,自己也对这些问题也做了一些解答.

### 第一题

(1) 下面的代码输出什么？

```
@implementation Son : Father
- (id)init {
    self = [super init];
    if (self) {
        NSLog(@"%@", NSStringFromClass([self class]));
        NSLog(@"%@", NSStringFromClass([super class]));
    }
    return self;
}
@end

```
答案:

通过运行代码，我们能看到打印结果为:

```
2016-07-28 22:07:31.197 self和super[7400:885003] Son
2016-07-28 22:07:31.198 self和super[7400:885003] Son
```

### 代码解析

利用clang命令把oc代码编译成底层的runtime实现

```
clang -rewrite-objc Son.m
```

编译后

```
NSLog(@"%@", NSStringFromClass([self class]));
NSLog(@"%@", NSStringFromClass([super class]));

编译后的代码为:
NSLog((NSString *)&__NSConstantStringImpl__var_folders_11_6m7430zx1kv5x5682xvld9p80000gn_T_Son_873ef2_mi_0, NSStringFromClass(((Class (*)(id, SEL))(void *)objc_msgSend)((id)self, sel_registerName("class"))));
       
       
NSLog((NSString *)&__NSConstantStringImpl__var_folders_11_6m7430zx1kv5x5682xvld9p80000gn_T_Son_873ef2_mi_1, NSStringFromClass(((Class (*)(__rw_objc_super *, SEL))(void *)objc_msgSendSuper)((__rw_objc_super){(id)self, (id)class_getSuperclass(objc_getClass("Son"))}, sel_registerName("class"))));


把打印和强转去掉后:
objc_msgSend((id)self, sel_registerName("class"))
objc_msgSendSuper((__rw_objc_super){(id)self, (id)class_getSuperclass(objc_getClass("Son"))}, sel_registerName("class"))));


```

关于 objc_msgSend

```
/** 
 * Sends a message with a simple return value to an instance of a class.
 * 
 * @param 参数self是当前类的实例,用来接受消息
 * @param 参数op 消息selector
 */
OBJC_EXPORT id objc_msgSend(id self, SEL op, ...)

```
所以objc_msgSend((id)self, sel_registerName("class"))的方法调用者为son,消息为class.首先去son中找class方法的实现,没有。则去父类father中找,仍然没有.最后在NSObject中找到方法的实现.而消息的接收者为son的实例,所以打印为son.



关于objc_msgSendSuper

```
/** 
 * @param 参数super 是一个指向objc_super结构体的指针
 * @param 参数op 消息selector
 */
OBJC_EXPORT id objc_msgSendSuper(struct objc_super *super, SEL op, ...)

super是指向结构体objc_super的指针


struct objc_super {
    /// 消息的接收实例
    __unsafe_unretained id receiver;

    /// Specifies the particular superclass of the instance to message. 
#if !defined(__cplusplus)  &&  !__OBJC2__
    /* 在 old objc-runtime.h header 中有效*/
    __unsafe_unretained Class class;
#else
    __unsafe_unretained Class super_class;
#endif
    /* super_class is the first class to search */
};

```
所以objc_msgSendSuper((__rw_objc_super){(id)self, (id)class_getSuperclass(objc_getClass("Son"))}, sel_registerName("class"))))方法,首先转换为结构体

```
objc_super {
   
    __unsafe_unretained id son;
    
    __unsafe_unretained Class Father;

};

```
然后去父类找class的实例,没有.最后在NSObject中找到class的实现,而消息的接收者为son。所以打印结果为son.


### 总结:self和super的区别

self是指当前方法调用者的实例,当调用某个方法时,首先从当前的类中寻找改方法的实例,没有则再去其父类中寻找方法实现.直至找到基类中寻找方法的实现,如果还没有找到,则会调用方法的动态解析.而super调用某个方法时,则会从当前类的父类开始寻找方法的实现,没有则会一级一级的往上找,直至找到基类为止,若没有找到,就会掉用方法的动态解析.而本问题中son和father都没有实现class这个方法,所以都会调用基类NSObject的class方法.而这个方法的接受者2个都是son，所以最后的打印同为son.




