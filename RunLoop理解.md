### 学习RunLoop理解

#### 通过http://blog.ibireme.com/2015/05/18/runloop/来学习RunLoop。

1.iOS的RunLoop底层是基于CFRunLoop的，苹果通过RunLoop实现了自动释放池、延迟回调、触摸事件。屏幕刷新等功能。

2.上述博客中将RunLoop的学习分为了

RunLoop的概念

RunLoop与线程的关系

RunLoop对外的接口

RunLoop的Mode

RunLoop的内部逻辑

RunLoop的底层实现

苹果用RunLoop实现的功能

​	AutorealeasePool

​	事件响应

​	手势识别

​	界面更新

​	定时器

​	PerformSelecter

​	关于GCD

​	关于网络请求

RunLoop的实际应用举例

​	AFNetworking	

​	AsyncDisplayKit

#### RunLoop的概念

​	RunLoop实际上就是一个对象，这个对象管理了其需要处理的事件和消息，在管理事件/消息时，使得线程在没有处理消息时休眠以避免资源占用、在有消息来到时立刻被唤醒。它提供了一个入口函数，线程执行了这个函数后，就会一直处于这个函数内部“接受消息->等待->处理”的循环中，知道这个循环结束，函数返回。

​	iOS系统中，提供了两个这样的对象：NSRunLoop和CFRunLoopRef

​	CFRunLoopRef是在CoreFoundation框架内的，它提供了纯C函数的API，所有这些API都是线程安全的。

​	NSRunLoop是基于RunLoopRef的封装。提供了面向对象的API，但是这些API不是线程安全的。

​	CFRunLoopRef的[开源代码]([http://opensource.apple.com/tarballs/CF/](http://opensource.apple.com/tarballs/CF/))在这里。

​	swift开源后开源了一个跨平台的[CoreFoundation版本]([https://github.com/apple/swift-corelibs-foundation/](https://github.com/apple/swift-corelibs-foundation/)).

#### RunLoop与线程的关系

​	pthread和NSThread是一一对应的。

​	pthread_main_thread_np()—————>[NSThread mainThread]

​	pthread_self()———————————>[NSThread currentThread]

​	CFRunLoop是基于pthread来管理的。

​	苹果不允许直接创建RunLoop，它只提供两个自动获取的函数：CFRunLoopGetMain()和CFRunLoopGetCurrent().

```/// 全局的Dictionary，key 是 pthread_t， value 是 CFRunLoopRef```

```static CFMutableDictionaryRef loopsDic;```

```/// 访问 loopsDic 时的锁```

```static CFSpinLock_t loopsLock;```

 

```/// 获取一个 pthread 对应的 RunLoop。```

```CFRunLoopRef _CFRunLoopGet(pthread_t thread) {```

 ```   OSSpinLockLock(&loopsLock);```

    

 ```   if (!loopsDic) {```

```        // 第一次进入时，初始化全局Dic，并先为主线程创建一个 RunLoop。```

      ```  loopsDic = CFDictionaryCreateMutable();```

       ``` CFRunLoopRef mainLoop = _CFRunLoopCreate();```

        ```CFDictionarySetValue(loopsDic, pthread_main_thread_np(), mainLoop);```

    ```}```

    

 ```   /// 直接从 Dictionary 里获取。```

```    CFRunLoopRef loop = CFDictionaryGetValue(loopsDic, thread));```

    

 ```   if (!loop) {```

```        /// 取不到时，创建一个```

   ```     loop = _CFRunLoopCreate();```

   ```     CFDictionarySetValue(loopsDic, thread, loop);```

      ```  /// 注册一个回调，当线程销毁时，顺便也销毁其对应的 RunLoop。```

     ```   _CFSetTSD(..., thread, loop, __CFFinalizeRunLoop);```

```    }```

    

    ```OSSpinLockUnLock(&loopsLock);```

 ```   return loop;```

```}```

 

```CFRunLoopRef CFRunLoopGetMain() {```

   ``` return _CFRunLoopGet(pthread_main_thread_np());```

```}```

 

```CFRunLoopRef CFRunLoopGetCurrent() {```

    ```return _CFRunLoopGet(pthread_self());```

```}```

​	线程和RunLoop是一一对象的，线程是key，runloop是value，保存在一个全局的字典中。线程刚创建时并没有 RunLoop，如果你不主动获取，那它一直都不会有。RunLoop的创建时发生在第一次获取时，RunLoop的销毁是发生在线程结束时。、、、只能在一个线程的内部获取其RunLoop（主线程除外）？？？。

#### RunLoop对外的接口

![](http://blog.ibireme.com/wp-content/uploads/2015/05/RunLoop_0.png)

​	一个 RunLoop 包含若干个 Mode，每个 Mode 又包含若干个Source/Timer/Observer。每次调用 RunLoop 的主函数时，只能指定其中一个 Mode，这个Mode被称作 CurrentMode。如果需要切换 Mode，只能退出 Loop，再重新指定一个 Mode 进入。这样做主要是为了分隔开不同组的 Source/Timer/Observer，让其互不影响。

​	实际上 RunLoop 就是这样一个函数，其内部是一个 do-while 循环。当你调用 CFRunLoopRun() 时，线程就会一直停留在这个循环里；直到超时或被手动停止，该函数才会返回。

#### RunLoop的底层实现

​	RunLoop 的核心是基于 mach port 的，其进入休眠时调用的函数是 mach_msg()。

![](http://blog.ibireme.com/wp-content/uploads/2015/05/RunLoop_3.png)

​	RunLoop 的核心就是一个 mach_msg() (见上面代码的第7步)，RunLoop 调用这个函数去接收消息，如果没有别人发送 port 消息过来，内核会将线程置于等待状态。例如你在模拟器里跑起一个 iOS 的 App，然后在 App 静止时点击暂停，你会看到主线程调用栈是停留在 mach_msg_trap() 这个地方。

```
{
    /// 1. 通知Observers，即将进入RunLoop
    /// 此处有Observer会创建AutoreleasePool: _objc_autoreleasePoolPush();
    __CFRUNLOOP_IS_CALLING_OUT_TO_AN_OBSERVER_CALLBACK_FUNCTION__(kCFRunLoopEntry);
    do {
 
        /// 2. 通知 Observers: 即将触发 Timer 回调。
        __CFRUNLOOP_IS_CALLING_OUT_TO_AN_OBSERVER_CALLBACK_FUNCTION__(kCFRunLoopBeforeTimers);
        /// 3. 通知 Observers: 即将触发 Source (非基于port的,Source0) 回调。
        __CFRUNLOOP_IS_CALLING_OUT_TO_AN_OBSERVER_CALLBACK_FUNCTION__(kCFRunLoopBeforeSources);
        __CFRUNLOOP_IS_CALLING_OUT_TO_A_BLOCK__(block);
 
        /// 4. 触发 Source0 (非基于port的) 回调。
        __CFRUNLOOP_IS_CALLING_OUT_TO_A_SOURCE0_PERFORM_FUNCTION__(source0);
        __CFRUNLOOP_IS_CALLING_OUT_TO_A_BLOCK__(block);
 
        /// 6. 通知Observers，即将进入休眠
        /// 此处有Observer释放并新建AutoreleasePool: _objc_autoreleasePoolPop(); _objc_autoreleasePoolPush();
        __CFRUNLOOP_IS_CALLING_OUT_TO_AN_OBSERVER_CALLBACK_FUNCTION__(kCFRunLoopBeforeWaiting);
 
        /// 7. sleep to wait msg.
        mach_msg() -> mach_msg_trap();
        
 
        /// 8. 通知Observers，线程被唤醒
        __CFRUNLOOP_IS_CALLING_OUT_TO_AN_OBSERVER_CALLBACK_FUNCTION__(kCFRunLoopAfterWaiting);
 
        /// 9. 如果是被Timer唤醒的，回调Timer
        __CFRUNLOOP_IS_CALLING_OUT_TO_A_TIMER_CALLBACK_FUNCTION__(timer);
 
        /// 9. 如果是被dispatch唤醒的，执行所有调用 dispatch_async 等方法放入main queue 的 block
        __CFRUNLOOP_IS_SERVICING_THE_MAIN_DISPATCH_QUEUE__(dispatched_block);
 
        /// 9. 如果如果Runloop是被 Source1 (基于port的) 的事件唤醒了，处理这个事件
        __CFRUNLOOP_IS_CALLING_OUT_TO_A_SOURCE1_PERFORM_FUNCTION__(source1);
 
 
    } while (...);
 
    /// 10. 通知Observers，即将退出RunLoop
    /// 此处有Observer释放AutoreleasePool: _objc_autoreleasePoolPop();
    __CFRUNLOOP_IS_CALLING_OUT_TO_AN_OBSERVER_CALLBACK_FUNCTION__(kCFRunLoopExit);
}
```

#### RunLoop的实际应用举例

​	AFNetworking

​	AsyncDisplayKit



##### 看别人的博客，作为 记录， 也是锻炼markdown的写作语法，所以写了本篇文章。

