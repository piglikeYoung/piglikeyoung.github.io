---
title: NSInteger与NSUInteger
date: 2017-02-26 15:33:41
tags: 基础数据类型
category: 拿来主义
---

## 前言

在 iOS 开发中我想最经常使用的基本数据类型非 `NSInteger` 与 `NSUInteger` 莫属，我们经常使用它们来代表有符号与无符号的整数。严格上说，NSInteger 并不能算是“Objective-C 的整数”，**因为 NSInteger 其实是C语言的类型，而不是 Objective-C 对象，用来表示数字的 Objective-C 对象是 `NSNumber` 以及 `NSDecimalNumber`**。

## 定义

在 Objective-C 头文件中看到：

```objc
#if __LP64__ || (TARGET_OS_EMBEDDED && !TARGET_OS_IPHONE) || TARGET_OS_WIN32 || NS_BUILD_32_LIKE_64
typedef long NSInteger;
typedef unsigned long NSUInteger;
#else
typedef int NSInteger;
typedef unsigned int NSUInteger;
#endif
```
在64位系统下，NSInteger 是 long 类型，也就是 64位系统的整数，在32位系统下，是 int。同时，NSUInteger 在64位与32位系统下分别是 unsigned long 与 unsigned int。

总的来说 NSInteger 与 NSUInteger 就是C的整数。

## 使用

我们写代码通常是运行在64位或者32位系统上，比如 iPhone 5和之前机型使用 armv7 ，CPU是32位的系统，iPhone 5s 之后则是在 arm64 上。**因此，当我们使用整数的时候，即使我们也可以直接使用 int 或 long，但我们会尽量使用 NSInteger 与 NSUInteger，让 compiler 帮我们决定使用哪个整数。**

另外，使用浮点数的时候，也尽量使用 CGFloat。CGFloat 一样会在不同环境下，当成32位或者64位的浮点型。

## 参考链接
* [KKBOX iOS/Mac OS X 基本開發教材](https://www.gitbook.com/book/zonble/kkbox-ios-dev/details)


