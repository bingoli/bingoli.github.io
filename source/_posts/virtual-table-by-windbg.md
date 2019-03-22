---
title: virtual_table_by_windbg
date: 2019-03-21 23:24:29
tags:
 - C++
 - Windbg
---
在复杂情况下，要想用好C++，就得熟悉C++对象模型。如果能利用好调试工具，比如windbg、GDB等，就能够更快速的掌握C++对象模型原理。本人使用windbg剖析C++对象的相关原理，本文将先从虚函数表开始。 
说明：为了实现方便，本文采用了C++11的相关语法，编译环境为VS2017，编译选项为X86。
``` C++
#include <iostream>
using namespace std;

class CA {
public:
    int a = 1;
};

class CB {
public:
    virtual void FuncB() {}
    int a = 2;
};

class CC : public CB {
public:
    void FuncB() override {}
    virtual void FuncC() {}
};

int main() {
    CA a;
    CB b;
    CC c;
    cout << "Size of Pointer : " << sizeof(void *) << endl;
    cout << "Size of CA : " << sizeof(a) << endl;
    cout << "Size of CB : " << sizeof(b) << endl;
    cout << "Size of CC : " << sizeof(c) << endl;
    return 0;
}
```
32位模式下的运行结果如下：
```
Size of Pointer : 4
Size of CA : 4
Size of CB : 8
Size of CC : 8
```
从运行结果中看出，如果添加了虚函数，对象实体的大小就会增大4个字节，也就是虚函数表指针的大小。

## 关于作者
bingoli
微信公众号：bingoli
Blog: https://bingoli.github.io/
GitHub: https://github.com/bingoli


