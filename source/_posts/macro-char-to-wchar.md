---
title: 使用C++宏嵌套实现窄字符转换为宽字符
date: 2019-04-04 22:10:23
tags:
 - C++
 - 宏
 - 宽字符
---
## 背景
我们经常用宏定义减少字符串的冗余信息，从而让国际化或多品牌的需求变更能够更快速的实现。由于软件中需要同时用到窄字符和宽字符，因此，很容易产生一些冗余信息。例如，下面的代码用不同的宏定义了窄字符和宽字符。
``` C++
// 相关定义
#define PRODUCT_NAME "Chrome"
#define PRODUCT_NAME_W L"Chrome"

// 实际使用
std::string product_name = PRODUCT_NAME;
std::wstring product_name_w = PRODUCT_NAME_W;
```
如果能通过宏实现窄字符转换为宽字符，就可以减少这部分冗余信息。期望的宏应该是这样定义的：
``` C++
#define PRODUCT_NAME "Chrome"
#define PRODUCT_NAME_W TO_UNICODE(PRODUCT_NAME)
```
接下来的任务就是实现宏TO_UNICODE。
## 宏参数的编译错误
在宏定义中，#和##很常用。其中，#是把宏参数转换为字符串；##是连接符，把宏参数和其他数据直接连接起来。宽字符实际上就是窄字符前面多了一个L，因此，宏TO_UNICODE应该使用##把L和窄字符连接起来。定义如下：
``` C++
#define TO_UNICODE(x) L##x
```
经过测试，发现使用TO_UNICODE直接转换字符串是没有问题的。如下面的代码：
``` C++
// 结果正确，宏PRODUCT_NAME_W展开后是L"Chrome"
#define PRODUCT_NAME_W TO_UNICODE("Chrome")
std::wstring product_name = PRODUCT_NAME_W;
```
但是，如果宏的参数也是一个宏，但编译就会出错。编译出错的代码如下所示。
``` C++
// 编译错误，宏PRODUCT_NAME_W展开后是LPRODUCT_NAME
#define PRODUCT_NAME_W TO_UNICODE(PRODUCT_NAME)
std::wstring product_name = PRODUCT_NAME_W;
```
编译的错误提示是：
```
error C2065: 'LPRODUCT_NAME': undeclared identifier
```
从错误提示中可以看出，PRODUCT_NAME并没有被展开。这里是因为##的特殊性，此时，如果x是宏，也不会展开参数x的内容。
## 宏嵌套的实现
为了解决宏没有展开的问题，需要用到宏嵌套，即要让宏PRODUCT_NAME在使用##之前就展开。实现起来也很简单，就是再加一层宏转换。实现代码如下：
``` C++
#define _TO_UNICODE(y) L##y
#define TO_UNICODE(x) _TO_UNICODE(x)

#define PRODUCT_NAME "Chrome"
#define PRODUCT_NAME_W TO_UNICODE(PRODUCT_NAME)

std::string product_name = PRODUCT_NAME;
std::wstring product_name_w = PRODUCT_NAME_W;
```
## 结语
宽窄字符使用不同的宏定义，在实践中是很常见的一个问题。字符串冗余在平时只是一个小问题，如果数量多了，在需要修改的时候，就会是一个大问题。宏嵌套是个小技巧，却能让我们的代码更规范。
## 关于作者
微信公众号：程序员bingo
![微信公众号：程序员bingo](https://bingoli.github.io/wechat.jpeg)
Blog: https://bingoli.github.io/
GitHub: https://github.com/bingoli