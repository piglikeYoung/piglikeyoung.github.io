---
title: Objective-C 链式语法的实现
date: 2016-11-5 13:37:40
tags: Block
category: 能工巧匠
---

## 前言
每次朋友看到我写OC代码都说方法名怎么这么长，简直可以写文章了！对应OC的语法，喜欢的人说它接近自然语言，可读性强，不用看文档或者注释也能知道方法的大概使用，不喜欢的人会觉得太啰嗦了，太麻烦了。

其实 OC 也可以实现类似别的语言那样的`点链式语法`。[Masonry](https://github.com/SnapKit/Masonry)这个鼎鼎大名的第三方自动布局库就是OC`点链式语法`开创者。

## Block
本文重点不在介绍 **Block**，想要了解的同学可以戳底部参考文章。
我们先了解链式语法：

```objc
[view1 mas_makeConstraints:^(MASConstraintMaker *make) {
    make.top.equalTo(superview.mas_top).with.offset(padding.top);
    make.left.equalTo(superview.mas_left).with.offset(padding.left);
    make.bottom.equalTo(superview.mas_bottom).with.offset(-padding.bottom);
    make.right.equalTo(superview.mas_right).with.offset(-padding.right);
}];
```

从 **Masonry** 的示例代码中可以看出链式语法包含：`点语法`，`小括号调用`，`连续调用`。

- 点语法: 在别的语言中常见的调用方法的方式，OC中常用的是属性访问，比如访问frame，self.frame
- 小括号调用: OC中调用方法一般使用中括号`[]`来实现，但是 Block 可以使用小括号`()`来调用
- 连续调用: Block 可以看看成是有自动变量的匿名函数，匿名函数可以有返回值的。要实现连续调用，可以每次调用方法后，返回当前实例本身，即 self.method1() 返回 self ，这样又可以调用 self.method2()。

> 这样我们可以对外配置一些只读Block属性，返回 Block 本身，实现 Block 属性的 getter 方法。

## 链式调用计算器
那么我们实践一下：
创建个链式调用计算器

`JHCalculator.h`
```objc

@interface JHCalculator : NSObject

/**
 计算结果
 */
@property (nonatomic, readonly, assign) NSInteger resultValue;

// Block NSInteger 类型参数， 返回当前 JHCalculator 类型

/**
 加
 */
@property (readonly, nonatomic, copy) JHCalculator * (^add)(NSInteger num);

/**
 减
 */
@property (readonly, nonatomic, copy) JHCalculator * (^minus)(NSInteger num);

/**
 乘
 */
@property (readonly, nonatomic, copy) JHCalculator * (^multiply)(NSInteger num);

/**
 除
 */
@property (readonly, nonatomic, copy) JHCalculator * (^divide)(NSInteger num);

```

`JHCalculator.m`

```objc
- (JHCalculator * (^)(NSInteger num)) add {
    return ^(NSInteger num) {
        _resultValue += num;
        return self;
    };
}

- (JHCalculator * (^)(NSInteger num)) minus {
    return ^(NSInteger num) {
        _resultValue -= num;
        return self;
    };
}

- (JHCalculator * (^)(NSInteger num)) multiply {
    return ^id(NSInteger num) {
        _resultValue *= num;
        return self;
    };
}

- (JHCalculator * (^)(NSInteger num)) divide {
    return ^(NSInteger num) {
        NSAssert(num != 0, @"除数不能为零！");
        _resultValue /= num;
        return self;
    };
}
```

使用：

```objc
JHCalculator *calculator = [JHCalculator new];
calculator.add(10).minus(5).multiply(3).divide(5);
    
NSLog(@"%zd", calculator.resultValue);// print 3
```

### 分析
上面每次调用加减乘除都会访问对应的属性，比如调用 add 就是访问 add 的getter方法，即[calculator add] 方法，它会返回一个Block。

```objc
return ^(NSInteger num) {
   _resultValue += num;
   return self;
};
```
block 返回 JHCalculator 类型，所以直接返回 self 。

详细流程是这样：
1. calculator.add 获取一个 Block
2. calculator.add(10) 执行了 Block，返回了self，即当前 JHCalculator 实例
3. 然后通过返回的 self 可以继续访问 JHCalculator 中的其他属性，一步步点下去

## 更简洁的实现
上述方法是[《objective-c 一个链式加法计算器实现》](http://www.cnblogs.com/longling2344/p/5126021.html)这篇文章中的实现方式，每次都需要增加一个属性，并实现该属性的getter方法，但是在 Masonry 中有更简洁的实现方式。

其实在OC中点语法的本质是：`将 classInstance.XXX 转换成 [classInstance XXX] 来实现方法调用的`。

* 点语法其实只是表面现象，**classInstance.XXX** 在等号右边是**getter**方法， 会编译成**[classInstance XXX]** ，**classInstance.XXX** 在等号左边是 **setter** 方法，会编译成 **[classInstance setXXX:value]**。
* 既然在编译时 **classInstance.XXX** 会转换成 **[classInstance XXX]**，那就和OC的方法调用是一样的，那么我们只需要创建一个 XXX 的方法，也能顺利的调用。

### 优化后
`JHCalculator2.h`

```objc
@interface JHCalculator2 : NSObject

/**
 计算结果
 */
@property (nonatomic, readonly, assign) NSInteger resultValue;

- (JHCalculator2 * (^)(NSInteger num)) add;

- (JHCalculator2 * (^)(NSInteger num)) minus;

- (JHCalculator2 * (^)(NSInteger num)) multiply;

- (JHCalculator2 * (^)(NSInteger num)) divide;

@end
```
.m文件内容不变，.h文件不需要写属性了，直接通过方法调用，比之前简洁了很多。

## 总结
OC链式语法总的来说是通过 **Block特性** 与 **OC点语法本质** 结合共同实现的，不过在Xcode上有个小问题，就是代码不会补全**()**，相比方便简洁的方法调用来说，这个小问题可以忽略了。

[Demo地址](https://github.com/piglikeYoung/BlockCalculator)

## 参考资料
* [黑幕背后的__block修饰符](http://chun.tips/blog/2014/11/13/hei-mu-bei-hou-de-blockxiu-shi-fu/)
* [Block技巧与底层解析](http://www.jianshu.com/p/51d04b7639f1)
* [神奇的Block](http://www.jianshu.com/p/4b1cfd7e6361)
* [用Block实现链式编程](http://www.tuicool.com/articles/qQJfYn3)
* [objective-c 一个链式加法计算器实现](http://www.cnblogs.com/longling2344/p/5126021.html)

