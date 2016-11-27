---
title: NSTimer（二）
date: 2016-11-27 11:42:40
tags: NSTimer
category: 能工巧匠
---

## 前言
上一篇博客讨论了 NSTimer 与 它的target 的关系。本篇博客将继续讨论 NSTimer。

## NSTimer并不会准时触发事件

在开发者文档[CFRunLoopTimer](https://developer.apple.com/reference/corefoundation/1666612-cfrunlooptimer)中有说明：
> A timer is not a real-time mechanism; it fires only when one of the run loop modes to which the timer has been added is running and able to check if the timer’s firing time has passed. If a timer’s firing time occurs while the run loop is in a mode that is not monitoring the timer or during a long callout, the timer does not fire until the next time the run loop checks the timer. Therefore, the actual time at which the timer fires potentially can be a significant period of time after the scheduled firing time.

翻译过来是：

> NSTimer不是一个实时系统，只有被添加到Runloop，并且到达触发时间才能够被触发。如果在当前Runloop中timer没有被检测到或者Runloop长时间无响应，该timer将不会被触发，直到下一个Runloop周期再次检测。因此，timer的实际触发时间有可能比预定触发时间稍晚。

假设一个 timer 指定3秒后触发某事件，但是当前线程进行很复杂很耗时的运算，timer后等到该运算结束后才会执行，如果延迟超过了一个周期，则会和后面的触发合并，即在一个周期内只会触发一次。即使 timer 延迟的时间超级长，timer后面的触发时间总是倍数于第一次添加timer的间隔。

看下面这个例子：


```objc
- (void)viewDidLoad {
    [super viewDidLoad];
    JHTestTimerObject *testObject2 = [[JHTestTimerObject alloc] init];
    [NSTimer scheduledTimerWithTimeInterval:1 target:testObject2 selector:@selector(timerAction:) userInfo:nil repeats:YES];
    
    [self performSelector:@selector(busyCalculation) withObject:nil afterDelay:3];
}

// 模拟当前线程正好繁忙的情况
- (void)busyCalculation {
    NSLog(@"start busy!");
    NSUInteger caculateCount = 0xFFFFFFFF;
    CGFloat uselessValue = 0;
    for (NSUInteger i = 0; i < caculateCount; ++i) {
        uselessValue = i / 0.3333;
    }
    NSLog(@"finish busy!");
}
```

看看打印结果：
{% asset_img Snip20161127_2.png print image %}

观察打印结果可以得知，当线程空闲的时候 timer 的触发还是比较准确，当开启大量计算时timer开始停止，等到计算结束后才开始触发消息，这个线程繁忙的过程超过了一个周期，但是timer并没有连着触发两次消息，而只是触发了一次。等线程忙完以后后面的消息触发的时间仍然都是整数倍与开始我们指定的时间，这也从侧面证明，timer并不会因为触发延迟而导致后面的触发时间发生延迟。

### 小结
NSTimer 并是不实时的机制，会发生延迟，延迟的程度和当前线程的 Runloop 有关。

## NSTimer要添加到RunLoop中才会有作用

也许有人问：前面那没多例子，没有一个添加到 Runloop 的，不照样能够定时触发吗？
我们没有手动添加，不代表系统 API 没有帮我们添加。

你可以尝试把 NSTimer 创建的 API 改为：

```objc
    [NSTimer timerWithTimeInterval:1 target:testObject2 selector:@selector(timerAction:) userInfo:nil repeats:YES];
```
你会发现 timer 并没有触发方法。

👆的代码改为：

```objc
_timer = [NSTimer timerWithTimeInterval:1 target:testObject2 selector:@selector(timerAction:) userInfo:nil repeats:YES];
[[NSRunLoop mainRunLoop] addTimer:_timer forMode:NSDefaultRunLoopMode];
```

会发现 timer 又可以正常的触发了。

总的来说这个和 NSTimer 初始化方式有关：

- **timerWithTimeInterval**  创建出来的 timer 如果手动调用 addTimer: forMode 方法加入主循环池中，将不会循环执行。
- **scheduledTimerWithTimeInterval**  创建的 timer 会自动将 timer 添加到当前的运行循环，运行循环的模式为默认模式。
- **init** 创建的 timer 需要手动加入循环池，它会在设定的启动时间启动。

## NSTimer 添加到 Runloop 中但迟迟不触发

原因主要是：`确认 Runloop 是否正常运行。`

每个线程都有它自己的 Runloop ，iOS 的主线程也有自己的 Runloop，前面创建的 timer 都是添加到主线程的 Runloop，所以都能正常的触发，但是如果是添加到自己的新建的线程，它的 Runloop 是不会自己运行起来的，我们需要手动把 Runloop 启动。

举个例子：

```objc
- (void)viewDidLoad {
    [super viewDidLoad];
    _thread = [[NSThread alloc]initWithTarget:self selector:@selector(testTimerRunloop) object:nil];
    //开启线程
    [_thread start];
}

- (void)testTimerRunloop {
    JHTestTimerObject *testObject2 = [[JHTestTimerObject alloc] init];
    NSTimer *timer = [NSTimer timerWithTimeInterval:1 target:testObject2 selector:@selector(timerAction:) userInfo:nil repeats:YES];
    [[NSRunLoop currentRunLoop] addTimer:timer forMode:NSDefaultRunLoopMode];
    
    // 打开下面一行输出runloop的内容就可以看出，timer却是已经被添加进去
//    NSLog(@"the thread's runloop: %@", [NSRunLoop currentRunLoop]);
    
    // 打开下面一行, 该线程的runloop就会运行起来，timer才会起作用
//    [[NSRunLoop currentRunLoop] run];
}
```

👆创建了一个子线程，并且启动了线程，然后在新建的线程中创建 timer 并添加到当前 Runloop，但打印结果：

{% asset_img Snip20161127_3.png print image %}

timer 直到销毁了都没触发，如果把注释打开，timer 就能正常触发了。

## 总结

NSTimer 的触发时机与当前线程有关（与线程 Runloop 有关）。


## 参考链接

* [NSTimer](http://justsee.iteye.com/blog/1774722)
* [iOS 中的 NSTimer](http://blog.callmewhy.com/2015/07/06/weak-timer-in-ios/)





