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
        CHECK_FOR_FORK();
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
static int32_t __CFRunLoopRun(CFRunLoopRef rl, CFRunLoopModeRef rlm, CFTimeInterval seconds, Boolean stopAfterHandle, CFRunLoopModeRef previousMode) {
    int32_t retVal = 0;
    do {
        // 通知Observers：即将处理Timers
        __CFRunLoopDoObservers(rl, rlm, kCFRunLoopBeforeTimers); 
        
        // 通知Observers：即将处理Sources
        __CFRunLoopDoObservers(rl, rlm, kCFRunLoopBeforeSources);
        
        // 处理Blocks
        __CFRunLoopDoBlocks(rl, rlm);
        
        // 处理Sources0
        if (__CFRunLoopDoSources0(rl, rlm, stopAfterHandle)) {
            // 处理Blocks
            __CFRunLoopDoBlocks(rl, rlm);
        }
        
        // 如果有Sources1，就跳转到handle_msg标记处
        if (__CFRunLoopServiceMachPort(dispatchPort, &msg, sizeof(msg_buffer), &livePort, 0, &voucherState, NULL)) {
            goto handle_msg;
        }
        
        // 通知Observers：即将休眠
        __CFRunLoopDoObservers(rl, rlm, kCFRunLoopBeforeWaiting);
        
        // 进入休眠，等待其他消息唤醒
        __CFRunLoopSetSleeping(rl);
        __CFPortSetInsert(dispatchPort, waitSet);
        do {
            __CFRunLoopServiceMachPort(waitSet, &msg, sizeof(msg_buffer), &livePort, poll ? 0 : TIMEOUT_INFINITY, &voucherState, &voucherCopy);
        } while (1);
        
        // 醒来
        __CFPortSetRemove(dispatchPort, waitSet);
        __CFRunLoopUnsetSleeping(rl);
        
        // 通知Observers：已经唤醒
        __CFRunLoopDoObservers(rl, rlm, kCFRunLoopAfterWaiting);
        
    handle_msg: // 看看是谁唤醒了RunLoop，进行相应的处理
        if (被Timer唤醒的) {
            // 处理Timer
            __CFRunLoopDoTimers(rl, rlm, mach_absolute_time());
        }
        else if (被GCD唤醒的) {
            __CFRUNLOOP_IS_SERVICING_THE_MAIN_DISPATCH_QUEUE__(msg);
        } else { // 被Sources1唤醒的
            __CFRunLoopDoSource1(rl, rlm, rls, msg, msg->msgh_size, &reply);
        }
        
        // 执行Blocks
        __CFRunLoopDoBlocks(rl, rlm);
        
        // 根据之前的执行结果，来决定怎么做，为retVal赋相应的值
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
    
    return retVal;
}
```






