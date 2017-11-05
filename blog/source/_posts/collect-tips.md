---
title: iOS 实际开发中使用到的小技巧
date: 2017-07-01 16:17:11
tags: Debug
category: Tips
---

## 前言

以下是我收集的关于 iOS 开发提高效率的小技巧。

## 一、如何快速的查看一段代码的执行时间


```objc
#define TICK   NSDate *startTime = [NSDate date]
#define TOCK   NSLog(@"Time: %f", -[startTime timeIntervalSinceNow])
```

在想要查看执行时间的代码的地方进行这么处理

```objc
TICK
//do your work here
TOC
```

## 二、如何快速查看一个函数的调用次数，且不添加一句代码

在 if 或者 for 打个断点，然后编辑断点。

{% asset_img calculate-the-number-of-calls.jpg calculate-the-number-of-calls %}

这种方法适合于一个if方法，一个for循环，而且不会中断程序，且不需要加一句代码。但是一定要记得选中下面的 **automatically continue after evaluting actions**;

## 三、快速手机截屏

在iOS开发中我们在和产品和设计沟通的时候我们经常需要截取手机的屏幕或者模拟器上的屏幕，我们用手机可能会使用 Home 键 + 开机键，然后再通过 iPhoto 或者在手机用 qq 传过去，但是我教大家一个方法直接使用快捷键截取手机上的图到电脑桌面上。

{% asset_img take-screenshot.jpg take-screenshot %}

在 Xcode的 debug菜单中找到 **viewDebugging**,即使当前程序没有运行，也可以直接截取手机上的图片直接到桌面。（哈哈哈这样再不需要TM的按TM的手机上的按键再用 iPhoto拷贝到桌面了）。年轻人你以为这样就完了吗！？你还是太稚嫩啊，谁TM的想找到这个debug菜单再找到下面的一堆东西，当然要改成快捷键了，如何做看下图。

{% asset_img take-screenshot-fast.jpg take-screenshot-fast %}

看到这个血淋漓的红色的箭头了嘛，你首先找到 debug 的快捷键菜单项，在把它改成 ?+?这个，这时候有冲突了怎么办？你不知道有没有影响到其他快捷键怎么办，小傻瓜，改呗！把以前的这个功能去掉?+?（ps:以前的就是 show complete list 如同点击一个?一个效果，那你还要它做嘛啊?），为什么改成这个份听哥的，你改成这个绝壁会用着特别爽。（好了以后要给产品还是设计发图分分钟的事情了~~）

## 四、iOS调试技巧只显示图片的对齐尺寸和 frame

我记得以前一个说显示对齐尺寸的，他是这么做的：

“在应项目的Edit Scheme中设置一个启动参数 **UIViewShowAlignmentRects** 并将参数值设置为YES，可以让程序在运行时显示视图的对齐矩阵（alignment rectangle）。”

这种方式太麻烦了，有个更简单的方式，上个截图 **viewDebuging** 中有 **showViewFrame** 和 **ShowAlignmentRects**，当然点击这些菜单就会出现我这些效果了。

{% asset_img debug-frame.gif debug-frame %}

## 五、计算UIImageView在视图中位置的函数

`AVMakeRectWithAspectRatioInsideRect` 通过这个函数，我们可以计算一个图片放在另一个 view 按照一定的比例居中显示。比如，将一个 UIImageView 居中显示到一个 View 中，那么UIImageView 居中后，X 和 Y 的值是多少，通过该函数能够非常方便的计算出来。

{% asset_img AVMakeRectWithAspectRatioInsideRect.jpeg AVMakeRectWithAspectRatioInsideRect %}

```objc
// 将要显示的图片size放到第一个参数，第二个参数是装载UIImageView的View容器frame
// 函数返回的frame就是图片居中显示的frame
CGRect imageViewAspectRect = AVMakeRectWithAspectRatioInsideRect(CGSizeMake(image.size.width, image.size.height), _videoPicBackground.frame);
// 赋值
mageView.frame = imageViewAspectRect;
```

函数在AV库中，记得引入头文件

```objc
#import <AVFoundation/AVFoundation.h>
```


## 参考链接
[iOS 开发的一些小技巧篇一](http://www.jianshu.com/p/221507eb8590)
[iOS 开发的一些小技巧篇二](http://www.jianshu.com/p/80ebbb1950ba)
[iOS 开发的一些小技巧篇三](http://www.jianshu.com/p/827090aa933b)






