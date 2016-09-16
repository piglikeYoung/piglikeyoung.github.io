---
title: Objective-C中几种锁的实现(二)
date: 2016-09-15 11:35:21
tags: Objective-C 锁
category: 能工巧匠
---

## 前言

本篇博客转载自[Objective-C中不同方式实现锁(二)](http://www.tanhao.me/pieces/643.html/)

在上一篇转载中介绍Objective-C锁的几种实现，也用代码实际的演示了如何通过构建一个互斥锁来实现多线程的资源共享及线程安全，今天我们继续讨论锁的一些高级用法。

## NSRecursiveLock递归锁
平时我们在代码中使用锁的时候，最容易犯的一个错误就是造成死锁，而容易造成死锁的一种情形就是在递归或循环中，如下代码:

```objc
//主线程中
NSLock *theLock = [NSLock new];
TestObj *obj = [TestObj new];
    
// 线程1
dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
   static void(^TestMethod)(int);
   TestMethod = ^(int value) {
       [theLock lock];
       if (value > 0) {
           [obj method1];
           sleep(5);
           TestMethod(value - 1);
       }
       [theLock unlock];
   };
   TestMethod(5);
});
    
// 线程2
dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
   sleep(1);//以保证让线程2的代码后执行
   [theLock lock];
   [obj method2];
   [theLock unlock];
});
```
上述代码中，就是一种典型的死锁，因为在线程1的递归block中，锁会被多次lock，所以自己也被阻塞，由于以上的代码非常的简短，所以很容易能识别死锁，但在较为复杂的代码中，就不那么容易发现了，那么如何在递归或循环中正确的使用锁呢？

Objective-C中提供了`NSRecursiveLock`类来解决这个问题，将**NSLock**换成**NSRecursiveLock**就可以了。**NSRecursiveLock**类定义的锁可以在同一线程多次lock，并且不会造成死锁。这个递归锁会跟踪它被lock多少次。每次成功的lock都必须对称调用unlock。只有所有的锁住和解锁都平衡的时候，锁才真正被释放给其他线程获得。


## NSConditionLock条件锁
当我们在使用多线程的时候，有时一把只会lock和unlock的锁未必就能完全满足我们的使用。因为普通的锁只能关心锁与不锁，而不在乎用什么钥匙才能开锁，而我们在处理资源共享的时候，多数情况是只有满足一定条件的情况下才能打开这把锁：

```objc
//主线程中
NSConditionLock *theLock = [NSConditionLock new];
    
// 线程1
dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
   for (NSInteger i = 0; i<=3; i++) {
       [theLock lock];
       NSLog(@"thread1:%zd",i);
       sleep(2);
       [theLock unlockWithCondition:i];
   }
});
    
// 线程2
dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
   [theLock lockWhenCondition:3];
   NSLog(@"thread2");
   [theLock unlock];
});
```

在线程1中的加锁使用了lock，所以是不需要条件的，所以顺利的就锁住了，但在**unlockWithCondition:**使用了一个整型的条件，它可以开启其它线程中正在等待这把钥匙的临界值（如果只是使用**unlock**，线程2会因为没有等待到开启线程的条件而一直阻塞），而线程2则需要一把被标识为3的钥匙，所以当线程1循环到最后一次的时候，才最终打开了线程2中的阻塞。但即便如此，**NSConditionLock**也跟其它的锁一样，是需要lock与unlock对应的，只是`lock`，`lockWhenCondition:`与`unlock`，`unlockWithCondition:`是可以随意组合的，当然这是与你的需求相关的。

## NSDistributedLock分布式锁（用于Mac的多进程开发）
以上所有的锁都是在解决多线程之间的冲突，但是如果遇上多个进程或者多个程序之间需要构建互斥的情景该怎么办呢？这个时候我们就需要使用到`NSDistributedLock`了，从它的类名就知道这是一个分布式的Lock，**NSDistributedLock的实现是通过文件系统的，所以使用它才可以有效的实现不同进程之间的互斥**，但NSDistributedLock并非继承于NSLock，它没有lock方法，它只实现了**tryLock**，**unlock**，**breakLock**，所以如果需要lock的话，你就必须自己实现一个tryLock的轮询，下面通过代码简单的演示一下吧：

程序A：
```objc
dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
   lock = [[NSDistributedLock alloc] initWithPath:@"/Users/mac/Desktop/earning__"];
   [lock breakLock];
   [lock tryLock];
   sleep(10);
   [lock unlock];
   NSLog(@"appA: OK");
});
```

程序B：

```objc
dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
   lock = [[NSDistributedLock alloc] initWithPath:@"/Users/mac/Desktop/earning__"];

   while (![lock tryLock]) {
       NSLog(@"appB: waiting");
       sleep(1);
   }
   [lock unlock];
   NSLog(@"appB: OK");
});
```

先运行程序A，然后立即运行程序B，根据打印你可以清楚的发现，当程序A刚运行的时候，程序B一直处于等待中，当大概10秒过后，程序B便打印出了appB:OK的输出，以上便实现了两个不同程序之间的互斥。**/Users/mac/Desktop/earning__**是一个文件或文件夹的地址，如果该文件或文件夹不存在，那么在tryLock返回YES时，会自动创建该文件/文件夹。在结束的时候该文件/文件夹会被清除，所以在选择的该路径的时候，应该选择一个不存在的路径，以防止误删了文件。


