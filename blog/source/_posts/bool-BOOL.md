---
title: bool与BOOL
date: 2017-02-12 14:50:13
tags: 基础数据类型
category: 拿来主义
---

## 前言

过完年后，总有有些懒惰，加上公司工作任务繁重，所以好久没更新自己的学习了。今天就来了解下Objective-C中的bool和BOOL。

## 历史

Objective-C 是在 C语言的基础上，增加了面向对象与动态语言的特性的语言，很多基本数据类型都是直接来自C语言。

C语言在刚开始时，并没有布尔类型，所以Objective-C语言在发展的过程中，定义了自己的`BOOL`，但是在C99标准中，C语言又有了自己的布尔类型`bool`，并且**Objective-C又可以和C++混编成Objective-C++，C++里头也有bool**。

## 差别

那么，Objective-C 的 BOOL，和 C99 以及 C++ 的bool的差别在哪里？
先看看 Objective-C 中 iOS 10 SDK的头文件定义：


```objc
#include <stdbool.h>

....

#define OBJC_BOOL_DEFINED

/// Type to represent a boolean value.
#if (TARGET_OS_IPHONE && __LP64__)  ||  TARGET_OS_WATCH
#define OBJC_BOOL_IS_BOOL 1
typedef bool BOOL;
#else
#define OBJC_BOOL_IS_CHAR 1
typedef signed char BOOL; 
// BOOL is explicitly signed so @encode(BOOL) == "c" rather than "C" 
// even if -funsigned-char is used.
#endif

#if __has_feature(objc_bool)
#define YES __objc_yes
#define NO  __objc_no
#else
#define YES ((BOOL)1)
#define NO  ((BOOL)0)
#endif
```


所以说，在 iOS 的64位系统或者在Apple Watch 上，Objective-C 的 BOOL 直接等于定义在  stdbool.h 里头的 bool，其实是 int，假如使用了C++，那么stdbool.h里面的定义就变成了C++的 bool。但是，如果是在 MacOS 或者 32位系统的 iOS ，BOOL会被定义成一个 char，而 BOOL 与 bool，就分别是一个 byte 或者是 四个 bytes 的差别。

```objc
NSLog(@"YES: %zd", sizeof(YES));
NSLog(@"true: %zd", sizeof(true));
```

{% asset_img Snip20170305_1.png YES-true %}

总的来说，在64位系统或者 Apple Watch上，BOOL 与 bool 并没有差别，但我们通常不能假设我们的代码只会在这个环境下运行，虽然其他环境下，使用 BOOL 或者是 bool 通常也没什么影响，但是既然某个 API 明确要求传入 BOOL，那么就传入 YES 或 NO，好像也没有非要传入 true 或 false 的理由。

在 Xcode4.4 之后，增加了 literals 的写法，很简单的把 BOOL 转成 NSNumber，写成 `@(YES)` 或 `@(NO)`  ，另外在 Core Foundation 里面也定义了 `kCFBooleanTrue` 與 `kCFBooleanFalse` 也具有同样的功能。

## 参考链接
* [KKBOX iOS/Mac OS X 基本開發教材](https://www.gitbook.com/book/zonble/kkbox-ios-dev/details)


