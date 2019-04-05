---
title: 使用Windbg分析C++的多重继承原理
date: 2019-03-27 06:35:56
tags:
 - C++
 - 多重继承
 - Windbg
---
## 前言
windbg是windows平台下C++开发调试的工具，常用于分析软件崩溃，也是学习C++开发的利器。windbg功能很强大，通过它的命令，我们能看到软件底层的实现原理。本系列文章就是借助windbg工具，对C++的实现原理进行深入分析。本文使用windbg分析C++的多重继承的实现原理，主要涉及的内容是：
1. 多重继承的内存分布
2. 多重继承的类型转换

## 通过源码分析多重继承的空间占用
先来看看源码，定义了3个类，CC继承自CA和CB。
说明：为了实现方便，所有源码会采用了C++11的语法，编译环境为VS2017，默认编译选项为Win32。
``` C++
#include <iostream>
using namespace std;

class CA {
public:
    virtual void FunA() {}
    int a = 1; 
};

class CB {
public:
    virtual void FunB() {}
    int b = 2;
};

class CC : public CA, public CB {
public:
    virtual void FunC() {}
    int c = 3;
};

int main() {
    // 打印出各个类占用空间大小
    cout << "sizeof CA : " << sizeof(CA) << endl;
    cout << "sizeof CB : " << sizeof(CB) << endl;
    cout << "sizeof CC : " << sizeof(CC) << endl;

    // 安全的向上类型转换
    CC* pc = new CC();
    CA* pa = pc;
    CB* pb = pc;

    // 不安全的向下类型转换
    CC* pc21 = (CC*)pa;
    CC* pc22 = (CC*)pb;

    // 错误的类型转换
    CA* pa31 = (CA*)pb;
    CB* pb31 = (CB*)pa;
    CB* pb32 = reinterpret_cast<CB*>(pc);

    delete pc;
    
    return 0;
}
```
### 设置断点
设置断点的过程如下：
1. 通过x命令找到main函数的地址。
2. 在main函数的地址上设置断点，使用g命令继续运行，程序会在main函数入口处挂起，并在左侧窗口打开代码。
3. 鼠标点击代码delete pc;所在行，按下F9设置断点，然后g命令继续运行，程序会在代码delete pc;所在行挂起。
后面的操作会在这个断点下进行。
相关的操作的命令如下：
```
0:000> x sample!main*
00911600          sample!main (void)
0:000> bp 00911600
0:000> g
```
### 占用内存分析
先通过打印出来的日志，看下各个类占用的内存大小。
```
sizeof CA : 8
sizeof CB : 8
sizeof CC : 20
```
其中，CC的大小是CA+CB再加上一个int的大小，说明CC是包含了CA和CB的虚函数表指针。通过windbg命令来看下。
```
0:000> dt pc
Local var @ 0x53fe9c Type CC*
0x0068be88 
   +0x000 __VFN_table : 0x00919b84 
   +0x004 a                : 0n1
   +0x008 __VFN_table : 0x00919b94 
   +0x00c b                : 0n2
   +0x010 c                : 0n3
```
运行结果显示，CC的内存分布依次是
1. CA的虚函数表指针
2. CA的成员变量a
3. CB的虚函数表指针
4. CB的成员变量b
5. CC的成员变量c

### 向上转换
向上转换，即从子类转换为父类，是安全的。先来看看转换后的内存地址和结构。
```
0:000> dt pa
Local var @ 0x53fe98 Type CA*
0x0068be88 
   +0x000 __VFN_table : 0x00919b84 
   +0x004 a                : 0n1
0:000> dt pb
Local var @ 0x53fe94 Type CB*
0x0068be90 
   +0x000 __VFN_table : 0x00919b94 
   +0x004 b                : 0n2
```
由于CA位于CC的头部，两者地址是一样的，转换是直接转换就可以，但是CB跟CC是有8个字节的偏移的。这个向上转换是静态转换，编译的时候就已经完成，因为编译器知道类的内存结构，在生成汇编代码的时候，只需要偏移8个字节。通过反汇编的代码也说明了这一点。可通过uf main来反汇编整个main函数，下面节选了从pc转换到pb附近的反汇编代码，其中add ecx,8就是处理偏移量。
```
0:000> uf main
   29 00911719 8b4dfc          mov     ecx,dword ptr [ebp-4]
   29 0091171c 83c108          add     ecx,8
   29 0091171f 894dd0          mov     dword ptr [ebp-30h],ecx
   29 00911722 eb07            jmp     sample!main+0x12b (0091172b)
```
### 向下转换
向下转换，即从父类转换到子类，不一定是安全的，因为父类可能仅仅是个父类。本示例中，以为是简单已知的模型，知道是能正确转换，仅为了说明转换的实现原理。这里的类CB转换到类CC，编译器也是通过计算偏移量实现，正好跟向上转换是反向操作，执行代码是sub ecx,8。
```
0:000> uf main
   32 0091173d 8b4df4          mov     ecx,dword ptr [ebp-0Ch]
   32 00911740 83e908          sub     ecx,8
   32 00911743 894dcc          mov     dword ptr [ebp-34h],ecx
   32 00911746 eb07            jmp     sample!main+0x14f (0091174f)
```
### 危险的类型转换
如果两个类之间没有继承关系，如果使用()进行类型转换的话，编译就只能使用地址强制转换了。这样的操作是非常危险的。我这个例子中，由于CA和CB的内存结构相同，转换后并不会造成崩溃，但是里面的值是交换的。如果程序继续运行，结果可能就是不符合预期的。
```
0:000> dt pa31
Local var @ 0x53fe88 Type CA*
0x0068be90 
   +0x000 __VFN_table : 0x00919b94 
   +0x004 a                : 0n2
0:000> dt pb31
Local var @ 0x53fe84 Type CB*
0x0068be88 
   +0x000 __VFN_table : 0x00919b84 
   +0x004 b                : 0n1
```
## 结语
类的类型转换，在实际使用过程中，很容易造成各种问题。如果使用了多重继承，更是加大了出现问题的概率。了解了多重继承的原理，我们在实际调试中，才能更好去调试软件的异常，从而修复并规避相关问题的出现。
## 关于作者
微信公众号：程序员bingo
![微信公众号：程序员bingo](https://bingoli.github.io/wechat.jpeg)
Blog: https://bingoli.github.io/
GitHub: https://github.com/bingoli