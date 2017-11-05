---
title: UIDebuggingInformationOverlay
date: 2017-05-27 20:51:37
tags: Debug
category: 能工巧匠
---

## 前言

今天网上突然爆出了一篇文章[《UIDebuggingInformationOverlay》](http://ryanipete.com/blog/ios/swift/objective-c/uidebugginginformationoverlay/)，介绍了 iOS 界面调试工具。

以下是我对文章的理解+实践！

之前我们经常通过 [Reveal](https://revealapp.com/) 或者 [FLEX](https://github.com/Flipboard/FLEX) 来调试界面，现在爆出了苹果的私有方法 `UIDebuggingInformationOverlay` 可以用来打开调试界面：

{% asset_img uidebugginginformationoverlay_0.png uidebugginginformationoverlay_0 %}

> 注意：
> 添加的方法是私有方法，苹果未对外公开，有可能会被拒绝上架 App Store。

## 怎么使用

在某个 ViewController 中加入以下代码：

```swift
let overlayClass = NSClassFromString("UIDebuggingInformationOverlay") as? UIWindow.Type
_ = overlayClass?.perform(NSSelectorFromString("prepareDebuggingOverlay"))
let overlay = overlayClass?.perform(NSSelectorFromString("overlay")).takeUnretainedValue() as? UIWindow
_ = overlay?.perform(NSSelectorFromString("toggleVisibility"))
```

APP启动后，加载代码完毕就会调用调试框。

Objective-C代码：

```objc
dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(3 * NSEC_PER_SEC)), dispatch_get_main_queue(), ^{
	#ifdef DEBUG
	#pragma clang diagnostic push
	#pragma clang diagnostic ignored "-Warc-performSelector-leaks"
	        id debugClass = NSClassFromString(@"UIDebuggingInformationOverlay");
	        [debugClass performSelector:NSSelectorFromString(@"prepareDebuggingOverlay")];
	        
	        id debugOverlayInstance = [debugClass performSelector:NSSelectorFromString(@"overlay")];
	        [debugOverlayInstance performSelector:NSSelectorFromString(@"toggleVisibility")];
	#pragma clang diagnostic pop
	#endif
});
```

### 更简单的方式

只需要加入：

```swift
let overlayClass = NSClassFromString("UIDebuggingInformationOverlay") as? UIWindow.Type
_ = overlayClass?.perform(NSSelectorFromString("prepareDebuggingOverlay"))
```

运行程序后，`两根手指`同时点击状态栏可以调起之前那个调试界面。

## 能做什么

### View Hierarchy 

查看APP的层级关系

这个选项可以查看界面的层级树，点击后面的感叹号就可以进入详情（点击Cell无反应），进入详情可以看到 View 的 Class、Frame、Opacity、Bounds 等属性

{% asset_img uidebugginginformationoverlay_viewhierarchy.png viewhierarchy %}
{% asset_img uidebugginginformationoverlay_viewhierarchyinfo.png viewhierarchyinfo %}

### VC Hierarchy 

查看当前 Controller 的属性或者它的子类的属性，你还可以看看 Controller 是通过 modal 的方式弹出来或者是 presented 出来。

{% asset_img uidebugginginformationoverlay_VCHierarchy.jpeg VCHierarchy %}

### Ivar Explorer

查看 UIApplication 的内部属性

{% asset_img uidebugginginformationoverlay_ivarexplorer.jpeg ivarexplorer %}

### Measure

点击进入后，发现有三个选项：None、Vertical 和 Horizontal 。

你可以通过选择选项，在屏幕上拖拽，显示某个控件的高度或者宽度。

{% asset_img uidebugginginformationoverlay_measure2.jpeg measure2 %}

### Spec Compare

前后界面对比，从相册里读取图片和当前界面对比。（需要获取相册访问权限NSPhotoLibraryUsageDescription）

1. 点击 Add 
2. 从相册选择一个界面截图 ，点击刚添加的截图
3. 手指在屏幕（调试窗口外部）上下滑动，可动态改变截图的透明度来对比截图和当前界面的差异 
4. 双击退出

{% asset_img uidebugginginformationoverlay_speccompareoverlay.png speccompareoverlay %}

{% asset_img uidebugginginformationoverlay_speccomparepicker.png speccomparepicker %}

## IPAPatch

上面说的都只是你有源码可以添加代码的情况下打开调试器，假如是别的APP呢？

这里我推荐一个开源的工具 [IPAPatch](https://github.com/Naituw/IPAPatch) 可以免越狱调试、修改第三方App，它的使用可以参考[链接](http://weibo.com/ttarticle/p/show?id=2309404086977153611942)

通过 **IPAPatch** + **UIDebuggingInformationOverlay** 就可以查看第三方APP的层级结构了，使用 **IPAPatch** 有个要求就是必须是解密过的的 IPA，可以在 [iphonecake](https://www.iphonecake.com) 下载（注意有好多假链接）也可以通过越狱的手机导出 IPA，按照参考链接的教程，就可以查看第三方APP的层级结构了（其实前面有些截图就是查看Snapchat的）。

## 参考链接

* [UIDebuggingInformationOverlay](http://ryanipete.com/blog/ios/swift/objective-c/uidebugginginformationoverlay/)
* [IPAPatch: 免越狱调试、修改第三方App](http://weibo.com/ttarticle/p/show?id=2309404086977153611942)








