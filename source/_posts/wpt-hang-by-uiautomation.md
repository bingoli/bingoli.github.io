---
title: 使用WPT分析第三方软件导致的软件卡顿问题
date: 2020-1-3 21:11:45
tags:
- WPT
- 性能优化
- UIAutomation
---

# 引言

在Windows系统中，存在一些拥有超级权限的软件（如驱动、服务等），而这些软件一旦没有做好，应用软件也会受到它们的影响，从而造成各种问题。这类问题，可以统一划分为兼容性问题。在处理用户反馈的时候，如果发现可能存在兼容性问题，可以通过卸载一些软件来定位具体是和哪个软件不兼容。如果要定位造成不兼容的原因，则需要使用一些专业的软件来处理，比如WPT(Windows Performance Toolkit)。WPT的功能非常强大，本文通过一个实际案例，介绍WPT定位第三方软件造成的卡顿原因的能力。

# 术语

* WPR（Windows Performance Recorder），用于采集用户的Trace，会生成.etl文件。
* WPA（Windows Performance Analyzer），用于分析采集到的.etl文件。
* WPT（Windows Performace Toolkit），采集和分析用户Trace的工具集，包括WPR和WPA。
* im.exe，指当前使用的聊天软件的主进程。
* srv.exe，指造成im.exe卡顿的第三方进程。

# 问题现象

在使用聊天软件的时候，出现了断断续续的卡顿，主要表现为：打字的时候，能明显的感觉到顿了一下，打字上屏的速度比较慢，删除已输入的字时，也是比较慢；稍微打字快一点，就能感觉字是一蹦一蹦的输入。

由于这种现象是持续性，且只有个别电脑出现，很大概率跟第三方软件有关。要想知道究竟是谁在捣乱，就得上WPT这种专业的抓现场工具。

# 分析过程

通过WPR，已经抓到卡顿过程的Trace信息，只需要使用WPA来分析了。WPA打开etl文件之后，需要先把Symbols加载。操作路径为勾选主菜单Trace下面的子菜单Load Symbols。

## 分析UI线程的调用栈

由于是聊天软件出现了卡顿，这种情况一般是UI线程被短暂的阻塞了，因此，可通过WPR的火焰图来重点观察im.exe的UI线程的情况。通过下图也可以看出，UI线程大部分的调用都与UIAutomationCore.dll有关。这显然跟预期是不符合的，需要看看UIAutomationCore.dll到底是干什么的。

![im.exe的UI线程调用栈](https://bingoli.github.io/wpt-hang-by-uiautomation-flame.png)

## UIAutomation在UI线程调用SendMessageTimeout造成卡顿

使用WPA分析Trace，是可以看到系统函数的各种调用。通过排查发现，UIAutomation调用了SendMessageTimeout，这是一个同步调用，如果对方没有及时回复，就必须等到超时才会返回。更庆幸的是，它调用得不是SendMessage，否则有可能一直卡死在这里。

![SendMessageTimeout调用栈](https://bingoli.github.io/wpt-hang-by-uiautomation-stack.png)

## 查查UIAutomation的底细

通过查询资料了解到，UIAutomationCore是Windows平台下UI自动化测试的核心模块，调用方和提供方都需要使用。因此，只要查到也在调用UIAutomationCore.dll的进程，就能查到导致聊天软件卡顿的元凶。

![UIAutomation架构](https://bingoli.github.io/wpt-hang-by-uiautomation-core.dll.png)

## 通过模块查找进程 

WPA分析问题时，列表里有很多选项可以选择，也可以对选项进行排序，并且，WPA会对放在前面的列进行优先聚合。这次，我们要做的时，过滤出来与UIAutomationCore.dll有关的所有进程。通过下面的数据可以看出，只有两个进程与UIAutomationCore.dll有关，其中一个是im.exe（正在使用的聊天软件的主进程），另一个是srv.exe(某壁纸软件的进程)。于是，捣乱嫌疑进程找到了。

![通过模块查找进程](https://bingoli.github.io/wpt-hang-by-uiautomation-module.png)

## 分析srv.exe的调用栈

找到了可疑进程，那就对可疑进程进行调用栈展开，看看它都在干什么。从UIAutomationCore.dll!UiaNodeTraverser::Traverse能看出来，它是在遍历自动化测试节点，跟im.exe的UI线程调用栈有很大的关联性。

![srv.exe的调用栈](https://bingoli.github.io/wpt-hang-by-uiautomation-target-exe.png)

## 进一步的证据

两个软件之间，到底有多大的关联性，我们还可以看一看它们的CPU调用占比。把这两个进程的的CPU调用放到一起比较，虽然占比不一样，但是节奏非常一致，简直就跟军训时走正步一样整齐。

![CPU调用](https://bingoli.github.io/wpt-hang-by-uiautomation-compare.png)

## 验证

啥也别说了，都知道是你了，那就查查这个进程是谁家的。根据进程的启动路径，知道了是某壁纸的进程。于是，把该壁纸卸载了，聊天软件的卡顿问题就消除了。

# 后记

* 自动化测试接口也叫无障碍接口，可通过读屏软件给视觉障碍的人提供帮助。但是，也有人用这个接口来做其他一些不太好的事，更关键的是，还没有用好，给用户使用其他软件造成了困扰。
* WPT通过抓Trace，可用来分析整个系统里的软件的各种行为。当然，这也可能会涉及到个人隐私的泄露，也要合理的使用。

# 参考资料
* Windows Performance Toolkit : https://docs.microsoft.com/en-us/windows-hardware/test/wpt/

# 关于作者
微信公众号：程序员bingo
![微信公众号：程序员bingo](https://bingoli.github.io/wechat.jpeg)
Blog: https://bingoli.github.io/
GitHub: https://github.com/bingoli