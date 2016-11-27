---
title: NSTimer（一）
date: 2016-11-19 12:42:49
tags: NSTimer
category: 能工巧匠
---

## 前言
NSTimer iOS中使用的计时器，定时去完成任务的东西。但是其中的坑点相当多：

1. NSTimer 会 retain 你添加调用方法的对象，比如 self（最近被它坑到了）
2. NSTimer 要添加到 runloop 才会执行
3. NSTimer 并不是精确的按照指定的时间触发
4. NSTimer 添加到 runloop 后，也不一定按照你想象的那样执行

## 什么是NSTimer

NSTimer 就是一个能在从现在开始的后面的某一个时刻或者周期性的执行我们指定的方法的对象。

## NSTimer 与 Target

从上面的解释来看，NSTimer 会在未来某个时刻执行一次或者多次我们指定的方法，那么它是怎么知道该方法那个时候是否还是有效的，如果该方法不存在了，消息分发会引发异常的。

解决方法很简单，`就是把要指定给timer的“selector”的“target” retain 一份就搞定了，实际上系统也是这样做的`，不管是重复性的timer还是一次性的timer都会对它的方法的接收者进行retain，这两种timer的区别在于“**一次性的timer在完成调用以后会自动将自己invalidate，而重复的timer则将永生，直到你显示的invalidate它为止**”。

举个例子：

**JHTestTimerObject.h**
```objc
@interface JHTestTimerObject : NSObject
/*
 * 公开方法，timer响应函数，用来做测试
 */
- (void)timerAction:(NSTimer*)timer;

@end
```

**JHTestTimerObject.m**
```objc
@implementation JHTestTimerObject
- (id)init {
    self = [super init];
    if (self) {
        NSLog(@"instance %@ has been created!", self);
    }
    
    return self;
}

- (void)dealloc {
    NSLog(@"instance %@ has been dealloced!", self);
}

- (void)timerAction:(NSTimer*)timer {
    NSLog(@"Hi, Timer Action for instance %@", self);
}
@end
```

**调用**

```objc
- (void)viewDidLoad {
    [super viewDidLoad];
    
    [self testNonRepeatTimer];
    //[self testRepeatTimer];
}

- (void)testNonRepeatTimer
{
    NSLog(@"Test retatin target for non-repeat timer!");
    JHTestTimerObject *testObject = [[JHTestTimerObject alloc] init];
    [NSTimer scheduledTimerWithTimeInterval:3 target:testObject selector:@selector(timerAction:) userInfo:nil repeats:NO];
    NSLog(@"Invoke release to testObject!");
}

- (void)testRepeatTimer
{
    NSLog(@"Test retain target for repeat Timer");
    JHTestTimerObject *testObject2 = [[JHTestTimerObject alloc] init];
    [NSTimer scheduledTimerWithTimeInterval:3 target:testObject2 selector:@selector(timerAction:) userInfo:nil repeats:YES];
    NSLog(@"Invoke release to testObject2!");
}
```

## 分析

上面这个例子，创建了一个测试对象 JHTestTimerObject ，在它的init，dealloc 和 timerAction 三个方法中分别打印了信息，并且在 ViewController 中 分别测试 单次执行timer 和 重复执行timer，由于在 ARC 环境下，testObject 超出**{}**的作用域范围就会被释放，`通过这点可以确认timer是否对“target”做了 retain 操作`。

调用 **NonRepeatTimer** 打印结果：
{% asset_img Snip20161126_2.png CALayer image %}
根据打印结果，可以看出在**14:24:38**时，testObject 才执行了 dealloc 方法，系统等待 timer 执行结束后才释放 testObject 对象。这就证明了`单次执行timer也会 retain 它的“target”，直到它自己失效为止`。




调用 **RepeatTimer** 打印结果：
{% asset_img Snip20161126_3.png CALayer image %}
据观察，timer一直在周期性的调用指定的方法，而且 testObject2 一直也没被释放。

## 小结

通过上面的例子，我们可以发现在timer对它的接收者进行retain，从而保证了timer调用时的正确性，但是又引入了接收者的内存管理问题。特别是对于重复性的timer，它所引用的对象将一直存在，将会造成内存泄露。

解决办法， NSTimer 方法提供了一个销毁的方法 invalidate，我们可以用来解决这个问题。不管是一次性的还是重复性的timer，在执行完invalidate以后都会变成无效，因此对于重复性的timer我们一定要有对应的invalidate。

## 强引用引发的”血案“

相信不少人说我有调用 invalidate 销毁方法啊，但是并没有释放相关的对象，因为他们十有八九是在 dealloc 中调用的，犯了自欺欺人的错误。

举个例子：

**ViewController.m**
```objc
- (void)viewDidLoad {
    [super viewDidLoad];
    _button = [UIButton buttonWithType:UIButtonTypeSystem];
    _button.frame = CGRectMake(100, 200, 50, 50);
    [_button setTitle:@"点一下" forState:UIControlStateNormal];
    [_button addTarget:self action:@selector(pushNextVC:) forControlEvents:UIControlEventTouchUpInside];
    [self.view addSubview:_button];
}

- (void)pushNextVC:(UIButton *)button {
    JHTestTimerViewController *vc = [JHTestTimerViewController new];
    [self.navigationController pushViewController:vc animated:YES];
}
```

**JHTestTimerViewController.m**
```objc
- (void)viewDidLoad {
    [super viewDidLoad];
    
    _textLabel = [[UILabel alloc] initWithFrame:CGRectMake(100, 200, 40, 60)];
    _textLabel.text = @"这是第二个VC";
    [self.view addSubview:_textLabel];
    
    _timer = [NSTimer scheduledTimerWithTimeInterval:1.0f target:self selector:@selector(testTimer:) userInfo:nil repeats:YES];
}

- (void)dealloc {
    // 自欺欺人的写法，永远都不会执行到，除非你在外部手动invalidate这个timer
    [_timer invalidate];
}

- (void)testTimer:(NSTimer*)timer {
    NSLog(@"haha!");
}
```
按照常理 JHTestTimerViewController 会在点击导航栏的返回按钮后被销毁，然后触发 dealloc 方法，在 dealloc 方法中会销毁定时器，然而实际上并没有。

原因是 _timer 会对“target”进行 retain ，只要强引用在，JHTestTimerViewController 就不会被销毁，就不会走 dealloc 方法，也就不会销毁 _timer ，_timer 不销毁就一直强引用 JHTestTimerViewController，这就造成了内存泄漏。

## 小结2
timer 都会对它的 target 进行 retain，我们需要小心对待这个 target 的生命周期问题，尤其是重复性的timer。（NSTimer 初始化后，self 的 retainCount 加 1。 那么，我们需要在释放这个类之前，执行[timer invalidate];否则，不会执行该类的 dealloc 方法。）

## 解决方法1
既然知道了是强引用造成的原因，那么肯定有解决办法的。👇使用了一个简洁的方法来解决：

```objc
//
//  NSTimer+EZ_Helper.m
//  EZToolKit
//
//  Created by yangjun zhu on 15/5/20.
//  Copyright (c) 2015年 Cactus. All rights reserved.
//

#import "NSTimer+EZ_Helper.h"

@implementation NSTimer (EZ_Helper)
+ (NSTimer *)ez_scheduledTimerWithTimeInterval:(NSTimeInterval)inTimeInterval block:(void (^)())inBlock repeats:(BOOL)inRepeats
{
    void (^block)() = [inBlock copy];
    NSTimer * timer = [self scheduledTimerWithTimeInterval:inTimeInterval target:self selector:@selector(__executeTimerBlock:) userInfo:block repeats:inRepeats];
    return timer;
}

+ (NSTimer *)ez_timerWithTimeInterval:(NSTimeInterval)inTimeInterval block:(void (^)())inBlock repeats:(BOOL)inRepeats
{
    void (^block)() = [inBlock copy];
    NSTimer * timer = [self timerWithTimeInterval:inTimeInterval target:self selector:@selector(__executeTimerBlock:) userInfo:block repeats:inRepeats];
    return timer;
}

+ (void)__executeTimerBlock:(NSTimer *)inTimer;
{
    if([inTimer userInfo])
    {
        void (^block)() = (void (^)())[inTimer userInfo];
        block();
    }
}
@end
```
给 NSTimer 写的一个category，在 category 中把一个”伪造“的 ”self“ 赋值给了 ”target“，这个 ”self“ 类似于一个中间的代理人，作用是接受 NSTimer 的强引用，这样 NSTimer 就不会持有原始的对象，原始对象就能正常调用 dealloc 方法，在 dealloc 中的 invalidate 方法就会销毁定 timer。（通过打断点可以看出 self 实际上是 NSTimer）

## 解决方法2

在 iOS10 中，我们发现 NSTimer 多了几个API：

```objc
+ (NSTimer *)timerWithTimeInterval:(NSTimeInterval)interval repeats:
(BOOL)repeats block:(void (^)(NSTimer *timer))block
API_AVAILABLE(macosx(10.12), ios(10.0), watchos(3.0), tvos(10.0));

+ (NSTimer *)scheduledTimerWithTimeInterval:(NSTimeInterval)interval repeats:
(BOOL)repeats block:(void (^)(NSTimer *timer))block
API_AVAILABLE(macosx(10.12), ios(10.0), watchos(3.0), tvos(10.0));
``` 

这两个 API 是 iOS10 新添加的，前两个参数很好理解：**触发间隔** 和 **是否重复触发**。第三个参数是个 block ，有没有发觉和**解决方法1**非常相似，只是参数顺序不一样。

看看第三个参数的注释：

> block  The execution body of the timer; the timer itself is passed as the parameter to this block when executed to aid in avoiding cyclical references

翻译过来是：block 是 timer 需要执行的内容；当 block 被执行时，timer 自己本身会被当做参数传递给 block ，避免循环引用。

从注释就可以看出，新的API解决方式和**解决方法1**是一样的，避免持有相应的对象，这就是苹果给出了终极解决方案，但是缺点是要 iOS10 以上才能调用。


## 总结
本章主要讨论了 NSTimer 与 它的target 的关系，强引用 target 引发的问题，以及解决办法。下一章继续讨论其他坑点。

> **注意：**无论是解决方法1还是解决方法2，在 block 中，一定要使用弱引用 weakself，否则还是会造成循环引用，内存泄漏。


## 参考链接

* [NSTimer](http://justsee.iteye.com/blog/1774722)
* [iOS 中的 NSTimer](http://blog.callmewhy.com/2015/07/06/weak-timer-in-ios/)



