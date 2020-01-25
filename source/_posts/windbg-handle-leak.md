---
title: 使用Windbg分析C++的句柄泄露问题
date: 2020-1-9 22:34:39
tags:
- Windbg
- 句柄泄露
- C++
---

# 引言

句柄泄露是因为创建句柄之后，没有及时销毁句柄。因此，排查句柄泄露的原因，重点需要找到是哪些句柄发生了泄露，以及创建这些句柄的代码。本文将通过一个例子来演示使用Windbg分析句柄泄露的方法。

# 检测句柄泄露
检测进程使用句柄数量的工具有很多，如果看到句柄的数量在持续增加，那就是发生了句柄泄露。下面将对进程id为20298(0x51c0)的进程进行分析。

### 任务管理器

句柄列默认是隐藏的，需要通过“选择列”配置把句柄列显示。

![微信公众号：程序员bingo](https://bingoli.github.io/windbg-handle-leak-task-manager.png)

### Process Explorer
Process Explorer查看进程详细信息，在Performance标签的内容中，包括了进程的句柄数量统计。

![微信公众号：程序员bingo](https://bingoli.github.io/windbg-handle-leak-process-explorer.png)

### Windbg

需要先通过主菜单绑定进程，然后通过!handle命令查看当前进程的已使用的句柄数量

> 0:001> **!handle**
> 135 Handles
> Type           	Count
> None           	6
> Event          	8
> File           	7
> Directory      	3
> Key            	4
> Thread         	92
> IoCompletion   	3
> TpWorkerFactory	3
> ALPC Port      	1
> WaitCompletionPacket	8

通过上面数据可以看出，当前已打开的句柄的主要为Thread。而通过~*查询发现，当前进程只有2个线程，打开这么多线程句柄是异常的。

> 0:001> **~\***
>    0  Id: 51c0.85a4 Suspend: 1 Teb: 00626000 Unfrozen
>       Start: handle_leak!ILT+825(_mainCRTStartup) (00a1133e)
>       Priority: 0  Priority class: 32  Affinity: fff
> .  1  Id: 51c0.3ad8 Suspend: 1 Teb: 0063b000 Unfrozen
>       Start: ntdll!DbgUiRemoteBreakin (7775abe0)
>       Priority: 0  Priority class: 32  Affinity: fff

# 使用Windbg分析Thread句柄泄露的原因

!htrace命令可以跟踪创建和销毁句柄的调用栈，通过调用栈可以判断句柄泄露的原因。首先，通过!htrace -enable打开功能，并且获取当前所有的句柄的快照。然后通过g命令继续运行程序。

> 0:001> **!htrace -enable**
> Handle tracing enabled.
> Handle tracing information snapshot successfully taken.

> 0:001> **g**

运行一段时间后，通过ATL+DEL快捷键暂停程序运行。通过!htrace -diff命令可查看创建快照之后的所有句柄操作。

> 0:001> **!htrace -diff**
> Handle tracing information snapshot successfully taken.
> 0x17 new stack traces since the previous snapshot.
> Ignoring handles that were already closed...
> Outstanding handles opened since the previous snapshot:
> \--------------------------------------
> \*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*
> \--------------------------------------
> Handle = 0x00000244 - OPEN
> Thread ID = 0x000085a4, Process ID = 0x000051c0
> 
> 0x5e1c8322: +0x5e1c8322
> 0x5e1c7c83: +0x5e1c7c83
> 0x5ddd2d15: +0x5ddd2d15
> 0x8fcbe5d4: +0x8fcbe5d4
> 0x8f615ae4: +0x8f615ae4
> 0x8f617123: +0x8f617123
> 0x776a1783: +0x776a1783
> 0x776a1199: +0x776a1199
> 0x8f61c77a: +0x8f61c77a
> 0x8f61c637: +0x8f61c637
> 0x8fcf3fb3: +0x8fcf3fb3
> 0x8fce1db5: +0x8fce1db5
> 0x8fc91853: +0x8fc91853
> 0x8fc917fe: +0x8fc917fe
> 0x7772300c: ntdll!NtOpenThread+0x0000000c
> 0x7570f208: KERNELBASE!OpenThread+0x00000048
> \--------------------------------------
> Displayed 0x17 stack traces for outstanding handles opened since the previous snapshot.

从上面的数据显示，在这段时间内，有23(0x17)个句柄发生了泄露。也可以通过!handle验证一下现有句柄的差值，Thread句柄数量已经从92增长到了115个，差值为23个。

> 0:001> **!handle**
> 158 Handles
> Type           	Count
> None           	6
> Event          	8
> File           	7
> Directory      	3
> Key            	4
> Thread         	115
> IoCompletion   	3
> TpWorkerFactory	3
> ALPC Port      	1
> WaitCompletionPacket	8

!handle命令也可查看单个句柄的详情，里面有创建句柄的线程信息，在分析多线程程序时会用得着。

> 0:001> **!handle 0x00000244 0xf**
> Handle 244
>   Type         	Thread
>   Attributes   	0
>   GrantedAccess	0x1fffff:
>          Delete,ReadControl,WriteDac,WriteOwner,Synch
>          Terminate,Suspend,Alert,GetContext,SetContext,SetInfo,QueryInfo,SetToken,Impersonate,DirectImpersonate
>   HandleCount  	131
>   PointerCount 	130760
>   Name         	&lt;none&gt;
>   Object Specific Information
>     Thread Id   51c0.85a4
>     Priority    10
>     Base Priority 0
>     Start Address a1133e handle_leak!ILT+825(_mainCRTStartup)

现在的问题是，通过栈信息，只能看到KERNELBASE!OpenThread，看不到更上层的调用栈。由于本次句柄泄露是可以重现的，因此，可通过打断点的方式来查找句柄泄露的原因。在ntdll!NtOpenThread+0x0000000c中打上断点之后，继续运行程序，等待断点命中。断点命中之后，可通过k命令查看详细的调用栈。

> 0:001> **bp ntdll!NtOpenThread+0x0000000c**

> 0:001> **g**

> Breakpoint 0 hit
> eax=00000000 ebx=00623000 ecx=93e70000 edx=00000000 esi=008ff5c0 edi=008ff698
> eip=7772300c esp=008ff574 ebp=008ff5ac iopl=0         nv up ei pl nz na pe nc
> cs=0023  ss=002b  ds=002b  es=002b  fs=0053  gs=002b             efl=00000206
> ntdll!NtOpenThread+0xc:
> 7772300c c21000          ret     10h

> 0:000> **k**
>  \# ChildEBP RetAddr  
> 00 008ff570 7570f208 ntdll!NtOpenThread+0xc
> WARNING: Stack unwind information not available. Following frames may be wrong.
> 01 008ff5ac 00a11767 KERNELBASE!OpenThread+0x48
> 02 008ff698 00a118a6 handle_leak!OpenThreadFun+0x47 [D:\github\samples\windows\handle_leak\main.cpp @ 5] 
> 03 008ff76c 00a12013 handle_leak!main+0x36 [D:\github\samples\windows\handle_leak\main.cpp @ 14] 
> 04 008ff78c 00a11e67 handle_leak!invoke_main+0x33
> 05 008ff7e8 00a11cfd handle_leak!\__scrt_common_main_seh+0x157
> 06 008ff7f0 00a12098 handle_leak!\__scrt_common_main+0xd
> 07 008ff7f8 77176359 handle_leak!mainCRTStartup+0x8
> 08 008ff808 77717b74 KERNEL32!BaseThreadInitThunk+0x19
> 09 008ff864 77717b44 ntdll!__RtlUserThreadStart+0x2f
> 0a 008ff874 00000000 ntdll!_RtlUserThreadStart+0x1b

有了详细栈信息，就知道发生句柄泄露的上层调用函数为handle_leak!OpenThreadFun，下面就是通过代码来分析为什么会发生句柄泄露了。

# 代码分析

本文内存泄露的一个简化例子，看下代码，很容易就知道句柄泄露的原因，就是因为手误注释掉了CloseHandle，打开线程句柄之后未关闭句柄。如果是大型工程的话，可能需要更详细大的分析。本文使用的代码如下：

``` C++
#include <iostream>
#include <windows.h>

void OpenThreadFun() {
    HANDLE handle = ::OpenThread(THREAD_ALL_ACCESS, TRUE, ::GetCurrentThreadId());

    ::Sleep(1000);
    // ::CloseHandle(handle);
}

int main() {
    while(1) {
        OpenThreadFun();
    }
}
```

# 关于作者
微信公众号：程序员bingo
![微信公众号：程序员bingo](https://bingoli.github.io/wechat.jpeg)
Blog: https://bingoli.github.io/
GitHub: https://github.com/bingoli