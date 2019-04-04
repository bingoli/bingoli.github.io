---
title: 使用C++宏嵌套实现窄字符转换为宽字符
date: 2019-04-04 22:10:23
tags:
 - C++
 - 宏
 - 宽字符
---
## 背景
随着软件的发展，软件内置字符串的变更是很常见的需求，比如国际化、多品牌相关的字符串设置。把会重复使用的字符串定义为宏，是减少需求变更成本的一个常用手段，也是很多团队软件开发的基本规范。在实际开发过程中，也有很多人觉得这是小技巧，并没有遵守这个规范，从而导致字符串的变更变得更加困难，因为很容易漏修改。因此，在开发过程中，还是应该尽量减少冗余信息。但是，即使使用了宏定义，也仍然还会有冗余信息，比如同一个字符串既需要窄字符，也需要宽字符。比如，下面代码定义的产品名称，就存在冗余信息，如果需要修改产品名称时，就需要修改2处。如果这类冗余信息多了，改起来就会很痛苦，也很容易出错（谁改谁知道）。
``` C++
// 相关定义
#define PRODUCT_NAME "Chrome"
#define PRODUCT_NAME_W L"Chrome"

// 实际使用
std::string product_name = PRODUCT_NAME;
std::wstring product_name_w = PRODUCT_NAME_W;
```
为了减少冗余信息，可通过宏来实现窄字符转换为宽字符，最终期望的宏应该是这样定义的：
``` C++
#define PRODUCT_NAME "Chrome"
#define PRODUCT_NAME_W TO_UNICODE(PRODUCT_NAME)
```
接下来的任务就是需要实现宏TO_UNICODE。
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
从错误提示中可以看出，PRODUCT_NAME并没有被展开。这里是因为##直接就把参数x的内容连接起来了。
## 宏嵌套的实现
为了解决宏没有展开的问题，就需要用到宏嵌套了，要让宏PRODUCT_NAME在使用##连接之前就展开。实现起来也很简单，就是在加一层宏转换。实现代码如下：
``` C++
#define _TO_UNICODE(y) L##y
#define TO_UNICODE(x) _TO_UNICODE(x)

#define PRODUCT_NAME "Chrome"
#define PRODUCT_NAME_W TO_UNICODE(PRODUCT_NAME)

std::string product_name = PRODUCT_NAME;
std::wstring product_name_w = PRODUCT_NAME_W;
```
在以上代码的PRODUCT_NAME_W定义中，宏TO_UNICODE的参数x为宏PRODUCT_NAME，而x只是作为宏_TO_UNICODE的输入参数，因此，此时可以展开宏PRODUCT_NAME。展开后的结果如下：
``` 
_TO_UNICODE("Chrome")
```
展开后，_TO_UNICODE还是一个有效的宏，可再次展开，展开后为：
```
L"Chrome"
```
这个最终结果就符合预期了。有了窄字符转换为宽字符的宏，同一字符串就只需要修改一次了。
## 关于作者
微信公众号：程序员bingo
![公众号：程序员bingo](bingo_wechat.jpeg)
Blog: https://bingoli.github.io/
GitHub: https://github.com/bingoli