---
title: Windbg使用系列（1）：分析虚函数表
date: 2019-03-21 23:24:29
tags:
 - C++
 - Windbg
---

## 前言
要想学好C++，就得熟悉C++对象模型。如果能利用好调试工具，比如windbg、GDB等，就能够更快速的掌握C++对象模型原理。本系列文章是通过windbg来深入分析C++对象原理，以便更好的理解C++相关知识点。 
说明：为了实现方便，所有源码会采用了C++11的语法，编译环境为VS2017，默认编译选项为Win32。
## 检测虚函数表指针大小
根据C++对象的知识可知，存在虚函数的类对象实例会多出1个一个指向虚函数表的指针，下面就先用代码来测试一下，其中类CA不存在虚函数，类CB存在虚函数。
``` C++
#include <iostream>
using namespace std;

class CA {
public:
    void Fun1() {}
    void Fun2() {}
    void Fun3() {}
    void Fun4() {}
    int a = 1;
};

class CB {
public:
    virtual void Fun1() {}
    virtual void Fun2() {}
    void Fun3() {}
    void Fun4() {}
    int b = 2;
};

int main() {
    CA* pa = new CA();
    pa->Fun1();
    pa->Fun2();
    pa->Fun3();
    pa->Fun4();

    CB* pb = new CB();
    pb->Fun1();
    pb->Fun2();
    pb->Fun3();
    pb->Fun4();

    cout << "Size of Pointer : " << sizeof(void *) << endl;
    cout << "Size of CA : " << sizeof(*pa) << endl;
    cout << "Size of CB : " << sizeof(*pb) << endl;

    delete pa;
    pa = nullptr;
    delete pb;
    pb = nullptr;
    return 0;
}
```
在win32下的运行结果如下：
```
Size of Pointer : 4
Size of CA : 4
Size of CB : 8
```
从运行结果来看，CB的实例比CA的实例多出了4个字节，即一个指针的大小。通过VS的基本查看变量功能，我们无法看到虚函数表指针的值，也就看不到虚函数表的内容，需要用更专业的软件来看。下面，我们就用windbg来分析下虚函数表的具体内容。
## 使用Windbg分析虚函数表
在使用windbg之前，先编译上面的代码，生成了exe文件。本文的示例代码生成的是sample1.exe，后续用到的模块名称都是sample1。
### 1. 使用windbg打开exe
打开windbg，可Ctrl+E找到sample1.exe所在位置，把exe打开。打开之后，进程会暂停在进入main函数之前。这时候，我们可以设置断点，之后再运行。
### 2. 设置断点
设置断点需要知道要断点的代码的运行地址，这个地址可以通过x命令来模糊查询，比如我这次操作就是先查找main函数的地址，然后在main的入口处打断点，运行后，windbg会自动显示代码，就可以在左侧代码中通过界面来设置断点了。我把断点打在了pb创建之后的下一行代码上。
查找main函数的命令如下。其中类似0:000>开头的，是我输入的命令，其他的是输入命令后的运行结果。
```
0:000> X sample1!main*
*** WARNING: Unable to verify checksum for sample1.exe
00a25fd0          sample1!main (void)
00a22bf0          sample1!mainCRTStartup (void)
0:000> bp 00a25fd0
0:000> g
Breakpoint 0 hit
eax=10104750 ebx=003d3000 ecx=00000001 edx=00a2c73c esi=00a212b7 edi=00a212b7
eip=00a25fd0 esp=0053fec8 ebp=0053fed8 iopl=0         nv up ei pl nz na pe nc
cs=0023  ss=002b  ds=002b  es=002b  fs=0053  gs=002b             efl=00000206
sample1!main:
00a25fd0 55              push    ebp
```
在pb创建之后设置好断点之后，再次输入g命令，程序会继续暂停。
### 3. 对比类实例的值
在当前的程序中，已经创建了pa和pb，可以通过dt命令来看下两个变量的详细信息，从下面的结果可以看出，pb比pa多了一个虚函数表指针。
```
0:000> dt pa
Local var @ 0x53fec0 Type CA*
0x0006a6a8 
   +0x000 a                : 0n1
0:000> dt pb
Local var @ 0x53febc Type CB*
0x00065c70 
   +0x000 __VFN_table : 0x00a29b7c 
   +0x004 b                : 0n2
```
### 4. 查看虚函数表内容
从上面的结果看出，pb的虚函数表的指针是0x00a29b7c，可通过dt查看pb的虚函数表内容。从dt的输出结果看，没有看到完整的信息。因此，可以通过dd命令查看虚函数表的内存，在用dt命令查看对应内存的信息。通过结果看出，虚函数表里的两个函数，就是pb的两个虚函数Fun1和Fun2。
```
0:000> dt 0x00a29b7c 
CB::`vftable'
[3] 0x00a21307 
 void  sample1!ILT+770(?Fun1CBUAEXXZ)+0( void )
0:000> dd 0x00a29b7c 
00a29b7c  00a21307 00a2130c 00000000 00000000
00a29b8c  00000000 00000000 00a2a8b8 00a210dc
00a29b9c  00000000 00a2a910 00a21159 00a21294
00a29bac  00000000 6e6b6e55 206e776f 65637865
00a29bbc  6f697470 0000006e 00000000 00a2a968
00a29bcc  00a21131 00a21294 00000000 20646162
00a29bdc  6f6c6c61 69746163 00006e6f 00000000
00a29bec  00a2a9c4 00a21203 00a21294 00000000
0:000> dt 00a21307 
ILT+770(?Fun1CBUAEXXZ)
Symbol  not found.
0:000> dt 00a2130c 
ILT+775(?Fun2CBUAEXXZ)
Symbol  not found.
```
### 5. 进一步分析虚函数表
x命令可以查询程序中的符号的信息，下面我使用x查询了CA和CB相关的符号信息。其中，CB的虚函数表指针，跟pb中的虚函数表地址是一样的，说明所有类实例共用一个虚函数表，这个也可以通过再创建一个类对象来验证。
```
0:000> x sample1!CA::*
00a22260          sample1!CA::Fun1 (void)
00a21ca0          sample1!CA::Fun2 (void)
00a21850          sample1!CA::Fun3 (void)
00a21830          sample1!CA::Fun4 (void)
00a21e10          sample1!CA::CA (void)
0:000> x sample1!CB::*
00a29b7c          sample1!CB::`vftable' = <function> *[3]
00a21ef0          sample1!CB::Fun3 (void)
00a21cf0          sample1!CB::Fun2 (void)
00a21800          sample1!CB::Fun1 (void)
00a21cc0          sample1!CB::Fun4 (void)
00a21e40          sample1!CB::CB (void)
00a2a830          sample1!CB::`RTTI Base Class Array' = <no type information>
00a2a81c          sample1!CB::`RTTI Class Hierarchy Descriptor' = <no type information>
00a2a838          sample1!CB::`RTTI Base Class Descriptor at (0,-1,0,64)' = <no type information>
00a2a804          sample1!CB::`RTTI Complete Object Locator' = <no type information>
```
通过上面的结果看出，CB::Fun1的地址是00a21800，跟pb的虚函数表里的00a21307地址不相同的。然而，我通过uf命令，发现这反汇编出来的函数是一样的。这又是怎么回事呢？先来看看反汇编的结果。
```
0:000> uf 0xa21307
sample1!CB::Fun1 [c:\bingo\github\samples\cplusplus\inside-object\sample2\main.cpp @ 15]:
   15 00a21800 55              push    ebp
   15 00a21801 8bec            mov     ebp,esp
   15 00a21803 51              push    ecx
   15 00a21804 c745fccccccccc  mov     dword ptr [ebp-4],0CCCCCCCCh
   15 00a2180b 894dfc          mov     dword ptr [ebp-4],ecx
   15 00a2180e 8be5            mov     esp,ebp
   15 00a21810 5d              pop     ebp
   15 00a21811 c3              ret

sample1!ILT+770(?Fun1CBUAEXXZ):
00a21307 e9f4040000      jmp     sample1!CB::Fun1 (00a21800)  Branch
```
从反汇编的结果来看，00a21800是CB::Fun1的入口地址，而0xa21307指向的值是跳转到CB::Fun1函数的命令，也就是说，虚函数表保存的是一个跳转命令。
## 小结
通过windbg工具辅助，把很多VS不能显示信息展示出来，能够了解更多C++虚函数表的实现细节，对掌握C++的相关原理很有帮助。后续，我还会用windbg分析更多的实例。
## 关于作者
bingoli
微信公众号：bingoli
Blog: https://bingoli.github.io/
GitHub: https://github.com/bingoli


