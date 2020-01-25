---
title: C++的构造函数可以调用其他虚函数吗？
date: 2020-1-9 22:34:39
tags:
- C++
- Windbg
- 虚函数
---

# 引言

在C++面试中，构造函数和析构函数应该是经常会聊到的一类话题，比如

* 子类和父类中，构造函数和析构函数的调用顺序是什么？
* 构造函数和析构函数，可以调用其他的虚函数吗？

对于这些问题，只要了解C++的构造和析构原理，以及调用虚函数的原理，就能很轻松的回答上来。很多C++教科书中，都会讲解构造函数和析构函数的原理，但对于一些实现细节，比如虚函数表指针的初始化，讲解的并不是特别详细。而且这些C++的原理，没有在C++代码中体现，但反汇编代码里是有这些细节的，因此，通过研究反汇编代码，是可以更容易的理解这些的原理。本文将通过一个例子，讲解使用Windbg通过反汇编，分析C++的构造函数和析构函数对虚函数的调用实现。

# 源码

先来看下源码，下面的代码实现了一个父类CBase和一个子类CDerived，在父类和子类中构造函数和析构函数中，都调用了虚函数Init和Uninit。为了在反汇编代码中更容易的区分父类和子类，在CBase类中，构建了一个成员变量base_value，并赋初始值为0x1234;在CDerived类中，构建了一个成员变量derived_value，并赋初始值为0x5678。

``` C++
#include <iostream>

using namespace std;

class CBase {
public:
  CBase() {
    Init();
  }

  virtual ~CBase() {
    Uninit();
  };

  virtual void Init() {
    Fun();
  }

  virtual void Uninit() {
    cout << "CBase::Uninit" << endl;
  }

  virtual void Fun() {
    cout << "CBase::Fun" << endl;
  }

  int base_value = 0x1234;
};

class CDerived : public CBase {
public:
  CDerived() {
    Init();
  }

  virtual ~CDerived() {
    Uninit();
  }

  virtual void Init() {
    cout << "CDerived::Init" << endl;
  }

  virtual void Uninit() {
    cout << "CDerived::Uninit" << endl;
  }

  virtual void Fun() {
    cout << "CDerived::Fun" << endl;
  }

  int derived_value = 0x5678;
};

int main() {
  CDerived* derived = new CDerived();
  delete derived;
  derived = nullptr;
  return 0;
}
```

# 编译和运行环境

本文使用的编译器是VS2019，编译配置为Release Win32，并把编译优化选项关闭，工程生成的可执行文件为sample.exe。用windbg把sample.exe运行，会自动中止运行。先要把sample.exe的符号加载上，才能打断点和查询相关类和函数的符号。

> 0:000> .reload /s /f sample.exe

# 构造函数、析构函数的调用顺序

要了解构造函数和析构函数的调用顺序，只需要在相关函数中打印一些信息，就很容易获得到相关顺序。相关信息打印如下：

> CBase::CBase()
CDerived::CDerived()
CDerived::CDerived()
CBase::CBase()
>

结论如下：

* 构造时，先调用父类构造函数，再调用子类构造函数
* 析构时，先调用子类析构函数，再调用父类析构函数

为了简化反汇编代码，后续代码中注释了这些打印信息。

# 构造函数的实现细节

为了了解构造函数的实现细节，需要对构造函数的代码进行反汇编，先把子类的代码进行展开。CDerived的构造函数中的主要流程如下

* 调用父类CBase的构造函数
* 把CDerived的虚函数表地址赋值给对象的虚函数表指针
* 初始化CDerived的成员变量
* 调用函数Init

> 0:000> uf sample!CDerived::CDerived
sample!CDerived::CDerived :
   36 00eb1160 55              push    ebp
   36 00eb1161 8bec            mov     ebp,esp
   36 00eb1163 6aff            push    0FFFFFFFFh
   36 00eb1165 681828eb00      push    offset sample!_filter_x86_sse2_floating_point_exception_default+0xaa (00eb2818)
   36 00eb116a 64a100000000    mov     eax,dword ptr fs:[00000000h]
   36 00eb1170 50              push    eax
   36 00eb1171 51              push    ecx
   36 00eb1172 a10450eb00      mov     eax,dword ptr [sample!__security_cookie (00eb5004)]
   36 00eb1177 33c5            xor     eax,ebp
   36 00eb1179 50              push    eax
   36 00eb117a 8d45f4          lea     eax,[ebp-0Ch]
   36 00eb117d 64a300000000    mov     dword ptr fs:[00000000h],eax
   36 00eb1183 894df0          mov     dword ptr [ebp-10h],ecx
   36 00eb1186 8b4df0          mov     ecx,dword ptr [ebp-10h]
   // 调用父类的构造函数
   **36 00eb1189 e872feffff      call    sample!CBase::CBase (00eb1000)**
   36 00eb118e c745fc00000000  mov     dword ptr [ebp-4],0
   // 得到this指针
   36 00eb1195 8b45f0          mov     eax,dword ptr [ebp-10h]
   // 把CDerived的虚函数表指针赋值给对象的虚函数表指针
   **36 00eb1198 c7002032eb00    mov     dword ptr [eax],offset sample!CDerived::`vftable' (00eb3220)**
   56 00eb119e 8b4df0          mov     ecx,dword ptr [ebp-10h]
   // 初始化CDerived的成员变量derived_value
   **56 00eb11a1 c7410878560000  mov     dword ptr [ecx+8],5678h**
   37 00eb11a8 8b4df0          mov     ecx,dword ptr [ebp-10h]
   // 调用函数Init
   **37 00eb11ab e870000000      call    sample!CDerived::Init (00eb1220)**
   38 00eb11b0 c745fcffffffff  mov     dword ptr [ebp-4],0FFFFFFFFh
   38 00eb11b7 8b45f0          mov     eax,dword ptr [ebp-10h]
   38 00eb11ba 8b4df4          mov     ecx,dword ptr [ebp-0Ch]
   38 00eb11bd 64890d00000000  mov     dword ptr fs:[0],ecx
   38 00eb11c4 59              pop     ecx
   38 00eb11c5 8be5            mov     esp,ebp
   38 00eb11c7 5d              pop     ebp
   38 00eb11c8 c3              ret
>

CBase的构造函数的主要流程如下：

* 把CBase的虚函数表地址赋值给对象的虚函数表指针
* 初始化CBase的成员变量
* 调用函数Init

> 0:000> uf sample!CBase::CBase
sample!CBase::CBase :
    7 00eb1000 55              push    ebp
    7 00eb1001 8bec            mov     ebp,esp
    7 00eb1003 51              push    ecx
    7 00eb1004 894dfc          mov     dword ptr [ebp-4],ecx
    7 00eb1007 8b45fc          mov     eax,dword ptr [ebp-4]
    // 把CBase的虚函数表地址赋值给对象的虚函数表指针
    **7 00eb100a c7003432eb00    mov     dword ptr [eax],offset sample!CBase::`vftable' (00eb3234)**
    // 得到this指针的值
   31 00eb1010 8b4dfc          mov     ecx,dword ptr [ebp-4]
    // 初始化CBase的成员变量base_value
   **31 00eb1013 c7410434120000  mov     dword ptr [ecx+4],1234h**
    8 00eb101a 8b4dfc          mov     ecx,dword ptr [ebp-4]
    // 调用函数Init
    **8 00eb101d e85e000000      call    sample!CBase::Init (00eb1080)**
    9 00eb1022 8b45fc          mov     eax,dword ptr [ebp-4]
    9 00eb1025 8be5            mov     esp,ebp
    9 00eb1027 5d              pop     ebp
    9 00eb1028 c3              ret
>

# 析构函数的实现细节

CDerived析构函数的主要流程如下

* 把CDerived的虚函数表地址赋值给对象的虚函数表指针
* 调用函数Uninit
* 调用父类CBase的析构函数

> 0:000> uf sample!CDerived::~CDerived
sample!CDerived::~CDerived :
   40 00eb11d0 55              push    ebp
   40 00eb11d1 8bec            mov     ebp,esp
   40 00eb11d3 6aff            push    0FFFFFFFFh
   40 00eb11d5 68f027eb00      push    offset sample!_filter_x86_sse2_floating_point_exception_default+0x82 (00eb27f0)
   40 00eb11da 64a100000000    mov     eax,dword ptr fs:[00000000h]
   40 00eb11e0 50              push    eax
   40 00eb11e1 51              push    ecx
   40 00eb11e2 a10450eb00      mov     eax,dword ptr [sample!__security_cookie (00eb5004)]
   40 00eb11e7 33c5            xor     eax,ebp
   40 00eb11e9 50              push    eax
   40 00eb11ea 8d45f4          lea     eax,[ebp-0Ch]
   40 00eb11ed 64a300000000    mov     dword ptr fs:[00000000h],eax
   40 00eb11f3 894df0          mov     dword ptr [ebp-10h],ecx
   40 00eb11f6 8b45f0          mov     eax,dword ptr [ebp-10h]
   // 把CDerived的虚函数表地址赋值给对象的虚函数表指针
   **40 00eb11f9 c7002032eb00    mov     dword ptr [eax],offset sample!CDerived::`vftable' (00eb3220)**
   41 00eb11ff 8b4df0          mov     ecx,dword ptr [ebp-10h]
   // 调用函数Uninit
   **41 00eb1202 e849000000      call    sample!CDerived::Uninit (00eb1250)**
   42 00eb1207 8b4df0          mov     ecx,dword ptr [ebp-10h]
   // 调用父类CBase的析构函数
   **42 00eb120a e821feffff      call    sample!CBase::~CBase (00eb1030)**
   42 00eb120f 8b4df4          mov     ecx,dword ptr [ebp-0Ch]
   42 00eb1212 64890d00000000  mov     dword ptr fs:[0],ecx
   42 00eb1219 59              pop     ecx
   42 00eb121a 8be5            mov     esp,ebp
   42 00eb121c 5d              pop     ebp
   42 00eb121d c3              ret

CBase的析构函数的主要流程如下：

* 把CBase的虚函数表地址赋值给对象的虚函数表指针
* 调用函数Uninit

> 0:000> uf sample!CBase::~CBase
sample!CBase::~CBase :
   11 00eb1030 55              push    ebp
   11 00eb1031 8bec            mov     ebp,esp
   11 00eb1033 6aff            push    0FFFFFFFFh
   11 00eb1035 68f027eb00      push    offset sample!_filter_x86_sse2_floating_point_exception_default+0x82 (00eb27f0)
   11 00eb103a 64a100000000    mov     eax,dword ptr fs:[00000000h]
   11 00eb1040 50              push    eax
   11 00eb1041 51              push    ecx
   11 00eb1042 a10450eb00      mov     eax,dword ptr [sample!__security_cookie (00eb5004)]
   11 00eb1047 33c5            xor     eax,ebp
   11 00eb1049 50              push    eax
   11 00eb104a 8d45f4          lea     eax,[ebp-0Ch]
   11 00eb104d 64a300000000    mov     dword ptr fs:[00000000h],eax
   11 00eb1053 894df0          mov     dword ptr [ebp-10h],ecx
   11 00eb1056 8b45f0          mov     eax,dword ptr [ebp-10h]
   // 把CBase的虚函数表地址赋值给对象的虚函数表指针
   **11 00eb1059 c7003432eb00    mov     dword ptr [eax],offset sample!CBase::`vftable' (00eb3234)**
   12 00eb105f 8b4df0          mov     ecx,dword ptr [ebp-10h]
   // 调用函数Uninit
   **12 00eb1062 e849000000      call    sample!CBase::Uninit (00eb10b0)**
   13 00eb1067 8b4df4          mov     ecx,dword ptr [ebp-0Ch]
   13 00eb106a 64890d00000000  mov     dword ptr fs:[0],ecx
   13 00eb1071 59              pop     ecx
   13 00eb1072 8be5            mov     esp,ebp
   13 00eb1074 5d              pop     ebp
   13 00eb1075 c3              ret

# 构造函数调用虚函数

通过上面的反汇编代码看出，在构造函数或析构函数，调用其他虚函数时，是通过call命令调用函数地址，跟普通函数调用方式一样。

CBase的构造函数调用Init函数

> 8 00eb101d e85e000000      call    sample!CBase::Init (00eb1080)

CDerived的构造函数调用Init函数

> 37 00eb11ab e870000000      call    sample!CDerived::Init (00eb1220)

CDerived的析构函数调用Uninit函数

> 41 00eb1202 e849000000      call    sample!CDerived::Uninit (00eb1250)

CBase的析构函数调用Uninit函数

> 12 00eb1062 e849000000      call    sample!CBase::Uninit (00eb10b0)

以CBase的构造函数为例，调用虚函数Init函数，跟指定调用当前类的函数的实现是一样的。

``` C++
CBase::CBase() {
  CBase::Init();
}
```

这里是编译器对构造函数和析构函数做了特殊处理，但如果通过间接方式调用其他虚函数，就跟普通函数调用虚函数的方式是一样的。

# 如何调用虚函数

在上面的代码中，CBase::Init函数调用了另一个虚函数Fun。而调用虚函数，主要是要找到虚函数表指针，以及虚函数在虚函数表中的偏移，然后计算出虚函数的地址。主要步骤如下：

* 获取到this指针的地址。
* 通过this指针得到虚函数表地址，一般this指针就是指向虚函数表地址。
* 通过函数在虚函数表内的偏移量，加上虚函数表地址，计算出函数的地址。
* 通过call命令调用函数

CBase::Init的反汇编代码如下，主要步骤已经注释

> 0:000> uf sample!CBase::Init
sample!CBase::Init :
   27 00eb1110 55              push    ebp
   27 00eb1111 8bec            mov     ebp,esp
   27 00eb1113 51              push    ecx
   27 00eb1114 894dfc          mov     dword ptr [ebp-4],ecx
   // 得到this指针
   28 00eb1117 8b45fc          mov     eax,dword ptr [ebp-4]
   // 得到虚函数表指针
   **28 00eb111a 8b10            mov     edx,dword ptr [eax]**
   // 传递this指针，供函数Fun使用
   28 00eb111c 8b4dfc          mov     ecx,dword ptr [ebp-4]
   // 计算出虚函数Fun的地址
   **28 00eb111f 8b4204          mov     eax,dword ptr [edx+4]**
   // 调用虚函数
   **28 00eb1122 ffd0            call    eax**
   29 00eb1124 8be5            mov     esp,ebp
   29 00eb1126 5d              pop     ebp
   29 00eb1127 c3              ret
> 

通过反汇编代码分析可知，是调用的子类还是父类的虚函数，关键还是要看虚函数表指针指向的是谁的虚函数表。由于构造函数和析构函数，都会把虚函数表指针设置为当前类的虚函数表地址，因此，在构造函数和析构函数中调用的虚函数，都是调用得当前类的函数。

# 小结

构造函数和析构函数，调用其他虚函数时，由于虚函数表指针指向得是当前类的虚函数表，因此，调用得是当前类的函数。而这种实现，容易造成混淆和误解，所以，建议在构造函数和析构函数中应该避免直接或者间接调用其他虚函数。

# 关于作者
微信公众号：程序员bingo
![微信公众号：程序员bingo](https://bingoli.github.io/wechat.jpeg)
Blog: https://bingoli.github.io/
GitHub: https://github.com/bingoli