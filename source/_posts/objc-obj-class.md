title: Objective-C 类和对象
date: 2015-10-29 14:14:21
tags: Objective-C
categories: Objective-C
---
*Objective-C* 作为一门动态语言，给了程序员在运行期很大的自由度，要了解其机制，首先要从面向对象语言的基本元素对象和类说起。

## 对象
---
### 对象是什么
*Objective-C* 和 *C++* 都是产生于1980年代，而且又都是为了兼容 C 的同时，扩展出面向对象特性，但是思想是有不同的，*Objective-C* 更加注重类型的动态性，所有的方法调用都增加了一层，使用消息的方式，而非在编译时直接确定方法的内存地址，语法如下：
> [obj message]

那么，对象是什么呢？在 *Objective-C* 中，所有的对象都可以用`id`来表示，那`id`又是什么呢？在objc的头文件中有它的定义：
<!-- more -->
``` c
struct objc_object {
    Class isa  OBJC_ISA_AVAILABILITY;
};
typedef struct objc_object *id;
```
我们可以看到`id`是一个结构体指针，指向`objc_object`，而这个`objc_object`就是我们所说的对象了

### 对象内存结构
从定义来看，object只有一个`Class`类型的`isa`成员，至于`Class`到底是什么，后面会讲到，我们先看一下object在内存中到底是什么样子的，结构如下图：
![](/image/objc-obj-class/class-member.jpg)
最上面的是`isa`指针，下面依次是继承体系中基类以及派生类的成员变量，下面就来验证一下这个结构

首先定义类`A`和`B`，如下：

``` c
@interface A : NSObject {
@public
    int a;
}
@end
@implementation A
@end

@interface B : A {
@public
    int b;
}
@end
@implementation B
@end

B* b = [[B alloc] init];
```
先看一下`lldb`打印出来的结构

``` c
(lldb) p *b
(B) $0 = {
  A = {
    NSObject = {
      isa = B
    }
    a = 0
  }
  b = 0
}
```
从结果来看确实跟上图吻合，但是没有看到真正的内存结构还是不能确定。

下面打印对象`b`的成员，因为代码中不能直接访问`isa`指针，所以用`object_getClass`代替：

``` c
NSLog(@"b = %p, b->isa = %p, b->a = %p, b->b = %p", b, object_getClass(b), &b->a, &b->b);
NSLog(@"64bit in b addr : 0x%llx", *( u_int64_t*)(__bridge void*)b);
```

结果如下
> b = 0x100203cb0, b->isa = 0x1000012e8, b->a = 0x100203cb8, b->b = 0x100203cbc
> 64bit in b addr : 0x1d8001000012e9

可以看到，在64位环境下，`b->a`和`b->b`的偏移地址都是符合预期的，但是`b->isa`指针的地址看起来是错误的？没错，`0x1000012e8`确实不是`isa`的地址，而是`isa`指向的`类对象`的地址，暂时现不关注`类对象`，继续看下面打印的`b`的前8个字节，也就是`0x1d8001000012e9`，是不是感觉和`0x1000012e8`有些像呢？
在解释原因之前先看一下在64位情况下的`isa`指针都包含哪些信息：

```
(LSB)
1 bit    |  indexed           | 0 is raw isa, 1 is non-pointer isa.
1 bit    |  has_assoc         | Object has or once had an associated reference. Object with no associated references can deallocate faster.
1 bit    |  has_cxx_dtor      | Object has a C++ or ARC destructor. Objects with no destructor can deallocate faster.
30 bits  |  shiftcls          | Class pointer's non-zero bits.
9 bits   |  magic             | Equals 0xd2. Used by the debugger to distinguish real objects from uninitialized junk.
1 bit    |  weakly_referenced | Object is or once was pointed to by an ARC weak variable. Objects not weakly referenced can deallocate faster.
1 bit    |  deallocating      | Object is currently deallocating.
1 bit    |  has_sidetable_rc  | Object's retain count is too large to store inline.
19 bits  |  extra_rc          | Object's retain count above 1. (For example, if extra_rc is 5 then the object's real retain count is 6.)
(MSB)
```
虽然这个结构跟现在的有些区别，但还是有指示意义的，前33bit(去掉第一个bit)表示的才是真正的`isa`指向的地址，先假设`0x1d8001000012e9`就是`isa`指针，那么我们取其前33bit，也就是`0x1000012e9`，在去掉第一个bit，结果就是`0x1000012e8`，没错，和上面`object_getClass(b)`的地址是一样的，这里基本上可以确定对象内存结构中最上面的一个指针就是`isa`，但是为了再确认一次，可以在`lldb`中打印`isa`的地址

```
(lldb) p &b->isa
(Class *) $1 = 0x0000000100203cb0
```

没错，`0x0000000100203cb0`就是`b`的地址，也就是说对象最开始的位置就是`isa`指针，到这里基本上也就验证了对象的内存结构就是如上图所示的了，但是`isa`到底是个什么东西？在介绍之前我们先看看本文的另一个主题，<类>

## 类
---
### 类是什么

接触过面向对象的都知道类就是对事物的抽象，封装了属性以及方法，Objective-C 中使用`Class`表示，那么按照惯例先看一下`Class`的定义：
``` c
struct objc_class {
    Class isa  OBJC_ISA_AVAILABILITY;

#if !__OBJC2__
    Class super_class                                        OBJC2_UNAVAILABLE;
    const char *name                                         OBJC2_UNAVAILABLE;
    long version                                             OBJC2_UNAVAILABLE;
    long info                                                OBJC2_UNAVAILABLE;
    long instance_size                                       OBJC2_UNAVAILABLE;
    struct objc_ivar_list *ivars                             OBJC2_UNAVAILABLE;
    struct objc_method_list **methodLists                    OBJC2_UNAVAILABLE;
    struct objc_cache *cache                                 OBJC2_UNAVAILABLE;
    struct objc_protocol_list *protocols                     OBJC2_UNAVAILABLE;
#endif

} OBJC2_UNAVAILABLE;

typedef struct objc_class *Class;
```
结构体中下面的成员是为了兼容*OBJC1*的，虽然在 *OBJC2* 中不在这里定义了，但它们应该还是存在的，这里可以用它们来说明一下类的结构：
- `Class super_class` 指向父类的指针
- `const char *name` 类的名字, *object_getClassName*获取的应该就是它
- `long instance_size` 实例对象大小
- `struct objc_ivar_list *ivars` 实例变量列表
- `struct objc_method_list **methodLists` 指向方法列表，*Category*添加的方法列表应该就是加在这里的
- `struct objc_cache *cache` 方法缓存，在一定程度上提高消息响应速度，虽然还是比直接调用慢
- `struct objc_protocol_list *protocols` 实现的协议列表

通过上面这些，大概能理解当我们编写一个 *类* 代码的时候，编译器是怎么把它结构化的，当然编译器要做的肯定不止这些，这里只是示意一下。

接下来再看 *OBJC2* 中唯一一个成员`isa`， 怎么又是`isa`，这货怎么在 *类* 里也有？难道这里的 *类* 和 *对象* 有什么共同的特性？的确有，而且是跟 *C++* 大有不同的一个特性，*类* 也是一个 *对象*。

### 类对象
在 *Objective-C* 中，*对象* 是 *类* 的实例，*对象* 的`isa`正是指向其所属的 *类* 的，而 *类* 中的`isa`又是指向其所属的 *元类*，如本文开头所举的一个消息发送的例子：
> [obj message]

这里给`obj`发送消息，而`obj`对消息的响应方法就是在其所属的 *类* 中查找的，同样，向 *类* 发送的消息如：
> [class message]

响应方法则是在这个 *类对象* 所属的 *元类* 中查找的，下面说一下元类。

### 元类
*元类* 中会存储其实例 *类对象* 的方法，而 *元类* 本身也是一个对象，他所属的类是 *根元类*，也就是`NSObject`的 *元类*，不仅如此，所有的 *元类*， 不管继承层级是怎么样的，他们的 *元类* 都是 *根元类*。
看起来有点混乱，一图胜过千言，请看下图：
![](/image/objc-obj-class/class-diagram.jpg)
看到这里就清晰了吧，`superclass`指定了继承关系，`isa`指定了类型，各司其职，互不影响，但是还是需要再验证一下：

继续沿用之前的类定义

``` c
@interface A : NSObject {
@public
    int a;
}
@end
@implementation A
@end

@interface B : A {
@public
    int b;
}
@end
@implementation B
@end

B* b = [[B alloc] init];
```
迭代5次跟踪其`superclass`和`isa`指针

``` c
Class currentClass = [b class];
Class superClass = currentClass;
for (int i = 0; i < 5; i++)
{
    NSLog(@"isa - %p(%@) | superclass - %p(%@)", currentClass, currentClass, superClass , superClass);
    currentClass = object_getClass(currentClass);
    superClass = [superClass superclass];
}
```

结果如下

>isa - 0x1000012e8(B) | superclass - 0x1000012e8(B)
isa - 0x1000012c0(B) | superclass - 0x100001298(A)
isa - 0x7fff758f7118(NSObject) | superclass - 0x7fff758f70f0(NSObject)
isa - 0x7fff758f7118(NSObject) | superclass - 0x0((null))
isa - 0x7fff758f7118(NSObject) | superclass - 0x0((null))

很明显是符合图中所示的，这个就不解释了。

## isa
---
其实到这里对`isa`的作用比较清楚了，按白话来说，就是表示这个 *对象* 是个什么 *类*，*Objective-C* 的运行时多态就是通过`isa`来实现的，关于 *runtime* 更具体的原理和应用以后在专门来写吧，*类* 和 *对象* 的介绍就到这里了。
