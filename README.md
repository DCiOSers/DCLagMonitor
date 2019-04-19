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






