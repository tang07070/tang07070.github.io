title: RunLoop介绍
date: 2015-11-21 23:33:58
tags: [RunLoop, iOS]
categories: iOS
---

当我们运行一个程序的时候，它能够持续响应用户的各种操作行为、timer事件，而不是处理完一个事件后就结束退出，这是为什么呢？简单来说，就是在主线程有一个死循环，让程序一直保持在"存活"状态，事件驱动（Event-Driven）机制都是基于这个原理，再加以扩充，比如windows的消息循环（Message Loop）机制，Chromium也封装了一套跨平台的MessageLoop作为线程模型的基础设施，可见其重要性，而在OSX/iOS中，实现此功能的就是RunLoop，下面就来看看RunLoop具体包括哪些内容。
<!-- more -->

## RunLoop与线程
每个线程都有一个关联的RunLoop对象，但除了主线程是在启动时就由系统创建(其实也是在调用`mainRunLoop`时创建)，其他线程都需要手动创建，也就是在调用`currentRunLoop`时创建，其对应的Core Foundation代码（最新的CF代码可以在[这里](http://opensource.apple.com/tarballs/CF/)获取）如下：

```
CFRunLoopRef CFRunLoopGetCurrent(void) {
    CHECK_FOR_FORK();
    CFRunLoopRef rl = (CFRunLoopRef)_CFGetTSD(__CFTSDKeyRunLoop);
    if (rl) return rl;
    return _CFRunLoopGet0(pthread_self());
}
```

具体创建工作是在`_CFRunLoopGet0`中进行的：

```
CF_EXPORT CFRunLoopRef _CFRunLoopGet0(pthread_t t) {
	 ...
    
    CFRunLoopRef loop = (CFRunLoopRef)CFDictionaryGetValue(__CFRunLoops, pthreadPointer(t));

    if (!loop) {
        CFRunLoopRef newLoop = __CFRunLoopCreate(t);
        loop = (CFRunLoopRef)CFDictionaryGetValue(__CFRunLoops, pthreadPointer(t));
        if (!loop) {
            CFDictionarySetValue(__CFRunLoops, pthreadPointer(t), newLoop);
            loop = newLoop;
        }
    }
	
	...

    return loop;
}
```

其大致思路很简单，当需要获取当前RunLoop对象时，先查表看看是否已经创建，如果创建了就直接返回，没创建就创建后写表并返回。

需要注意的是Core Foundation中的`CFRunLoop`是线程安全的，但是Cocoa中的`NSRunLoop`并不是线程安全的，所以如果要对`NSRunLoop`进行操作的话，一定要在其所在线程中进行，如果在一个线程中操作另一个线程的`NSRunLoop`对象，则会导致程序crash或者异常。

## RunLoop三大组成部分
在RunLoop中有三个很重要的组成部分：

+ RunLoopMode
+ RunLoopSource
+ RunLoopObserver

下面就分别介绍一下。

### RunLoopMode

#### RunLoopMode是什么
RunLoopMode可以认为是当前RunLoop的运行模式，在某一运行模式下，只处理与之对于的事件，根据类型分类，便于管理，下面看一下`__CFRunLoopMode`结构体都包含些什么：

```
struct __CFRunLoopMode {
    ...
    CFStringRef _name;
    CFMutableSetRef _sources0;
    CFMutableSetRef _sources1;
    CFMutableArrayRef _observers;
    CFMutableArrayRef _timers;
    ...
};
```

其中_name用来指定类型，_sources0、_sources1、_timers是RunLoopSource的一个集合，_observers是RunLoopObserver的一个集合，如此看来，RunLoopMode可以看做一个容器，包含当前RunLoop需要用到的Source以及Observer。

#### RunLoopMode有哪些类型
系统定义了如下五种Mode：

+ Default
 + 由`NSDefaultRunLoopMode`(Cocoa)、`kCFRunLoopDefaultMode`(Core Foundation)指定
 + 是大部分操作的默认mode
+ Modal
 + 由`NSConnectionReplyMode`(Cocoa)指定
 + 当有模态页面展示时的mode
+ Event tracking
 + 由`NSEventTrackingRunLoopMode`(Cocoa)指定
 + UI滑动时的mode
+ Connection
 + 由`NSConnectionReplyMode`(Cocoa)指定
 + `NSConnection`使用，其他情况基本不会用到
+ Common modes
 + 由`NSRunLoopCommonModes`(Cocoa)、`kCFRunLoopCommonModes`(Core Foundation)指定
 + 其中包括一些可配置的modes
 + Cocoa默认包括：default、modal、Event tracking
 + CF默认只包括default
 
RunLoopMode需要在RunLoop循环运行前指定，如果要更换Mode，则需要重启RunLoop循环。


### RunLoopSource
RunLoopSource可以认为是RunLoop循环要处理的事件源，包括Input Sources和Timer Sources，如官方文档中的图示：
![](https://developer.apple.com/library/ios/documentation/Cocoa/Conceptual/Multithreading/Art/runloop.jpg)


下面来分别看下这两种Sources。

#### Input Sources
在Input Sources下又分为以下三种Sources类型：

+ Port-based Sources
+ Custom Input Sources
+ Perform Selector Sources

Port-based Sources是基于mach_port(一种IPC机制)实现的，可以跨进程/线程发送消息，如果当时Runloop处于睡眠状态，该消息会将其激活。在Cocoa中可以使用`NSPort`添加一个port到Runloop中。

Custom Input Sources需要自己定义一些callback函数，包括被加入的Runloop、从Runloop中移除（被cancel）、被执行时等。

Perform Selector Sources是Cocoa定义的一种Custom Input Sources，跟Prot-based Sources类似，它可以进行跨线程调用，不同的是，Perform Selector Sources会在Runloop执行后被移除掉。

下面是Perform Selector系列方法中的一部分：

```
performSelectorOnMainThread:withObject:waitUntilDone:modes:
performSelector:onThread:withObject:waitUntilDone:modes:
performSelector:withObject:afterDelay:inModes:
```
Selector真正执行的时机会根据参数的不同而不同：

+ 目标thread是自己，并且waitUntilDone是YES时，该方法立即执行，即在当前RunLoop循环中；
+ 其他情况下，会在下一次Runloop循环中执行。

上面的三种类型是官方文档给出的，但是代码中实际上是两种分类，`source0`和`source1`，首先看下定义：

```
struct __CFRunLoopSource {
    CFRuntimeBase _base;
    uint32_t _bits;
    pthread_mutex_t _lock;
    CFIndex _order;			/* immutable */
    CFMutableBagRef _runLoops;
    union {
	CFRunLoopSourceContext version0;	/* immutable, except invalidation */
        CFRunLoopSourceContext1 version1;	/* immutable, except invalidation */
    } _context;
};
```
其中`_context`指定了是哪种类型，`CFRunLoopSourceContext`代表`source0`，代码如下：

```
struct CFRunLoopSourceContext {
    CFIndex version;
    void *info;
    CFAllocatorRetainCallBack retain;
    CFAllocatorReleaseCallBack release;
    CFAllocatorCopyDescriptionCallBack copyDescription;
    CFRunLoopEqualCallBack equal;
    CFRunLoopHashCallBack hash;
    CFRunLoopScheduleCallBack schedule;
    CFRunLoopCancelCallBack cancel;
    CFRunLoopPerformCallBack perform;
};
```
这里有很多CallBack函数，也就是Custom Input Sources需要实现的，所以`source0`指的就是Custom Input Sources。

再看一下代表`source1`的`CFRunLoopSourceContext1`的定义：

```
struct CFRunLoopSourceContext1 {
    CFIndex version;
    void *info;
    CFAllocatorRetainCallBack retain;
    CFAllocatorReleaseCallBack release;
    CFAllocatorCopyDescriptionCallBack copyDescription;
    CFRunLoopEqualCallBack equal;
    CFRunLoopHashCallBack hash;
    CFRunLoopGetPortCallBack getPort;
    CFRunLoopMachPerformCallBack perform;
};
```
`CFRunLoopSourceContext1`比`CFRunLoopSourceContext`唯一多的字段是`getPort`，正是Port-based Sources需要实现的。

#### Timer Sources

我们平时使用的`NSTimer`就是一种Timer Sources，需要注意的是，Timer的执行并不是精确的，Runloop会在每次循环中执行已到期的Timer，并且只有在当前Runloop Mode与timer指定的Mode相符时才执行，如果不相符，则会在Runloop切换到对应的Mode后才会被执行，当然，如果Runloop一直没有切换到对于Mode，那么这些Timer将永远得不到机会执行。

Timer分为两种：一次性的和周期性的，其中周期性Timer触发时机基于Timer指定的时间，并不会受到上一次执行延迟的影响，比如：一个间隔为5s的Timer，它的触发时机就是每5s触发一次，如果第一次是7s时才得到触发，那么第二次依旧是第10s时触发，而不是第12s。

下面附上`__CFRunLoopTimer`的定义：

```
struct __CFRunLoopTimer {
    CFRuntimeBase _base;
    uint16_t _bits;
    pthread_mutex_t _lock;
    CFRunLoopRef _runLoop;
    CFMutableSetRef _rlModes;
    CFAbsoluteTime _nextFireDate;
    CFTimeInterval _interval;		/* immutable */
    CFTimeInterval _tolerance;          /* mutable */
    uint64_t _fireTSR;			/* TSR units */
    CFIndex _order;			/* immutable */
    CFRunLoopTimerCallBack _callout;	/* immutable */
    CFRunLoopTimerContext _context;	/* immutable, except invalidation */
};
```

### RunLoopObserver

RunLoop提供了一种观察者模式，让用户可以很方便的监听RunLoop循环中各个重要阶段的转换，支持的阶段如下：

```
/* Run Loop Observer Activities */
typedef CF_OPTIONS(CFOptionFlags, CFRunLoopActivity) {
    kCFRunLoopEntry = (1UL << 0),	  // 进入循环
    kCFRunLoopBeforeTimers = (1UL << 1),  // 处理Timer Sources前 
    kCFRunLoopBeforeSources = (1UL << 2), // 处理Input Sources前
    kCFRunLoopBeforeWaiting = (1UL << 5), // 进入睡眠状态前
    kCFRunLoopAfterWaiting = (1UL << 6),  // 退出睡眠状态后
    kCFRunLoopExit = (1UL << 7),          // 退出循环
    kCFRunLoopAllActivities = 0x0FFFFFFFU // 所有阶段
};
```
Observer也是Mode相关的，只有RunLoop运行在指定Mode下，相应的Observer才会被通知。

添加Observer需要在Core Foundation中，下面代码演示如何添加一个Observer：

```
    NSRunLoop* myRunLoop = [NSRunLoop currentRunLoop];
 
    // Create a run loop observer and attach it to the run loop.
    CFRunLoopObserverContext  context = {0, self, NULL, NULL, NULL};
    CFRunLoopObserverRef    observer = CFRunLoopObserverCreate(kCFAllocatorDefault,
            kCFRunLoopAllActivities, YES, 0, &myRunLoopObserver, &context);
 
    if (observer)
    {
        CFRunLoopRef    cfLoop = [myRunLoop getCFRunLoop];
        CFRunLoopAddObserver(cfLoop, observer, kCFRunLoopDefaultMode);
    }
```

同样，下面附上`__CFRunLoopObserver`的定义：

```
struct __CFRunLoopObserver {
    CFRuntimeBase _base;
    pthread_mutex_t _lock;
    CFRunLoopRef _runLoop;
    CFIndex _rlCount;
    CFOptionFlags _activities;		/* immutable */
    CFIndex _order;			/* immutable */
    CFRunLoopObserverCallBack _callout;	/* immutable */
    CFRunLoopObserverContext _context;	/* immutable, except invalidation */
};
```

## RunLoop循环
在了解了RunLoop中三个重要的组成部分后，我们来看一下RunLoop在每个循环中都做了些什么。

首先看一下RunLoop的启动函数`CFRunLoopRunSpecific`：

```
SInt32 CFRunLoopRunSpecific(CFRunLoopRef rl, CFStringRef modeName, CFTimeInterval seconds, Boolean returnAfterSourceHandled) {     /* DOES CALLOUT */
	...

	if (currentMode->_observerMask & kCFRunLoopEntry ) __CFRunLoopDoObservers(rl, currentMode, kCFRunLoopEntry);
	result = __CFRunLoopRun(rl, currentMode, seconds, returnAfterSourceHandled, previousMode);
	if (currentMode->_observerMask & kCFRunLoopExit ) __CFRunLoopDoObservers(rl, currentMode, kCFRunLoopExit);
	
	...
    return result;
}
```

其中`__CFRunLoopRun`是RunLoop循环所在，在之前和之后会分别触发进入和退出RunLoop的观察者。`__CFRunLoopRun`函数体很大，有三百多行！再看代码之前，先了解一下大体流程，如下图：
![](/image/runloop/runloop-sequence.png)

下面是只保留重要部分的代码，有兴趣的可以看一下：

```
static int32_t __CFRunLoopRun(CFRunLoopRef rl, CFRunLoopModeRef rlm, CFTimeInterval seconds, Boolean stopAfterHandle, CFRunLoopModeRef previousMode) {
    do {

        if (rlm->_observerMask & kCFRunLoopBeforeTimers) __CFRunLoopDoObservers(rl, rlm, kCFRunLoopBeforeTimers);
        if (rlm->_observerMask & kCFRunLoopBeforeSources) __CFRunLoopDoObservers(rl, rlm, kCFRunLoopBeforeSources);

	__CFRunLoopDoBlocks(rl, rlm);

        Boolean sourceHandledThisLoop = __CFRunLoopDoSources0(rl, rlm, stopAfterHandle);
        if (sourceHandledThisLoop) {
            __CFRunLoopDoBlocks(rl, rlm);
	}

        Boolean poll = sourceHandledThisLoop || (0ULL == timeout_context->termTSR);

        if (MACH_PORT_NULL != dispatchPort && !didDispatchPortLastTime) {
            msg = (mach_msg_header_t *)msg_buffer;
            if (__CFRunLoopServiceMachPort(dispatchPort, &msg, sizeof(msg_buffer), &livePort, 0, &voucherState, NULL)) {
                goto handle_msg;
            }
        }

	if (!poll && (rlm->_observerMask & kCFRunLoopBeforeWaiting)) __CFRunLoopDoObservers(rl, rlm, kCFRunLoopBeforeWaiting);

        __CFRunLoopServiceMachPort(waitSet, &msg, sizeof(msg_buffer), &livePort, poll ? 0 : TIMEOUT_INFINITY, &voucherState, &voucherCopy);

	if (!poll && (rlm->_observerMask & kCFRunLoopAfterWaiting)) __CFRunLoopDoObservers(rl, rlm, kCFRunLoopAfterWaiting);

handle_msg:;
  
        if (MACH_PORT_NULL == livePort) {
            CFRUNLOOP_WAKEUP_FOR_NOTHING();
        } else if (livePort == rl->_wakeUpPort) {
            CFRUNLOOP_WAKEUP_FOR_WAKEUP();
        }
        else if (livePort == dispatchPort) {
            CFRUNLOOP_WAKEUP_FOR_DISPATCH();
        } else {
            CFRUNLOOP_WAKEUP_FOR_SOURCE();
		}
	__CFRunLoopDoBlocks(rl, rlm);
        
	if (sourceHandledThisLoop && stopAfterHandle) {
	    retVal = kCFRunLoopRunHandledSource;
        } else if (timeout_context->termTSR < mach_absolute_time()) {
            retVal = kCFRunLoopRunTimedOut;
	} else if (__CFRunLoopIsStopped(rl)) {
            __CFRunLoopUnsetStopped(rl);
	    retVal = kCFRunLoopRunStopped;
	} else if (rlm->_stopped) {
	    rlm->_stopped = false;
	    retVal = kCFRunLoopRunStopped;
	} else if (__CFRunLoopModeIsEmpty(rl, rlm, previousMode)) {
	    retVal = kCFRunLoopRunFinished;
	}
    } while (0 == retVal);

    if (timeout_timer) {
        dispatch_source_cancel(timeout_timer);
        dispatch_release(timeout_timer);
    } else {
        free(timeout_context);
    }

    return retVal;
}

```

## 总结
RunLoop作为程序运行的基础，了解其本质是很有必要的，本文只是总结了RunLoop的一些基本概念，以及重要的三个组成部分和大致流程，更多更深入的了解可以去看官方文档和源代码。

参考：
[[1]https://developer.apple.com/library/ios/documentation/Cocoa/Conceptual/Multithreading/RunLoopManagement/RunLoopManagement.html](https://developer.apple.com/library/ios/documentation/Cocoa/Conceptual/Multithreading/RunLoopManagement/RunLoopManagement.html)
[[2]http://opensource.apple.com/tarballs/CF/CF-1153.18.tar.gz](http://opensource.apple.com/tarballs/CF/CF-1153.18.tar.gz)