# DCLagMonitor

卡顿问题，就是在主线程上无法响应用户交互的问题。
导致卡顿问题的几种原因：
- 复杂 UI 、图文混排的绘制量过大； 
- 在主线程上做网络同步请求；
- 在主线程做大量的 IO 操作；
- 运算量过大，CPU 持续高占用；
- 死锁和主子线程抢锁。

监控卡顿就是要去找到主线程上都做了哪些事，线程的消息事件是依赖于 NSRunLoop 的，通过监听 NSRunLoop 的状态，就能够发现调用方法是否会出现卡顿。

# RunLoop 原理
RunLoop 这个对象，在 iOS 里由 CFRunLoop 实现。简单来说，RunLoop 是用来监听输入源，进行调度处理的。这里的输入源可以是输入设备、网络、周期性或者延迟时间、异步回调。

RunLoop 会接收两种类型的输入源：一种是来自另一个线程或者来自不同应用的异步消息（Source1）；另一种是来自预订时间或者重复间隔的同步事件（Source0）。

接下来，通过 CFRunLoop 的源码来跟你分享下 RunLoop 的原理吧。

查看 CFRunLoop 的完整代码可以点击 [这个链接](https://opensource.apple.com/source/CF/CF-1153.18/CFRunLoop.c.auto.html)

下面是简化保留主流程的代码：

```objectiveC
// 共外部调用的公开的CFRunLoopRun方法，其内部会调用CFRunLoopRunSpecific
void CFRunLoopRun(void) {   /* DOES CALLOUT */
    int32_t result;
    do {
        result = CFRunLoopRunSpecific(CFRunLoopGetCurrent(), kCFRunLoopDefaultMode, 1.0e10, false);
    } while (kCFRunLoopRunStopped != result && kCFRunLoopRunFinished != result);
}

// 经过精简的 CFRunLoopRunSpecific 函数代码，其内部会调用__CFRunLoopRun函数
SInt32 CFRunLoopRunSpecific(CFRunLoopRef rl, CFStringRef modeName, CFTimeInterval seconds, Boolean returnAfterSourceHandled) {     /* DOES CALLOUT */

    // 通知Observers : 进入Loop
    // __CFRunLoopDoObservers内部会调用 __CFRUNLOOP_IS_CALLING_OUT_TO_AN_OBSERVER_CALLBACK_FUNCTION__
函数
    if (currentMode->_observerMask & kCFRunLoopEntry ) __CFRunLoopDoObservers(rl, currentMode, kCFRunLoopEntry);
    
    // 核心的Loop逻辑
    result = __CFRunLoopRun(rl, currentMode, seconds, returnAfterSourceHandled, previousMode);
    
    // 通知Observers : 退出Loop
    if (currentMode->_observerMask & kCFRunLoopExit ) __CFRunLoopDoObservers(rl, currentMode, kCFRunLoopExit);

    return result;
}

// 精简后的 __CFRunLoopRun函数，保留了主要代码
int32_t __CFRunLoopRun()
{
    // 通知即将进入runloop
    __CFRunLoopDoObservers(KCFRunLoopEntry);
    
    do
    {
        // 通知将要处理timer和source
        __CFRunLoopDoObservers(kCFRunLoopBeforeTimers);
        __CFRunLoopDoObservers(kCFRunLoopBeforeSources);
        
        // 处理非延迟的主线程调用
        __CFRunLoopDoBlocks();
        // 处理Source0事件
        __CFRunLoopDoSource0();
        
        if (sourceHandledThisLoop) {
            __CFRunLoopDoBlocks();
         }
        /// 如果有 Source1 (基于port) 处于 ready 状态，直接处理这个 Source1 然后跳转去处理消息。
        if (__Source0DidDispatchPortLastTime) {
            Boolean hasMsg = __CFRunLoopServiceMachPort();
            if (hasMsg) goto handle_msg;
        }
            
        /// 通知 Observers: RunLoop 的线程即将进入休眠(sleep)。
        if (!sourceHandledThisLoop) {
            __CFRunLoopDoObservers(runloop, currentMode, kCFRunLoopBeforeWaiting);
        }
            
        // GCD dispatch main queue
        CheckIfExistMessagesInMainDispatchQueue();
        
        // 即将进入休眠
        __CFRunLoopDoObservers(kCFRunLoopBeforeWaiting);
        
        // 等待内核mach_msg事件
        mach_port_t wakeUpPort = SleepAndWaitForWakingUpPorts();
        
        // 等待。。。
        
        // 从等待中醒来
        __CFRunLoopDoObservers(kCFRunLoopAfterWaiting);
        
        // 处理因timer的唤醒
        if (wakeUpPort == timerPort)
            __CFRunLoopDoTimers();
        
        // 处理异步方法唤醒,如dispatch_async
        else if (wakeUpPort == mainDispatchQueuePort)
            __CFRUNLOOP_IS_SERVICING_THE_MAIN_DISPATCH_QUEUE__()
            
        // 处理Source1
        else
            __CFRunLoopDoSource1();
        
        // 再次确保是否有同步的方法需要调用
        __CFRunLoopDoBlocks();
        
    } while (!stop && !timeout);
    
    // 通知即将退出runloop
    __CFRunLoopDoObservers(CFRunLoopExit);
}
```

# 卡顿检测

通过对 RunLoop 原理的分析，我们可以看出在整个过程中，loop 的状态包括 6 个，其代码定义如下：
```objectiveC
typedef CF_OPTIONS(CFOptionFlags, CFRunLoopActivity) {
    kCFRunLoopEntry , // 进入 loop
    kCFRunLoopBeforeTimers , // 触发 Timer 回调
    kCFRunLoopBeforeSources , // 触发 Source0 回调
    kCFRunLoopBeforeWaiting , // 等待 mach_port 消息
    kCFRunLoopAfterWaiting ), // 接收 mach_port 消息
    kCFRunLoopExit , // 退出 loop
    kCFRunLoopAllActivities  // loop 所有状态改变
}
```
如果 RunLoop 的主线程，进入睡眠前方法的执行时间过长而导致无法进入睡眠，或者线程唤醒后接收消息时间过长而无法进入下一步的话，就可以认为是线程受阻了，表现出来的就是出现了卡顿。
要想监听 RunLoop，你就首先需要创建一个CFRunLoopObserverContext 观察者，添加到主线程 RunLoop 的 common 模式下观察。一旦发现进入睡眠前的 kCFRunLoopBeforeSources 状态，或者唤醒后的状态 kCFRunLoopAfterWaiting，在设置的时间阈值内一直没有变化，即可判定为卡顿。接下来，我们就可以 dump 出堆栈的信息，从而进一步分析出具体是哪个方法的执行时间过长。

具体的实现代码如下：

```objectiveC

// 开始监测卡顿
- (void)beginMonitor {
 
    if (runLoopObserver) {
        return;
    }
    dispatchSemaphore = dispatch_semaphore_create(0); //Dispatch Semaphore保证同步
    //创建一个观察者
    CFRunLoopObserverContext context = {0,(__bridge void*)self,NULL,NULL};
    runLoopObserver = CFRunLoopObserverCreate(kCFAllocatorDefault,
                                              kCFRunLoopAllActivities,
                                              YES,
                                              0,
                                              &runLoopObserverCallBack,
                                              &context);
    //将观察者添加到主线程runloop的common模式下的观察中
    CFRunLoopAddObserver(CFRunLoopGetMain(), runLoopObserver, kCFRunLoopCommonModes);
    
    //创建子线程监控
    dispatch_async(dispatch_get_global_queue(0, 0), ^{
        //子线程开启一个持续的loop用来进行监控
        while (YES) {
            // 设置2秒超时
            long semaphoreWait = dispatch_semaphore_wait(dispatchSemaphore, dispatch_time(DISPATCH_TIME_NOW, 2*NSEC_PER_SEC));
            if (semaphoreWait != 0) {
                if (!runLoopObserver) {
                    timeoutCount = 0;
                    dispatchSemaphore = 0;
                    runLoopActivity = 0;
                    return;
                }
                //两个runloop的状态，BeforeSources和AfterWaiting这两个状态区间时间能够检测到是否卡顿
                if (runLoopActivity == kCFRunLoopBeforeSources || runLoopActivity == kCFRunLoopAfterWaiting) {
                    //出现三次出结果
                    if (++timeoutCount < 3) {
                        continue;
                    }
                    
                    // 处理卡顿
                    dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_HIGH, 0), ^{
                        // 打印堆栈信息
                        
                    });
                }
            }// end semaphore wait
            timeoutCount = 0;
        }// end while
    });
    
}

// runloop 监听回调
static void runLoopObserverCallBack(CFRunLoopObserverRef observer, CFRunLoopActivity activity, void *info){
    SMLagMonitor *lagMonitor = (__bridge SMLagMonitor*)info;
    lagMonitor->runLoopActivity = activity;
    
    dispatch_semaphore_t semaphore = lagMonitor->dispatchSemaphore;
    dispatch_semaphore_signal(semaphore);
}

// 结束监听
- (void)endMonitor {
    if (!runLoopObserver) {
        return;
    }
    CFRunLoopRemoveObserver(CFRunLoopGetMain(), runLoopObserver, kCFRunLoopCommonModes);
    CFRelease(runLoopObserver);
    runLoopObserver = NULL;
}

```

# Matrix-iOS 卡顿监控
Matrix检测卡顿功能主要在WCBlockMonitorMgr下面。
目前微信使用的卡顿监控，主程序 Runloop 超时的阈值是 2 秒，子线程的检查周期是 1 秒。每隔 1 秒，子线程检查主线程的运行状态；如果检查到主线程 Runloop 运行超过 2 秒则认为是卡顿，并获得当前的线程快照。


__builtin_expect这个指令是gcc引入的，作用是允许程序员将最有可能执行的分支告诉编译器。
这个指令的写法为：__builtin_expect(EXP, N)。
意思是：EXP==N的概率很大。

通过这种方式，编译器在编译过程中，会将可能性更大的代码紧跟着起面的代码，从而减少指令跳转带来的性能上的下降。

一般的使用方法是将__builtin_expect指令封装为likely和unlikely宏。这两个宏的写法如下.

```objectiveC
// Compiler hints for "if" statements
#define likely_if(x) if(__builtin_expect(x,1))
#define unlikely_if(x) if(__builtin_expect(x,0))


int ksjson_addIntegerElement(KSJSONEncodeContext* const context,
                             const char* const name,
                             int64_t value)
{
    int result = ksjson_beginElement(context, name);
    unlikely_if(result != KSJSON_OK)
    {
        return result;
    }
    char buff[30];
    sprintf(buff, "%" PRId64, value);
    return addJSONData(context, buff, (int)strlen(buff));
}

```





