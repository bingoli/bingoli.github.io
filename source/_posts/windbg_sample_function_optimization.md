---
title: 为什么反汇编代码跟C++源码匹配不上？
date: 2020-1-26 10:22:30
tags:
- C++
- Windbg
- 汇编
---

# 引言

在分析C++代码的崩溃、卡顿等问题时，经常需要用到反汇编代码，但由于编译器的优化处理，经常会出现反汇编代码和C++代码不匹配的情况，这就需要我们仔细核对源码和反汇编代码，才能最终定位问题的原因。本文介绍的例子，是由于两个函数的内容相同，而被编译器优化成了一个函数，因此导致了反汇编代码和C++代码不匹配的问题。

# 编译和运行环境

本文使用的编译器是VS2019，编译配置为Release Win32，并把编译优化选项关闭，工程生成的可执行文件为sample.exe。用windbg把sample.exe运行，会自动中止运行。先要把sample.exe的符号加载上，才能打断点和查询相关类和函数的符号。

> 0:000> .reload /s /f sample.exe

# 反汇编代码

在调试的排查问题时，反汇编了构造函数和析构函数的源码，发现构造函数的反汇编代码和C++源码匹配不上。在反汇编代码中，构造函数竟然调用了Uninit，而实际上是调用的Init。

> 0:000> uf sample!CA::CA
sample!CA::CA [C:\Users\bingo\source\repos\sample\sample.cpp @ 7]:
    7 00731000 55              push    ebp
    7 00731001 8bec            mov     ebp,esp
    7 00731003 51              push    ecx
    7 00731004 894dfc          mov     dword ptr [ebp-4],ecx
    8 00731007 8b4dfc          mov     ecx,dword ptr [ebp-4]
    **8 0073100a e851000000      call    sample!CA::Uninit (00731060)**
    9 0073100f 8b45fc          mov     eax,dword ptr [ebp-4]
    9 00731012 8be5            mov     esp,ebp
    9 00731014 5d              pop     ebp
    9 00731015 c3              ret
> 

析构函数的反汇编代码是正常的，是调用了Unint函数。
  
> 0:000> uf sample!CA::~CA
sample!CA::~CA [C:\Users\bingo\source\repos\sample\sample.cpp @ 11]:
   11 00731020 55              push    ebp
   11 00731021 8bec            mov     ebp,esp
   11 00731023 6aff            push    0FFFFFFFFh
   11 00731025 68d0257300      push    offset sample!_filter_x86_sse2_floating_point_exception_default+0x7f (007325d0)
   11 0073102a 64a100000000    mov     eax,dword ptr fs:[00000000h]
   11 00731030 50              push    eax
   11 00731031 51              push    ecx
   11 00731032 a104507300      mov     eax,dword ptr [sample!__security_cookie (00735004)]
   11 00731037 33c5            xor     eax,ebp
   11 00731039 50              push    eax
   11 0073103a 8d45f4          lea     eax,[ebp-0Ch]
   11 0073103d 64a300000000    mov     dword ptr fs:[00000000h],eax
   11 00731043 894df0          mov     dword ptr [ebp-10h],ecx
   12 00731046 8b4df0          mov     ecx,dword ptr [ebp-10h]
   **12 00731049 e812000000      call    sample!CA::Uninit (00731060)**
   13 0073104e 8b4df4          mov     ecx,dword ptr [ebp-0Ch]
   13 00731051 64890d00000000  mov     dword ptr fs:[0],ecx
   13 00731058 59              pop     ecx
   13 00731059 8be5            mov     esp,ebp
   13 0073105b 5d              pop     ebp
   13 0073105c c3              ret
>

这到底是怎么回事呢？是不是Init函数被编译器遗漏了？是不是编译错误？进一步看下CA类的所有函数。

> 0:000> x sample!CA::*
**00731060          sample!CA::Uninit (void)**
00731150          sample!CA::`scalar deleting destructor' (void)
00731080          sample!CA::Fun (void)
00731000          sample!CA::CA (void)
00731020          sample!CA::~CA (void)
**00731060          sample!CA::Init (void)**
>

原来CA::Init并没有被编译器遗漏了，只是和CA::Uninit的函数地址是一样的，也就是说，调用Init和Unit实际上运行得是同一段代码。

# 源码分析

现在来看一下源码，源码比较简单，就是在构造函数和析构函数中，分别调用了一个初始化和反初始化的函数，而两个函数的具体实现相同。因此，编译器为了节省空间，把两个函数合并成了一个函数，而Windbg在反汇编的时候，就会出现一个地址对应多个函数，而在显示时，只显示一个函数名称，也就会出现某些匹配错误的现象。

``` C++
#include <iostream>

using namespace std;

class CA {
public:
  CA() {
    Init();
  }

  ~CA() {
    Uninit();
  };

  void Init() {
    Fun();
  }

  void Uninit() {
    Fun();
  }

  void Fun() {
    cout << "CA::Fun" << endl;
  }
};

int main() {
  CA* pa = new CA();
  delete pa;
  pa = nullptr;
  return 0;
}
```

# 小结

反汇编代码并不能完整匹配C++代码， 还需要我们结合上下文进行判断他们之间的对应关系。了解了编译器的一些优化手段，会让我们能够更快的排查问题。

# 关于作者
微信公众号：程序员bingo
![微信公众号：程序员bingo](https://bingoli.github.io/wechat.jpeg)
Blog: https://bingoli.github.io/
GitHub: https://github.com/bingoli