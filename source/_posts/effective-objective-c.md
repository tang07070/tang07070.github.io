title: Effective Objective-C 读书笔记
date: 2015-11-06 21:37:13
tags: Objective-C
categories: Objective-C
---
在学习C++的时候，读过Effective系列（Effective C++ / More Effective C++）书籍，其中虽然没有深入的原理实现，也没有多酷炫的奇淫巧技，大都是作者在多年工程经验中对如何用好一门语言的总结。通过它，经验丰富者可以对已在心中存在，但是还有些混乱的知识进行梳理沉淀，真正的化为己有；新手则可以按照书中所写作为今后的一个参考，提醒自己如何写出优秀的代码。

Effective Objective-C自然也是必读之一，在刚开始学习Objective-C不久的时候大概翻了一遍，没怎么用心，自然也没记住多少，只是混个眼熟。近来使用Objective-C开发也有一段时间，所以又拿起来重新读了一遍，趁着思维导图工具[iThoughtX](https://itunes.apple.com/cn/app/ithoughtsx/id720669838)特价时入了一个，正好作为辅助读书的工具，相比以前不爱记笔记的习惯，确实很不错。[iThoughtX](https://itunes.apple.com/cn/app/ithoughtsx/id720669838)支持导出很多种格式，包括MarkDown，可以直接放到博客上，真是太方便了。

这是脑图导出的图片，比较大，如果需要看的话，建议下载到本地或者[单独打开](/image/effective-objective-c/Effective-oc.png)。
<!--more-->

![](/image/effective-objective-c/Effective-oc.png)

------
下面是导出的MarkDown，也方便自己没事复习一下

## 第一章：熟悉object-c


### 条例1：了解oc起源

* 消息结构
    * 由Smalltalk演化
    * 运行时决定行为
* 运行期组建
    * 动态库
* C的超集
    * 引用计数

### 条例2：减少头文件引用

* 延后引用时机
* @class声明

### 条例3：多用字面语法

* 字面数值
    * @1
    * @YES
    * @2.5f
* 字面数组
    * @[@“cat”, @“dog”]
    * animals[index]
* 字面字典
    * @{@“key1”:@“value1”, @“key2”:@“value2”}
    * dict[key]

### 条例4：常量代替define

* 包含类型信息
* static定义可见范围

### 条例5：枚举表示状态

* NS_ENUM
* NS_OPTIONS
* switch不加default
    * 避免新增状态被默认处理


## 第二章：对象、消息、运行期

### 条例6：理解『属性』

* @property
    * 原子性
        * atomic
        * nonatomic
    * 读写权限
        * readwrite
            * 自动合成getter、setter
        * readonly
            * 只有setter
    * 内存管理
        * assign
            * 简单赋值
        * strong
            * 拥有关系
            * 保留新值，释放旧值
        * weak
            * 非拥有关系
            * 不保留新值，不释放旧值
            * 对象销毁时，属性置空
        * unsafe_unretained
            * 同weak，但不置空
        * copy
            * 不保留新值，只拷贝
            * 派生类为Mutable***时必须使用
    * 方法名
        * getter=<name>
        * setter=<name>

### 条例7：尽量直接访问实例变量

* 读取时，直接访问
    * 快
* 写入时，使用setter
    * 触发KVO
    * 符合内存管理语意，如copy
* init，dealloc，直接访问
    * 避免子类覆盖setter方法

### 条例8：对象等同性

* isEqual
    * 相同对象hash必然相等
* hash
    * hash相等对象不一定相同
    * 选择计算速度快，碰撞几率低的算法

### 条例9：『类族模式』隐藏细节

* 把实现细节隐藏到公共接口后面
* 系统框架经常使用类族

### 条例10：既有类中使用关联对象存放自定义数据

* func
    * objc_setAssociateObject
    * objc_getAssociateObject
    * objc_removeAssociateObject
* 连接两个对象
* 拥有内存语意
* 万不得已再使用

### 条例11：理解objc_msgSend

* objc_msgSend(id obj, SEL cmd, …)
    * 与对象方法参数一致，为了利用『尾调用优化』
* 从消息接受对象向上找方法，没找到的话执行消息转发

### 条例12：理解消息转发机制

* resolveInstanceMethod
    * forwardingTargetForSelector
        * forwardInvocation

### 条例13：method swizzling

* id (* IMP)(id, SEL, …)
* 运行期向类中新增或替换SEL对应的实现
* 使用另一份实现替换原有实现，叫做method swizzling

### 条例14：理解『类对象』的用意

* id就是objc_object，只有一个isa指针，指向objc_class
* objc_class的isa指向元类
* isMemberOfClass(是否为特定类实例)
* isKindOfClass(是否为某类或其派生类实例)


## 第三章：接口和API设计

### 条例15：用前缀避免命名空间冲突

* 苹果保留『两字母前缀』权利，尽量用三字母
* C函数也要加上前缀
* 若自己的程序库引用三方库，应为三方库加上前缀

### 条例16：提供『全能初始化方法』

* 提供全能初始化方法，其他初始化方法应该调用此方法
* 如果超类的初始化方法不适合子类，子类应该覆写或抛出异常

### 条例17：实现description方法

* 返回一个有意义的字符串，用来描述该实例
* 调试器对应debugDescription，默认调用description

### 条例18：尽量使用不可变对象

* 若某属性仅能内部修改，interface中设置为readonly
* 不要公开可变collection，提供方法修改他

### 条例19：使用清晰而协调的命名方式

* 遵循命名规范，更容易被开发者理解
* 方法名言简意赅，从左至右像日常用语
* 不要使用缩略后的类型名称
* 起名风格要与自己的代码或集成的框架相符

### 条例20：为私有方法名加前缀

* 区分公有方法
* 不要单用一个下划线，这是苹果预留的

### 条例21：理解Objective-C错误类型

* 可能导致崩溃的严重错误再使用异常
* 错误不严重的情况下，可使用delegate来处理NSError

### 条例22：理解NSCopying协议

* copyWithZone的zone指内存分区，现在只有一个分区了，可忽略
* 一般情况下采用浅拷贝，Foundation的collection默认都是浅拷贝
* 如需要深拷贝，考虑新增一个方法专门执行深拷贝


## 第四章：协议和分类


### 条例23：通过委托与数据源协议进行对象间通信

* 当某对象需要从其他对象获取数据时，这种委托模式又叫做『数据源协议』
* 若有必要，可以缓存委托对象是否实现协议

### 条例24：将类代码分散到分类中

* 按功能分类，易于管理
* 私有方法可以归入分类中，以隐藏实现细节

### 条例25：为第三方类的分类加前缀

* 避免分类覆盖
* 方法名同样要加前缀

### 条例26：勿在分类中声明属性

* 在主接口中定义全部属性
* 分类中可以使用*AssociatedObject方法，但并不推荐，除非不得已
* 在分类中用存取方法代替属性

### 条例27：使用『class-continuation』隐藏实现细节

* 唯一能声明实例变量的分类，而且没有实现文件，没有名字，必须定义在主实现文件中
* 隐藏不必要暴露的属性
* 与C++混用时能保证接口是纯OC的
* 可以在此将readonly扩充为readwrite
* 在此实现的协议可以不暴露给外界

### 条例28：通过协议提供匿名对象

* 实际上就是针对接口编程
    * 这里的接口就是protocol

## 第五章：内存管理


### 条例29：理解引用计数

* 『引用树的』的『根对象』
    * Mac OS X：NSApplication
    * iOS：UIApplication
* release后避免悬挂，指针置空
* 顺序很重要：先保留在释放
* autorelease延长对象生命周期
    * 通常在下一次loop中递减
* 弱引用避免保留环

### 条例30：以ARC简化引用计数

* 修饰符指定内存管理语意
    * __strong
    * __unsafe_unretained
    * __weak
    * __autoreleasing
* 省去许多样板代码
* CoreFoundation对象不归ARC管理，需要自己管理
    * CFRetain
    * CFRelease

### 条例31：在dealloc方法中只释放引用并解除监听

* 决不能主动调用dealloc
* 不要在dealloc中调用其他方法，尤其涉及到异步处理的
* 不要在dealloc中调用存取方法，可能触发kvo，而observer可能使用这个即将回收的对象，导致异常

### 条例32：编写『异常安全代码』时留意内存管理问题

* C++和Objective-C的异常是兼容的
* ARC默认不处理异常状态下的内存管理
    * 可以使用-fobjc-arc-exceptions打开，但会造成大量的样板代码，影响效率
* 若非致命错误，用NSError或错误信息来代替异常

### 条例33：以弱引用避免保留环

* 使用弱引用
    * weak
        * 对象回收后指向nil
    * unsafe_unreatined
        * 对象回收后指向不变

### 条例34：以『自动释放池块』降低内存峰值

* 主线程、GCD线程，默认有autoreleasepool
* 合理运用，可降低内存峰值
    * 如循环中

### 条例35：用『僵尸对象』调试内存管理问题

* 开启功能后，对象不被回收，而是转为僵尸对象
* 通过修改对象的isa指针，指向僵尸类
    * 对象可以响应消息，响应方式是打印相关信息并终止程序

### 条例36：不要使用retainCount

* 此方法在ARC中已被废弃

## 第六章：Block与GCD


### 条例37：理解『Block』这一概念

* return_type (^block_name)(parameters)
* 栈block、堆block、全局block

### 条例38：为常用block创建typedef


### 条例39：用handler block降低代码分散程度

* 创建对象时，使用handler block将业务逻辑一起声明
* 相比delegate，在多实例时，block不必区分是哪个实例
* 设计API时，可以给block指定一个NSOperationQueue

### 条例40：用block引用其所属对象时不要出现保留环

* API设计者需要在适当的时机解除保留环，不要把责任给API调用者

### 条例41：多用GCD，少用同步锁

* 同步与异步派发结合，可以实现同步锁的行为
* 并发队列中使用barrier可以让同一时间只执行一个task

### 条例42：多用GCD，少用performSelector系列方法

* performSelector在内存管理方面有疏失
    * 无法确定SEL具体是什么，所以ARC无法对应管理
* performSelector能处理的SEL太局限
* performSelector能做的，GCD都能做的更好

### 条例43：掌握GCD及操作队列的使用时机

* 操作队列相比GCD的优点
    * 任务未启动时可取消
    * 指定依赖关系、执行顺序
    * 可以使用KVO，如isCancelled、isFinished
    * 指定优先级
    * 重用NSOperation对象，DRY原则
* GCD是纯C的API，操作队列是Objective-C的
    * 操作队列底层基于GCD实现

### 条例44：通过Dispatch Group机制，根据系统资源状态来执行任务

* 通过group，可以以组为单位控制任务
* dispatch group可以并发执行，GCD负责调度，比自己实现方便很多

### 条例45：使用dispatch_once来执行只需执行一次的线程安全代码

* 单例模式
* 线程安全

### 条例46：不要使用dispatch_get_current_queue

* iOS6.0以后已弃用，只应做调试之用
* 派发队列是有层级关系的

## 第七章：系统框架


### 条例47：熟悉系统框架

* Cocoa不是框架，但是它包含很多框架
    * Foundation
        * NSObject、NSXxxxxxx
            * NS为NeXTSTEP
    * CoreFoundation
        * 提供很多Foundation的底层实现
    * CFNetwork
    * CoreAudio
    * AVFoundation
    * CoreData
    * CoreText
    * …

### 条例48：多用块枚举，少用for循环

* NSEnumerator
    * 遍历时可以忽略类型
    * 可以反向遍历
* 快速遍历
    * for(xx in collection)
    * 简单高效
    * 无法获取下标
* 基于块的遍历方式
    * 基于GCD
    * 可以修改回调Block中的类型

### 条例49：对自定义内存管理的collection使用无缝桥接

* 可以在Foundation框架对象和CoreFoundation框架对象中转换

### 条例50：构建缓存时选用NSCache而非NSDictionary

* NSCache在系统资源耗尽时自行删减，NSDictionary则需要手动处理
* NSCache是线程安全的
* 设置上限
    * 并非硬限制，仅指导作用
* 只缓存值得缓存的东西

### 条例51：精简initialize和load的实现代码

* load会在程序加载阶段调用
* initialize会在首次使用某个类时调用
* 实现中避免依赖其他模块，减少依赖环几率
* 可以在initialize中设置无法在编译期设置的常量

### 条例52：NSTimer会保留其目标对象

* 避免出现保留环
    * 用block

