---
title: 使用GCD（二）
date: 2016-10-01 18:32:11
tags: GCD
category: 能工巧匠
---

## 前言
前一篇文章（[链接](http://piglikeyoung.com/2016/09/25/Using-Grand-Central-Dispatch-1/)）已经简单的介绍GCD的简单使用，这篇文章继续使用GCD。

## dispatch_async
GCD中有2个异步的API：

```objc
void dispatch_async(dispatch_queue_t queue, dispatch_block_t block);
void dispatch_async_f(dispatch_queue_t queue, void *context, dispatch_function_t work);
```

这两个API都是将一个任务提交到queue中，提交之后立刻返回，不等待任务执行完成。系统会对queue做retain操作，任务执行完成后，queue才被release。两者的区别在于`dispatch_async`接受block作为参数，`dispatch_async_f`接受函数。

`dispatch_async_f`中，**context**作为第一个参数传给**work**函数。如果**work**不需要参数，**context**可以传入**NULL**。**work**参数不能传入**NULL**，否则会发生不可预料的事。

> 使用GCD的时候，block会被copy，block会在执行完之后release，而且block是被系统持有的，所以不用担心循环引用的问题，**block里面的self不需要weak**。

`async`是异步的意思，至于怎么异步，看下面的例子：
```objc
- (void)dispatchAsync {
    NSLog(@"1");
    dispatch_async(dispatch_get_main_queue(), ^{
        // do Something
        NSLog(@"2");
    });
    NSLog(@"3");// 1，3，2
}
```
因为block是异步的，打印不是按照1，2，3的顺序打印的，简单点说，就是block里面的任务添加到队列后，GCD立刻返回，不阻塞线程，不需要等待任务执行完成。

就像上面的例子，先打印 ”1“ ，block添加主队列里，添加完成立刻返回，然后打印 ”3“，等主队列里面前面的任务执行完成再打印 ”2“。


## dispatch_sync

和异步一样，GCD同步也有两个API：

```objc
void dispatch_sync(dispatch_queue_t queue, dispatch_block_t block);
void dispatch_sync_f(dispatch_queue_t queue, void *context, dispatch_function_t work);
```
参数内容和`dispatch_async`一样，不同的是任务加入queue之后不会立刻返回，阻塞线程，等待任务执行完成后再返回。


```objc
- (void)dispatchSync {
    NSLog(@"1");
    dispatch_sync(dispatch_get_global_queue(QOS_CLASS_DEFAULT, 0), ^{
        // do Something
        NSLog(@"2");
    });
    NSLog(@"3");// 1，2，3
}
```

从打印可以看出，打印是按照顺序来的
1. 先打印 ”1“ 
2. 然后dispatch_sync会将block提交到global queue中，等待block的执行
3. 全局队列中block前面的任务执行完成后，block执行
4. block执行完成后，dispatch_sync返回
5. dispatch_sync之后的代码执行，打印 ”3“

从示例代码中可以看出我在全局队列做同步操作的，没有在主队列里面做，是因为dispatch_sync需要等待block被执行，非常容易发生死锁，特别是在串行队列里面，而且主队列就是串行队列。


```objc
- (void)dispatchSyncDeadLock {
    NSLog(@"1");
    dispatch_sync(dispatch_get_main_queue(), ^{ // Dead Lock
        // do Something
        NSLog(@"2");
    });
    NSLog(@"3");
}
```

分析一下死锁怎么发生的：
1. 任务是放到主队列（串行队列）里面，串行队列任务是一个一个执行的
2. dispatch_sync需要等待block执行完成发返回，block需要等待前面的任务执行完成，也就是dispatch_sync执行完成。两者相互等待，永远不会执行完成，造成了死锁。

从中看出死锁是：`使用了dispatch_sync将任务加入串行队列`

如果是并行队列，则不会造成死锁。

## dispatch_after
在开发中经常需要到延迟几秒再执行的代码的情况，GCD也有这样的API:

```objc
void dispatch_after(dispatch_time_t when, dispatch_queue_t queue, dispatch_block_t block);
void dispatch_after_f(dispatch_time_t when, dispatch_queue_t queue, void *context, dispatch_function_t work);
```
when表示时间，如果传入 `DISPATCH_TIME_NOW` 会等同于 `dispatch_async` ，另外不允许传入 `DISPATCH_TIME_FOREVER` ，这会永远阻塞线程。

```objc
- (void)dispatchAfter {
    NSLog(@"1");
    dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(3 * NSEC_PER_SEC)), dispatch_get_main_queue(), ^{
        NSLog(@"2");
    });
    NSLog(@"3");
}
```
需要注意的是，dispatch_after 函数并不是在指定时间后执行处理，而只是在指定时间追加处理到 Dispatch Queue。此源代码与在3秒后用 dispatch_async 函数追加 Block 到 Main Dispatch Queue 相同。

因为 Main Dispatch Queue 在主线程的 RunLoop 中执行，所以在比如每隔1/60秒执行的 RunLoop 中，Block 最快在3秒后执行，最慢在3秒+1/60秒后执行，并且在 Main Dispatch Queue 有大量处理追加或者主线程的处理本身有延迟时，这个时间会更长。

虽然在有严格时间的要求下使用时会出现问题，但在想大致延迟处理时，该函数是非常有效的。

例子中 `dispatch_time` 参数原型是这样的

```objc
dispatch_time_t dispatch_time ( dispatch_time_t when, int64_t delta );
```
第一个参数用 `DISPATCH_TIME_NOW`表示当前。第二个参数delta表示纳秒，1秒=1000000000纳秒，系统用一些宏来简化

```objc
 #define NSEC_PER_SEC 1000000000ull //每秒有多少纳秒
 #define USEC_PER_SEC 1000000ull    //每毫秒有多少纳秒
 #define NSEC_PER_USEC 1000ull      //每秒有多少毫秒
```

这样1秒可以这样写：

```objc
dispatch_time(DISPATCH_TIME_NOW, 1 * NSEC_PER_SEC);
dispatch_time(DISPATCH_TIME_NOW, 1000 * USEC_PER_SEC);
dispatch_time(DISPATCH_TIME_NOW, USEC_PER_SEC * NSEC_PER_USEC);
```

## dispatch_once
GCD中还有保证在应用程序中只执行一次指定处理的API。

```objc
- (void)dispatchOne {
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        // do Something
    });
}
```
通常通过这种方式生成单例对象。


