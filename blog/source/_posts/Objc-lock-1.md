---
title: Objective-C中几种锁的实现(一)
date: 2016-09-15 09:57:18
tags: Objective-C 锁
category: 能工巧匠
---

## 前言

本篇博客转载自[Objective-C中不同方式实现锁(一)](http://www.tanhao.me/pieces/616.html/)

任何的语言的高级部分都有会有多线程，使用多线程都会涉及“锁”。那么在Objective-C中到底有多少种锁？有几种锁的实现方法？

我们先构建一个测试类TestObj，假设它是我们的一个共享资源，有两个互斥的方法 **method1** 和 **method2**，代码如下：

```objc
@implementation TestObj

- (void)method1
{
    NSLog(@"%@",NSStringFromSelector(_cmd));
}

- (void)method2
{
    NSLog(@"%@",NSStringFromSelector(_cmd));
}

@end
```

## 使用NSLock实现的锁

```objc
TestObj *obj = [TestObj new];
NSLock *objLock = [NSLock new];
// 线程1
dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
   [objLock lock];
   [obj method1];
   sleep(10);
   [objLock unlock];
});
    
// 线程2
dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
   sleep(1);//以保证让线程2的代码后执行
   [objLock lock];
   [obj method2];
   [objLock unlock];
});
```

运行的打印结果：

```bash
2016-09-15 09:50:39.718 test[1388:39695] method1
2016-09-15 09:50:49.722 test[1388:39688] method2
```
从打印结果你可以看到线程1锁住后，线程2会一直等待线程1把锁unlock后，才会执行method2方法。

`NSLock`是Cocoa提供给我们最基本的锁对象，这也是我们经常使用的，除了lock和unlock方法外，NSLock还提供了**tryLock**和**lockBeforeDate:**两个方法。
* `tryLock`方法会尝试加锁，如果锁不可用（已经被锁住），并不会阻塞线程，并返回NO，如果锁可以用，则自动调用lock方法锁住对象，阻塞线程，并返回YES。
* `lockBeforeDate:`方法会在所指定Date之前尝试加锁，如果在指定时间之前都不能加锁，则返回NO。


## 使用synchronized关键字构建的锁

Objective-C中还提供了`@synchronized`指令快速实现锁：

```objc
TestObj *obj = [TestObj new];
    
// 线程1
dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
   @synchronized (obj) {
       [obj method1];
       sleep(5);
   }
});
    
// 线程2
dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
   sleep(1);//以保证让线程2的代码后执行
   @synchronized (obj) {
       [obj method2];
   }
});
```
代码中**@synchronized**指令使用`obj`为该锁的唯一标识，只有当标识相同时，才为满足互斥，如果线程2中把`@synchronized (obj)`改为`@synchronized (other)`则线程2就不会被阻塞。

**@synchronized**实现锁的优点就是我们不需要再代码中现实的创建NSLock对象，便可以实现锁的机制，但作为一种预防措施，**@synchronized**块会隐式的添加一个异常处理例程来保护代码，该处理例程会在异常抛出的时候自动的释放互斥锁。所以如果不想让隐式的异常处理例程带来额外的开销，你可以考虑使用NSLock对象。

## 使用C语言的pthread_mutex_t实现的锁

```objc
TestObj *obj = [TestObj new];
__block pthread_mutex_t mutex;
pthread_mutex_init(&mutex, NULL);
    
// 线程1
dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
   pthread_mutex_lock(&mutex);
   [obj method1];
   sleep(5);
   pthread_mutex_unlock(&mutex);
});
    
// 线程2
dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
   sleep(1);//以保证让线程2的代码后执行
   pthread_mutex_lock(&mutex);
   [obj method2];
   pthread_mutex_unlock(&mutex);
});
```

使用pthread_mutex_t要记得`#import "pthread.h"`

## 使用GCD来实现的”锁”
以上代码构建多线程我们就已经用到了GCD的dispatch_async方法，其实在GCD中也已经提供了一种**信号机制**，使用它我们也可以来构建一把”锁”(从本质意义上讲，信号量与锁是有区别，可以Google[信号量与互斥锁的区别](https://www.google.com/search?q=%E4%BF%A1%E5%8F%B7%E9%87%8F%E4%B8%8E%E4%BA%92%E6%96%A5%E9%94%81%E7%9A%84%E5%8C%BA%E5%88%AB&oq=%E4%BF%A1%E5%8F%B7%E9%87%8F%E4%B8%8E%E4%BA%92%E6%96%A5%E9%94%81%E7%9A%84%E5%8C%BA%E5%88%AB&gs_l=serp.3...7725.7875.0.8139.2.2.0.0.0.0.233.233.2-1.1.0....0...1c.1.64.serp..1.0.0.bAEpnXn44rg)):

```objc
TestObj *obj = [TestObj new];
dispatch_semaphore_t semaphore = dispatch_semaphore_create(1);
    
// 线程1
dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
   dispatch_semaphore_wait(semaphore, DISPATCH_TIME_FOREVER);
   [obj method1];
   sleep(5);
   dispatch_semaphore_signal(semaphore);
});
    
// 线程2
dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
   sleep(1);//以保证让线程2的代码后执行
   dispatch_semaphore_wait(semaphore, DISPATCH_TIME_FOREVER);
   [obj method2];
   dispatch_semaphore_signal(semaphore);
});
```
代码的效果和前面的例子是一模一样的。但是信号锁的效率比NSLock高得多，YYKit也是采用这种方式加锁的，之前我也写过一篇[《GCD信号锁》](http://piglikeyoung.com/2016/09/11/GCD-signal-lock/)比较GCD信号锁和NSLock的效率。



