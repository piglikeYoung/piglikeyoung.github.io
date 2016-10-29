---
title: iOS中使用正则表达式
date: 2016-10-04 12:33:39
tags: 正则表达式
category: 能工巧匠
---

## 前言
在表单验证中，任何语言都会经常使用到正则表达式，来判断用户输入是不是合法，如果不合法提示用户输入错误，不提交到服务器，过滤输入的重点就在于**正则表达式**。这篇文章我们了解下iOS中正则表达式的使用。

## NSString实例方法
NSString实例里面提供了`rangeOfString:options:`方法来匹配正则表达式：

```objc
NSString *phoneNum = @"18123456789";
NSRange range = [phoneNum rangeOfString:@"^1[8]\\d{9}$" options:NSRegularExpressionSearch];
if (range.location != NSNotFound) {
   NSLog(@"%@", [phoneNum substringWithRange:range]);
}
```
`rangeOfString:options:`会返回第一个匹配结果的位置，如果匹配不到返回**NSNotFound**，也就是**NSIntegerMax**最大值。options中设定**NSRegularExpressionSearch**就是表示利用正则表达式匹配。

这种判断方法适合需要捕获用户输入内容的特定内容时使用，通过**NSRange**可以捕获内容，甚至可以替换内容。

## 谓词（NSPredicate）创建正则表达式
谓词是iOS里面一个很牛逼的功能，通过它可以过滤很多东西，比如数据库SQL，输入内容匹配，还有我们要使用的正则表达式。

```objc
// 正则判断是不是正整数
NSPredicate *predicate = [NSPredicate predicateWithFormat:@"SELF MATCHES %@", @"^\\d+$"];
BOOL isValid = [predicate evaluateWithObject:@"123"];
if (isValid) {
	// do Something
}
```

例子中创建了谓词对象，设置匹配条件，如果匹配格式，在if判断中可以做处理。

这种判断方法很简洁，如果只是要判断输入内容是否合法，用这种方式就可以了。

## NSRegularExpression类创建正则表达式
在iOS中`NSRegularExpression`类就是一个专门来处理正则表达式的类，相比前面的方法，这个类更专业。

初始化**NSRegularExpression**的方法有两种，一个init方法和一个类方法。其作用基本是一样的：

```objc
+ (NSRegularExpression *)regularExpressionWithPattern:(NSString *)pattern options:(NSRegularExpressionOptions)options error:(NSError **)error;

- (instancetype)initWithPattern:(NSString *)pattern options:(NSRegularExpressionOptions)options error:(NSError **)error 
```

**pattern**是正则表达式，**options**参数是一个枚举，表示正则模式的设置，如下：

```objc
typedef NS_OPTIONS(NSUInteger, NSRegularExpressionOptions) {
   NSRegularExpressionCaseInsensitive             = 1 << 0, //不区分字母大小写的模式
   NSRegularExpressionAllowCommentsAndWhitespace  = 1 << 1, //忽略掉正则表达式中的空格和#号之后的字符
   NSRegularExpressionIgnoreMetacharacters        = 1 << 2, //将正则表达式整体作为字符串处理
   NSRegularExpressionDotMatchesLineSeparators    = 1 << 3, //允许.匹配任何字符，包括换行符  
   NSRegularExpressionAnchorsMatchLines           = 1 << 4, //允许^和$符号匹配行的开头和结尾
   NSRegularExpressionUseUnixLineSeparators       = 1 << 5, //设置\n为唯一的行分隔符，否则所有的都有效。
   NSRegularExpressionUseUnicodeWordBoundaries    = 1 << 6  //使用Unicode TR#29标准作为词的边界，否则所有传统正则表达式的词边界都有效
};
```

可以通过 | 来设置多项参数。

> 注意：
1.NSRegularExpressionCaseInsensitive模式下正则表达式 aBc 会匹配到abc.
2.NSRegularExpressionAllowCommentsAndWhitespace模式下正则表达式a b c 会匹配到abc，正则表达式ab#c会匹配到ab。
3.NSRegularExpressionIgnoreMetacharacters模式下正则表达式[a-z]，会匹配到[a-z]。



iOS提供了两个API来匹配表达式：

### block方法

```objc
// 匹配是不是a到z
NSRegularExpression * regex = [[NSRegularExpression alloc]initWithPattern:@"[a-z]" options:NSRegularExpressionCaseInsensitive error:nil];
[regex enumerateMatchesInString:@"124a" options:NSMatchingReportProgress range:NSMakeRange(0, 4) usingBlock:^(NSTextCheckingResult *result, NSMatchingFlags flags, BOOL *stop) {
   NSLog(@"%@",result);
} ];
```

函数的第一个参数options是个枚举：

```objc
typedef NS_OPTIONS(NSUInteger, NSMatchingOptions) {
   NSMatchingReportProgress         = 1 << 0, //每次都调用block回调，无论是否匹配
   NSMatchingReportCompletion       = 1 << 1, //找到任何一个匹配串后都回调一次block
   NSMatchingAnchored               = 1 << 2, //从匹配范围的开始出进行匹配
   NSMatchingWithTransparentBounds  = 1 << 3, //允许匹配的范围超出设置的范围
   NSMatchingWithoutAnchoringBounds = 1 << 4  //禁止^和$自动匹配行开始和结束
};
```

block回调的flags枚举：

```objc
typedef NS_OPTIONS(NSUInteger, NSMatchingFlags) {
   NSMatchingProgress               = 1 << 0, //匹配到最长串时被设置     
   NSMatchingCompleted              = 1 << 1, //全部分配完成后被设置    
   NSMatchingHitEnd                 = 1 << 2, //匹配到设置范围的末尾时被设置   
   NSMatchingRequiredEnd            = 1 << 3, //当前匹配到的字符串在匹配范围的末尾时被设置     
   NSMatchingInternalError          = 1 << 4  //由于错误导致的匹配失败时被设置   
};
```

block中还有个BOOL指针`stop`，设置它为**YES**就会停止匹配。


### 非block方法

* 这个方法返回一个结果数组，将所有的匹配结果返回。

```objc
- (NSArray *)matchesInString:(NSString *)string options:(NSMatchingOptions)options range:(NSRange)range;
```


* 这个方法会返回匹配到的字符串个数

```objc
- (NSUInteger)numberOfMatchesInString:(NSString *)string options:(NSMatchingOptions)options range:(NSRange)range;
```

* 这个方法会返回第一个查询到的结果，这个**NSTextCheckingResult**对象中有一个range属性，可以得到匹配到的字符串的范围。

```objc
- (NSTextCheckingResult *)firstMatchInString:(NSString *)string options:(NSMatchingOptions)options range:(NSRange)range;
```

* 这个方法直接返回匹配到得范围，NSRange。

```objc
- (NSRange)rangeOfFirstMatchInString:(NSString *)string options:(NSMatchingOptions)options range:(NSRange)range;
```


### 一个辅助方法
NSRegularExpression类中还提供了一个辅助方法：

```objc
+ (NSString *)escapedPatternForString:(NSString *)string;
```
它可以帮助我们将正则表达式加上"\"进行保护，将元字符转化成字面值。


## 总结
如果你只是简单的匹配用户输入内容是否匹配，**NSString实例方法**和**谓词**方法会方便简洁的给你答案，如果你是想要更加强大的匹配支持**NSRegularExpression**完全能够满足你的需要。

## 参考链接
[iOS中正则表达式的使用](https://my.oschina.net/u/2340880/blog/403638)





