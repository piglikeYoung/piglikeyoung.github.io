---
title: 遍历中修改可变数组
date: 2016-09-04 16:25:26
tags: 拿来主义
---

## 前言
浏览微博时，看到一篇关于`遍历中修改可变数组`Tips，觉得很实用，在此记录下来。

原出处：[南峰子_老驴#iOS知识小集](http://huati.weibo.com/k/iOS%E7%9F%A5%E8%AF%86%E5%B0%8F%E9%9B%86?from=501)

## 问题
相信每个开发都会遇到在遍历中修改可变数组的情况，稍不小心就会数组越界，**我们不建议在遍历可变数组时修改数组**，包括添加，删除，替换元素。

不过，如果真的要在遍历的同时删除元素，可以采用`倒序遍历`，不会引起崩溃。

```objc
NSMutableArray *array = [NSMutableArray arrayWithObjects:@"a", @"b", @"c", @"d", @"e", nil];
[array enumerateObjectsWithOptions:NSEnumerationReverse usingBlock:^(id  _Nonnull obj, NSUInteger idx, BOOL * _Nonnull stop) {
   if ([obj isEqualToString:@"c"] || [obj isEqualToString:@"e"]) {
       [array removeObject:obj];
   }
}];
    
NSLog(@"%@", array);// a, b, d
```


```objc
NSMutableArray *array = [NSMutableArray arrayWithObjects:@"a", @"b", @"c", @"d", @"e", nil];
for (NSString *obj in array.reverseObjectEnumerator) {
if ([obj isEqualToString:@"a"] || [obj isEqualToString:@"e"] || [obj isEqualToString:@"c"]) {
       [array removeObject:obj];
   }
}
NSLog(@"%@", array);// b, d
```

## 分析原因
无论是`for...in...` 还是 `enumerateObjectsWithOptions`    都可以认为是先获取了数组的大小 **int count = array.count**，然后**for(int i=0;i<count;i++)**

假设count是10，这时候你在循环时删除了一个元素，数组剩余9个，你能获取到的index最多到8，但是你的循环还是执行到i到9为止，之后就越界了。

如果是倒序 **for(int i=count-1;i>=0;i--)**，这么遍历就算把数组元素全部删除也不会有问题。
你调用枚举器遍历数组时count只会获取一次，且中途不能修改。

**如果非要顺序删除也有方法，你删完立刻break。**


