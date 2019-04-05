---
title: JavaScript的Date类的函数特殊处理导致的问题
date: 2019-03-20 22:49:06
tags:
 - javascript
 - date
---

记得以前参加校招的时候，总是有日期相关的面试题，比如计算两个日期之间的间隔天数。以前还觉得这种题就是为了纯粹为了面试的，但工作了之后，还就碰到了跟日期相关的bug。下面是一段js代码，是要把字符串描述的日期转换为Date类型的函数。其中，字符串的格式为年占用4个字符，月份2个字符，天数2个字符，小时2个字符，分钟2个字符，秒数2个字符。
``` javascript
function str2Date(str) {
    var year = parseInt(str.substr(0, 4));
    // Month value range is [0, 11]
    var month = parseInt(str.substr(4, 2)) - 1;
    var day = parseInt(str.substr(6, 2));
    var hour = parseInt(str.substr(8, 2));
    var minute = parseInt(str.substr(10, 2));
    var second = parseInt(str.substr(12, 2));

    var date = new Date();
    date.setYear(year);
    date.setMonth(month);
    date.setDate(day);
    date.setHours(hour);
    date.setMinutes(minute);
    date.setSeconds(second);
    return date;
}

console.log(str2Date('20150515180000').toString());
console.log(str2Date('20160229235959').toString());
console.log(str2Date('20171231235959').toString());
console.log(str2Date('20181101000000').toString());
```

今天的日期是2019年3月20日，以上代码的测试用例运行结果全部正常。输出结果如下
```
Fri May 15 2015 18:00:00 GMT+0800 (CST)
Mon Feb 29 2016 23:59:59 GMT+0800 (CST)
Sun Dec 31 2017 23:59:59 GMT+0800 (CST)
Thu Nov 01 2018 00:00:00 GMT+0800 (CST)
```
但是在某个特定日期(月末），就发现这段代码的运行结果出错了。原来，new Date()生成的值是运行时的时间，在不同的时间，运行的结果是不同的。下面来看一个特定日期的运行结果。把第10行代码中的new Date的初始化，设置日期为2019年3月31日。
``` javascript
var date = new Date('2019-03-31 00:00:00');
```
修改之后的运行结果如下，有两个用例的运行结果是不符合预期的。
```
Fri May 15 2015 18:00:00 GMT+0800 (CST)
Tue Mar 29 2016 23:59:59 GMT+0800 (CST)
Sun Dec 31 2017 23:59:59 GMT+0800 (CST)
Sat Dec 01 2018 00:00:00 GMT+0800 (CST)
```
那究竟是怎么回事呢？只能逐步调试来看下哪一步出了问题。因为是日期错了，我这里通过日志把改变日期值的中间过程都打印出来。下面以第2个用例（2016年2月29日）调试看下。
``` javascript
function str2Date(str) {
    var year = parseInt(str.substr(0, 4));
    // Month value range is [0, 11]
    var month = parseInt(str.substr(4, 2)) - 1;
    var day = parseInt(str.substr(6, 2));
    var hour = parseInt(str.substr(8, 2));
    var minute = parseInt(str.substr(10, 2));
    var second = parseInt(str.substr(12, 2));

    var date = new Date('2019-03-31 00:00:00');
    date.setYear(year);
    console.log(date.toString());
    date.setMonth(month);
    console.log(date.toString());
    date.setDate(day);
    console.log(date.toString());
    date.setHours(hour);
    date.setMinutes(minute);
    date.setSeconds(second);
    return date;
}
```
输出的结果如下：
```
Thu Mar 31 2016 00:00:00 GMT+0800 (CST)
Wed Mar 02 2016 00:00:00 GMT+0800 (CST)
Tue Mar 29 2016 00:00:00 GMT+0800 (CST)
Tue Mar 29 2016 23:59:59 GMT+0800 (CST)
```
从输出结果来看SetMonth这一步就出错了。原来在当前场景下，调用SetMonth设置的月份，会导致预期时间是一个不合法的时间，而Date类进行了自动的纠正，也就改变了月份的值。后面又重新设置了天数，是合法值，Date不再对日期进行改变，最终就导致月份是错误的。
通过查询MDN文档，发现setFullYear函数是可以把年月日都作为参数输入，这样就能保证日期的完整性，特殊日期也不会出错。修改后的代码如下：
``` javascript
function str2Date(str) {
    var year = parseInt(str.substr(0, 4));
    // Month value range is [0, 11]
    var month = parseInt(str.substr(4, 2)) - 1;
    var day = parseInt(str.substr(6, 2));
    var hour = parseInt(str.substr(8, 2));
    var minute = parseInt(str.substr(10, 2));
    var second = parseInt(str.substr(12, 2));

    var date = new Date('2019-03-31 00:00:00');
    date.setFullYear(year, month, day);
    date.setHours(hour, minute, second);
    return date;
}

console.log(str2Date('20150515180000').toString());
console.log(str2Date('20160229235959').toString());
console.log(str2Date('20171231235959').toString());
console.log(str2Date('20181101000000').toString());
```
通过这个例子说明，每个函数的异常处理都有可能导致最终的结果出错，如果使用了系统函数或者其他库的函数，都需要对每个函数的异常处理有足够的了解，才能保证结果符合预期。

## 关于作者
微信公众号：程序员bingo
![微信公众号：程序员bingo](https://bingoli.github.io/wechat.jpeg)
Blog: https://bingoli.github.io/
GitHub: https://github.com/bingoli


