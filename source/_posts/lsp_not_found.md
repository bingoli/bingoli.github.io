---
title: LSP异常导致Electron启动不了的问题分析
date: 2020-2-2 20:01:25
tags:
- Electron
- LSP
---

# 问题描述

接到用户反馈，基于electron的应用都无法启动，包括VSCode，点击快捷方式，没有任何反应。非electron的其他软件正常，如Chrome。

# 问题分析

## 判断进程是否启动

从用户角度感知，完全看不到electron进程的启动。因此，需要先用工具判断下，electron启动进行到了哪一步。使用Process Monitor监控electron启动过程发现，只有主进程的UI线程启动了，其他进程和线程没有启动。

## 通过Windbg捕获到启动异常

既然进程能够启动，就可使用windbg来调试electron。使用Windbg启动electron后，electron会先中断，可通过g命令让软件继续运行。当发生异常时，Windbg也会自动中断，而此时，异常出现了。

## Windbg符号加载

由于在用户机器上操作，因此，需要把electron.exe符号拷贝到用户机器上。要知道该把符号拷贝到哪个目录，就得了解Windbg的符号加载路径，可使用!sym noisy开启加载符号的日志输出，通过日志查看Windbg的加载electron.exe.pdb的路径。

> 0:000> !sym noisy

使用.reload /f electron.exe可立即加载electron的符号。

> 0:000> .reload /f electron.exe
SYMSRV:  BYINDEX: 0xB
         c:\symbols\*http://msdl.microsoft.com/download/symbols
         electron.exe.pdb
         A5C8652F254ECEFA4C4C44205044422E1
SYMSRV:  UNC: c:\symbols\electron.exe.pdb\A5C8652F254ECEFA4C4C44205044422E1\electron.exe.pdb - path not found
SYMSRV:  UNC: c:\symbols\electron.exe.pdb\A5C8652F254ECEFA4C4C44205044422E1\electron.exe.pd_ - path not found
SYMSRV:  UNC: c:\symbols\electron.exe.pdb\A5C8652F254ECEFA4C4C44205044422E1\file.ptr - path not found
SYMSRV:  HTTPGET: /download/symbols/electron.exe.pdb/A5C8652F254ECEFA4C4C44205044422E1/electron.exe.pdb
SYMSRV:  HttpQueryInfo: 80190194 - HTTP_STATUS_NOT_FOUND
SYMSRV:  HTTPGET: /download/symbols/electron.exe.pdb/A5C8652F254ECEFA4C4C44205044422E1/electron.exe.pd_
SYMSRV:  HttpQueryInfo: 80190194 - HTTP_STATUS_NOT_FOUND
SYMSRV:  HTTPGET: /download/symbols/electron.exe.pdb/A5C8652F254ECEFA4C4C44205044422E1/file.ptr
SYMSRV:  HttpQueryInfo: 80190194 - HTTP_STATUS_NOT_FOUND
SYMSRV:  RESULT: 0x80190194
DBGHELP: electron.exe.pdb - file not found
>

通过日志看出，Windbg会去多个路径中搜索electron.exe.pdb，我们只需要把electron.exe.pdb放到其中一个路径即可。把electron.exe.pdb放到c:\symbols\electron.exe.pdb\A5C8652F254ECEFA4C4C44205044422E1\目录下，然后，再次加载electron的符号。

> 0:000> .reload /f electron.exe
SYMSRV:  BYINDEX: 0xD
         c:\symbols\*http://msdl.microsoft.com/download/symbols
         electron.exe.pdb
         A5C8652F254ECEFA4C4C44205044422E1
SYMSRV:  PATH: c:\symbols\electron.exe.pdb\A5C8652F254ECEFA4C4C44205044422E1\electron.exe.pdb
SYMSRV:  RESULT: 0x00000000
DBGHELP: c:\symbols\electron.exe.pdb\A5C8652F254ECEFA4C4C44205044422E1\electron.exe.pdb - mismatched pdb
DBGHELP: electron.exe.pdb - file not found
DBGHELP: Couldn't load mismatched pdb for electron.exe
>

这次符号还是加载失败了，原因是pdb不匹配。pdb之所以不匹配，是以为对编译后的electron.exe进行了部分修改。通过查询.reload的帮助文档，发现通过/i可强制加载guid不匹配的pdb。

> 0:000> .reload /f /i electron.exe
SYMSRV:  BYINDEX: 0x11
         c:\symbols\*http://msdl.microsoft.com/download/symbols
         electron.exe.pdb
         A5C8652F254ECEFA4C4C44205044422E1
SYMSRV:  PATH: c:\symbols\electron.exe.pdb\A5C8652F254ECEFA4C4C44205044422E1\electron.exe.pdb
SYMSRV:  RESULT: 0x00000000
DBGHELP: c:\symbols\electron.exe.pdb\A5C8652F254ECEFA4C4C44205044422E1\electron.exe.pdb - mismatched pdb
DBGHELP: electron.exe.pdb - file not found
DBGHELP: Loaded mismatched pdb for electron.exe
*** WARNING: Unable to verify timestamp for electron.exe
DBGHELP: electron - private symbols & lines 
        c:\symbols\electron.exe.pdb\A5C8652F254ECEFA4C4C44205044422E1\electron.exe.pdb - unmatched
>

## 通过反汇编代码定位socket创建失败

成功加载符号之后，就可通过kv命令查看调用栈的详情了。

> 0:000> kv
 \# ChildEBP RetAddr  Args to Child              
WARNING: Stack unwind information not available. Following frames may be wrong.
00 0042f8f0 04602b95 0000277a 0531269a 00000274 KERNELBASE+0x133e8
01 0042fd10 035564e2 0042fd2c 035414ed 05ec9bc0 electron!uv_winsock_init+0x1a1 (FPO: [0,0,0]) (CONV: cdecl) [D:\Project\electron-gn\src\third_party\electron_node\deps\uv\src\win\winsock.c @ 0] 
02 0042fd18 035414ed 05ec9bc0 00000031 00000045 electron!uv_init+0x32 (FPO: [0,0,4]) (CONV: cdecl) [D:\Project\electron-gn\src\third_party\electron_node\deps\uv\src\win\core.c @ 208] 
03 0042fd2c 03556486 060660e4 035564b0 0042fd44 electron!uv_once+0x2d (FPO: [2,0,0]) (CONV: cdecl) [D:\Project\electron-gn\src\third_party\electron_node\deps\uv\src\win\thread.c @ 73] 
04 0042fd3c 0354c158 0042fd5c 045e36e6 045de150 electron!uv__once_init+0x12 (FPO: [0,0,4]) (CONV: cdecl) [D:\Project\electron-gn\src\third_party\electron_node\deps\uv\src\win\core.c @ 315] 
05 0042fd44 045e36e6 045de150 0042fd5c d83a75b0 electron!uv_hrtime+0x8 (FPO: [0,0,4]) (CONV: cdecl) [D:\Project\electron-gn\src\third_party\electron_node\deps\uv\src\win\util.c @ 454] 
06 0042fd5c 051fcc3e 00000000 00000000 fffde000 electron!_GLOBAL__sub_I_node_perf.cc+0x16 (FPO: [0,0,0]) (CONV: cdecl) [D:\Project\electron-gn\src\third_party\electron_node\src\node_perf.cc @ 0] 
07 0042fd74 051e4d7e 05ec9afc 05ec9c10 d83a7554 electron!_initterm+0x38 (FPO: [Non-Fpo]) (CONV: cdecl) [D:\Project\electron-gn\src\out\Release-x86\minkernel\crts\ucrt\src\appcrt\startup\initterm.cpp @ 16] 
08 0042fdb8 76c0343d fffde000 0042fe04 77389802 electron!__scrt_common_main_seh+0x7c (FPO: [Non-Fpo]) (CONV: cdecl) [d:\agent\_work\6\s\src\vctools\crt\vcstartup\src\startup\exe_common.inl @ 256] 
09 0042fdc4 77389802 fffde000 763d9ed3 00000000 kernel32+0x1343d
0a 0042fe04 773897d5 051e4e80 fffde000 00000000 ntdll+0x39802
0b 0042fe1c 00000000 051e4e80 fffde000 00000000 ntdll+0x397d5
>

通过调用栈看出，异常发生在winsock.c文件的uv_winsock_init函数，但是没有精确匹配到具体的代码行。由于pdb文件是强制匹配的，也有可能存在pdb和exe的配对存在问题。因此，进行了多种可能性分析和反复推敲论证。过程略过不讲，只讲结论。在调用栈中，重点关注了KERNELBASE函数的两个参数，

- 0000277a
- 0531269a

其中051d269a是指向字符串"socket"的指针。说明这行代码应该是匹配错了。

> 0:000> da 051d269a
051d269a  "socket"

在uv_winsock_init函数内查找会使用"socket"作为函数参数的代码，找到了2行完全一行的代码，如下：

``` C++
uv_fatal_error(WSAGetLastError(), "socket");
```

阅读代码发现，有两段代码的处理逻辑类似，一个是处理ipv4，一个是处理ipv6。下面就以ipv4的处理为分析对象。通过完整的代码分析，出现上面这种情况，是创建socke失败了。

``` C++
  /* Detect non-IFS LSPs */
  dummy = socket(AF_INET, SOCK_STREAM, IPPROTO_IP);

  if (dummy != INVALID_SOCKET) {
    opt_len = (int) sizeof protocol_info;
    if (getsockopt(dummy,
                   SOL_SOCKET,
                   SO_PROTOCOL_INFOW,
                   (char*) &protocol_info,
                   &opt_len) == SOCKET_ERROR)
      uv_fatal_error(WSAGetLastError(), "getsockopt");

    if (!(protocol_info.dwServiceFlags1 & XP1_IFS_HANDLES))
      uv_tcp_non_ifs_lsp_ipv4 = 1;

    if (closesocket(dummy) == SOCKET_ERROR)
      uv_fatal_error(WSAGetLastError(), "closesocket");

  } else if (!error_means_no_support(WSAGetLastError())) {
    /* Any error other than "socket type not supported" is fatal. */
    uv_fatal_error(WSAGetLastError(), "socket");
  }
```

根据函数uv_fatal_error的参数压栈顺序，051d269a是第2个参数，0000277a是第1个参数，也就是WSAGetLastError()的返回值，转换为十进制为10106。

## socket创建失败的原因

创建socket失败的错误代码是10106，查询MSDN得知，10106对应的错误类型是WSAEPROVIDERFAILEDINIT。

> WSAEPROVIDERFAILEDINIT
> 10106
> Service provider failed to initialize.The requested service provider could not be loaded or initialized. This error is returned if either a service provider's DLL could not be loaded (LoadLibrary failed) or the provider's WSPStartup or NSPStartup function failed.
>

出现该错误时，可能是某个dll加载失败了。而加载dll，会有文件IO操作，可通过Process Monitor捕获这次IO操作。于是，打开Process Monitor，重复electron的运行步骤，通过抓取到的IO操作发现，加载netlsp.dll时失败了。

![微信公众号：程序员bingo](https://bingoli.github.io/lsp_dll_not_found.png)

## LSP问题排查

通过调查netlsp.dll发现，这应该是一个LSP文件。LSP（Layered Service Provider）简单来说，就是会对socket的网络操作进行拦截过滤。那就先排查LSP的问题，在命令行窗口中输入netsh winsock show catalog可查看与Winsock有关的设置。通过逐项排查发现，C:\ProgramData\Microsoft\Windows\Provider\目录下的netlsp.dll和netlspx.dll都不存在，这个跟Process Monitor发现的结论一致。初步判断，electron启动失败会跟这两个LSP设置有关，需要把这两个文件有关的LSP项都删除。

> C:\>netsh winsock show catalog
> 
> Winsock 目录提供程序项
> \------------------------------------------------------
> 项类型:                             分层链项
> 描述:                               NAT WLAN RAM over [RSVP TCPv6 服务提供商]
> 提供程序 ID:                        {FC0F29CD-9005-4E5C-80AF-1C4888AB67D8}
> 提供程序路径:                       C:\ProgramData\Microsoft\Windows\Provider\netlspx.dll
> 
> \------------------------------------------------------
> 项类型:                             分层链项(32)
> 描述:                               NAT WLAN RAM over [MSAFD Tcpip [TCP/IPv6]]
> 提供程序 ID:                        {BD284F3E-2E68-4A48-9648-B932F612EA56}
> 提供程序路径:                       C:\ProgramData\Microsoft\Windows\Provider\netlsp.dll
>

## LSP问题修复方法

找到了问题的根源，最后一步就是搜索如何修复LSP问题了。可通过Windows系统命令修复，也可以通过第三方工具修复。

系统命令修复操作如下：

> C:\>netsh winsock reset
> 
> 成功地重置 Winsock 目录。
> 你必须重新启动计算机才能完成重置。

# 问题解决

重置LSP设置之后，electron能够正常运行了，问题解决。通过现有线索判断，这个问题的根源，是第三方软件卸载LSP的时候，只把DLL删除了，但是没有把注册表清理干净。

# 特殊说明

删除LSP也可能会导致整个电脑上不网，因此操作要谨慎，建议找专业人士操作。

# 补充：被pdb误导的一个分析过程

由于pdb匹配错误，所以之前就一直基于pdb匹配来做分析，最后发现了部分参数跟推导过程冲突，才转向去分析调用参数的正确思路。

## 为什么调用栈不能显示正确的代码行数

看下通过kv看到的调用栈，发现匹配的代码行是winsock.c的0。这个是为什么呢？

> 01 0042fd10 035564e2 0042fd2c 035414ed 05ec9bc0 Lark!uv_winsock_init+0x1a1 (FPO: [0,0,0]) (CONV: cdecl) [D:\Project\electron-gn\src\third_party\electron_node\deps\uv\src\win\winsock.c @ 0] 

这时，可通过反汇编来精确定位出现问题的代码行。先反汇编uv_winsock_init+0x1a1的代码。

> 0:000> u lark!uv_winsock_init+0x1a1
> lark!uv_winsock_init+0x1a1 [D:\Project\electron-gn\src\third_party\electron_node\deps\uv\src\win\winsock.c @ 0]:
> 044c2b55 ff15ccb1e905    call    dword ptr [lark!_imp__WSAGetLastError (05e9b1cc)]
> 044c2b5b 681f837705      push    offset lark!`string (0577831f)
> 044c2b60 50              push    eax
> 044c2b61 e84ec9f5ff      call    lark!uv_fatal_error (0441f4b4)

call命令调用的是函数，可以跟C++中的代码匹配。反汇编代码里有使用常量字符串，通过da命令查看字符串的值。

> 0:000> da 0577831f
> 0577831f  "getsockopt"

结合函数调用和字符串，可查到能匹配的C++代码是uv_winsock_init函数的这行代码。

``` C++
uv_fatal_error(WSAGetLastError(), "getsockopt");
```

能够查看uv_winsock_init函数的完整代码，看到有两行一样的代码。编译器把两行代码优化成了一段汇编代码，因此，pdb匹配的时候，不知道匹配的是哪行代码，就显示为0。

## 矛盾点：socket的句柄值不对

通过反汇编uv_winsock_init函数的代码，看到上面的代码，就是getsockopt失败了。

- 反汇编代码片段1

> lark!uv_winsock_init+0x88 [D:\Project\electron-gn\src\third_party\electron_node\deps\uv\src\win\winsock.c @ 112]:
>   112 044c2a3c 89c6            mov     esi,eax
>   112 044c2a3e 8d85f0fbffff    lea     eax,[ebp-410h]
>   112 044c2a44 8d8df4fbffff    lea     ecx,[ebp-40Ch]
>   112 044c2a4a c70074020000    mov     dword ptr [eax],274h
>   113 044c2a50 50              push    eax
>   113 044c2a51 51              push    ecx
>   113 044c2a52 6805200000      push    2005h
>   113 044c2a57 68ffff0000      push    0FFFFh
>   113 044c2a5c 56              push    esi
>   113 044c2a5d ff1538b2e905    call    dword ptr [lark!_imp__getsockopt (05e9b238)]
>   113 044c2a63 83f8ff          cmp     eax,0FFFFFFFFh
>   113 044c2a66 0f84e9000000    je      lark!uv_winsock_init+0x1a1 (044c2b55)  Branch

- 反汇编代码片段2

> lark!uv_winsock_init+0x107 [D:\Project\electron-gn\src\third_party\electron_node\deps\uv\src\win\winsock.c @ 135]:
>   135 044c2abb 89c6            mov     esi,eax
>   135 044c2abd 8d85f0fbffff    lea     eax,[ebp-410h]
>   135 044c2ac3 8d8df4fbffff    lea     ecx,[ebp-40Ch]
>   135 044c2ac9 c70074020000    mov     dword ptr [eax],274h
>   136 044c2acf 50              push    eax
>   136 044c2ad0 51              push    ecx
>   136 044c2ad1 6805200000      push    2005h
>   136 044c2ad6 68ffff0000      push    0FFFFh
>   136 044c2adb 56              push    esi
>   136 044c2adc ff1538b2e905    call    dword ptr [lark!_imp__getsockopt (05e9b238)]
>   136 044c2ae2 83f8ff          cmp     eax,0FFFFFFFFh
>   136 044c2ae5 746e            je      lark!uv_winsock_init+0x1a1 (044c2b55)  Branch

匹配单行代码时，是能够匹配到精确的代码行，上面的反代码片段分别是winsock.c的112和135行，都是getsockopt的函数调用。通过阅读代码发现，两段代码的处理逻辑部分类似，只是一个是处理ipv4，一个是处理ipv6。下面就以ipv4的处理为分析对象。

``` C++
  /* Detect non-IFS LSPs */
  dummy = socket(AF_INET, SOCK_STREAM, IPPROTO_IP);

  if (dummy != INVALID_SOCKET) {
    opt_len = (int) sizeof protocol_info;
    if (getsockopt(dummy,
                   SOL_SOCKET,
                   SO_PROTOCOL_INFOW,
                   (char*) &protocol_info,
                   &opt_len) == SOCKET_ERROR)
      uv_fatal_error(WSAGetLastError(), "getsockopt");

    if (!(protocol_info.dwServiceFlags1 & XP1_IFS_HANDLES))
      uv_tcp_non_ifs_lsp_ipv4 = 1;

    if (closesocket(dummy) == SOCKET_ERROR)
      uv_fatal_error(WSAGetLastError(), "closesocket");

  } else if (!error_means_no_support(WSAGetLastError())) {
    /* Any error other than "socket type not supported" is fatal. */
    uv_fatal_error(WSAGetLastError(), "socket");
  }
```

通过分析这段代码，如果能运行到uv_fatal_error，说明getsockopt失败。但之前的socket应该是创建成功了。先通过.frame切换到uv_winsock_init函数的调用栈中。

> 0:000> .frame 01
> 01 0034f734 03416512 lark!uv_winsock_init+0x1a1 [D:\Project\electron-gn\src\third_party\electron_node\deps\uv\src\win\winsock.c @ 0] 

通过dv查看局部变量的值。

> 0:000> dv
>        wsa_data = struct WSAData
>   protocol_info = struct _WSAPROTOCOL_INFOW
>         opt_len = 0n628
>         errorno = <value unavailable>
>           dummy = 0x277a

dummy是socket的句柄，可通过!handle查看返回的句柄是否正常。

> 0:000> !handle 0x277a
> Handle 0000277a
>   Type                 <Error retrieving type>

这时候，出现了一个矛盾点，因为通过查询MSDN得知，socket()返回的值是能是有效的句柄或者是INVALID_SOCKET（-1）。通过查询附近的栈内容，也没有查到一个有效的socket句柄。

> 0:000> k
>  \# ChildEBP RetAddr  
> 00 0042f8f0 04602b95 KERNELBASE+0x133e8
> 
> 0:000> dd 0042f8f0
> 0042f8f0  0042fd10 04602b95 0000277a 0531269a
> 0042f900  00000274 00020066 00000000 00000000
> 0042f910  00000000 00000008 e70f1aa0 11cfab8b
> 0042f920  8000a38c 92a1485f 000003e9 00000001
> 0042f930  00000000 00000000 00000000 00000000
> 0042f940  00000000 00000000 00000000 00000002
> 0042f950  00000002 00000010 00000010 00000001
> 0042f960  00000006 00000000 00000000 00000000

查询了MSDN，发现dummy的值0x277a（10106）跟WSAGetLastError()的可能返回值相同，此时，开始怀疑pdb匹配有比较大的出入，需要重新分析。

# 关于作者
微信公众号：程序员bingo
![微信公众号：程序员bingo](https://bingoli.github.io/wechat.jpeg)
Blog: https://bingoli.github.io/
GitHub: https://github.com/bingoli