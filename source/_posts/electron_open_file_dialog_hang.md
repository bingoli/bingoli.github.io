---
title: Electron打开文件对话框卡死问题分析
date: 2020-1-1 08:51:23
tags:
 - Electron
 - Chromium
 - 卡死
 - WPT
 - Windbg
---

# 摘要
- 分析卡死问题时，可结合WPT和Windbg一起分析。
- Electron弹出的打开文件对话框存在必现的卡死场景，原因是弹窗线程COM反初始化卡死，而主线程在同步等待弹窗线程销毁。
- Electron的部分代码还没及时跟上Chromium的改动，分析Electron的问题时，可多跟最新版Chrome进行对比。

# 问题现象
测试同学在基于Electron Windows的App上测试发送图片时，出现了整个界面卡死。
# 现场信息分析
### 现场信息捕获
在测试同学测试时，出现了卡死，保留有现场。分别用以下工具抓取相关信息。
- Windows Performance Toolkit，抓取了系统的Trace信息。
- Process Explorer，抓取了主进程的Dump。

### 使用WPT分析Trace
### UI Delays分析
通过WPA的UI Delays可以捕获到UI卡死相关信息。
- MsgCheck Delay为消息队列被阻塞
- Input Delay为输入队列被阻塞
从下面的数据可看出，主线程和文件弹窗线程都被卡死了，且都被卡主了295s，因此，这两个线程之前的卡死可能存在关联性。
![ui_delays](https://bingoli.github.io/electron_open_file_dialog_hang_ui_delays.png)

### CPU占用分析
在Trace的CPU占用统计中，没有捕获到这两个线程的相关信息，原因是这两个线程已经被挂起了，CPU消耗很少。只能通过其他方法对问题进一步分析。
![WPT捕获的线程](https://bingoli.github.io/electron_open_file_dialog_hang_wpt_threads.png)

### 结论
根据已有线索进行推论：有2个线程被卡死了，可能存在关联。
- 主线程，id 6028, 0x178c
- ElectronFileDialogThread线程，id 10076，0x275c

## 使用Windbg分析Dump
### Dump详情
相关操作
- 打开Dump，加载符号
- 查看卡死线程调用栈信息

> ||0:000> .symfix C:\Symbols

> ||0:000> .reload

> ||0:000> !analyze -v
> EXCEPTION_RECORD:  (.exr -1)ExceptionAddress: 00000000
>    ExceptionCode: 80000003 (Break instruction exception)
>   ExceptionFlags: 00000000
> NumberParameters: 0
> FAULTING_THREAD:  0000178c
> PROCESS_NAME:  Electron.exe

> ||0:000> k
 \# ChildEBP RetAddr  
00 095bf480 7614e2c9 ntdll!NtWaitForSingleObject+0xc
WARNING: Stack unwind information not available. Following frames may be wrong.
01 095bf4f4 7614e222 KERNELBASE!WaitForSingleObjectEx+0x99
02 095bf508 01bcead8 KERNELBASE!WaitForSingleObject+0x12
03 095bf558 01bd2cbc Electron!base::PlatformThread::Join+0x78
04 095bf570 01be559b Electron!base::Thread::~Thread+0x2c
05 095bf57c 00348551 Electron!base::Thread::~Thread+0xb
06 095bf588 006bfe4c Electron!base::DeleteHelper&lt;net::SerialWorker&gt;DoDelete+0x11
07 095bf594 01b6783b Electron!base::internal::Invoker&lt;base::internal::BindStatev&lt;oid (*)(void *),unsigned char *&gt;,void ()>::RunOnce+0xc
08 095bf5d8 01b87903 Electron!base::debug::TaskAnnotator::RunTask+0xab
09 095bf640 01b87b0d Electron!base::MessageLoopImpl::RunTask+0xc3
0a 095bf660 01b87d5a Electron!base::MessageLoopImpl::DeferOrRunPendingTask+0x4d
0b 095bf728 01b88b38 Electron!base::MessageLoopImpl::DoWork+0xca
0c 095bf760 01b88541 Electron!base::MessagePumpForUI::DoRunLoop+0x78
0d 095bf780 01b8775f Electron!base::MessagePumpWin::Run+0x41
0e 095bf790 01ba1e03 Electron!base::MessageLoopImpl::Run+0x1f

根据上面的信息可得出，当前卡死的线程id为线程0x178c，即主线程。调用栈信息表明，主线程之所以卡死了，是因为在等待另一个线程结束。要知道是在等待哪个线程，就需要拿到该线程的进一步信息。切换到对应的调用函数，查看相关变量的值即可获取到线程信息，即Thread变量的值。
> ||0:0:000> .frame 05
> 05 095bf57c 00348551 Electron!base::Thread::~Thread+0xb

> ||0:0:000> dx this
> this                 : 0x1a67dea0 [Type: base::Thread \*]
>     [+0x004] com_status_      : STA (1) [Type: base::Thread::ComStatus]
>     [+0x008] joinable_        : true [Type: bool]
>     [+0x009] stopping_        : true [Type: bool]
>     [+0x00a] running_         : false [Type: bool]
>     [+0x00c] running_lock_    [Type: base::Lock]
>     [+0x010] thread_          [Type: base::PlatformThreadHandle]
>     [+0x014] thread_lock_     [Type: base::Lock]
>     [+0x018] id_              : 0x275c [Type: unsigned long]
>     [+0x01c] id_event_        [Type: base::WaitableEvent]
>     [+0x024] message_loop_    : 0x111ad860 [Type: base::MessageLoop \*]
>     [+0x028] run_loop_        : 0x2675f894 [Type: base::RunLoop \*]
>     [+0x02c] using_external_message_loop_ : false [Type: bool]
>     [+0x030] timer_slack_     : TIMER_SLACK_NONE (0) [Type: base::TimerSlack]
>     [+0x034] name_            : "ElectronFileDialogThread" [Type: std::basic_stringc&lt;har,std::char_traitsc&lt;har&gt;,std::allocator&lt;char&gt; &gt;]
>     [+0x04c] start_event_     [Type: base::WaitableEvent]
>     [+0x054] owning_sequence_checker_ [Type: base::SequenceChecker]

以上信息说明，主线程在等待的线程名为ElectronFileDialogThread，id为0x275c。这些信息，跟WPT获取的信息正好互相印证。有了线程id，就可以切换到该线程去查看调用栈信息。

> ||0:0:000> \~\~[0x275c]s
> eax=00000000 ebx=00000004 ecx=00000000 edx=00000000 esi=00000000 edi=0965a000
> eip=7781708c esp=2675e8f4 ebp=2675e968 iopl=0         nv up ei pl nz na po nc
> cs=0023  ss=002b  ds=002b  es=002b  fs=0053  gs=002b             efl=00200202
> win32u!NtUserMsgWaitForMultipleObjectsEx+0xc:

> ||0:0:072> k
> \#  ChildEBP RetAddr  
> 00 2675e8f0 7526dfd9 win32u!NtUserMsgWaitForMultipleObjectsEx+0xc
> 01 2675e968 7526df0d user32!RealMsgWaitForMultipleObjectsEx+0x79
> 02 2675e98c 76d388e6 user32!MsgWaitForMultipleObjectsEx+0x4d
> 03 2675ea0c 76d38735 combase!CCliModalLoop::BlockFn+0x166
> 04 2675ea54 76d3ac8c combase!ModalLoop+0xbe
> 05 2675ea6c 76d32d90 combase!ClassicSTAThreadDispatchCrossApartmentCall+0x4c
> 06 (Inline) -------- combase!CSyncClientCall::SwitchAptAndDispatchCall+0x3f5
> 07 2675eba0 76d39f6a combase!CSyncClientCall::SendReceive2+0x4d0
> 08 (Inline) -------- combase!SyncClientCallRetryContext::SendReceiveWithRetry+0x29
> 09 (Inline) -------- combase!CSyncClientCall::SendReceiveInRetryContext+0x29
> 0a 2675ec6c 76d32446 combase!ClassicSTAThreadSendReceive+0x1ea
> 0b 2675ee4c 76d5eb84 combase!CSyncClientCall::SendReceive+0x296
> 0c (Inline) -------- combase!CClientChannel::SendReceive+0x75
> 0d 2675ee74 75505fe5 combase!NdrExtpProxySendReceive+0xc4
> 0e 2675f364 76d8de30 rpcrt4!NdrClientCall2+0x5f5
> 0f 2675f384 76d85dcf combase!ObjectStublessClient+0x70
> 10 2675f394 76d303c6 combase!ObjectStubless+0xf
> 11 (Inline) -------- combase!RemoteReleaseRifRefHelper+0x123
> 12 (Inline) -------- combase!RemoteReleaseRifRef+0x1de
> 13 2675f4d0 76d21fcd combase!CStdMarshal::DisconnectCliIPIDs+0x826
> 14 2675f538 76d7136d combase!CStdMarshal::DisconnectWorker_ReleasesLock+0x5ad
> 15 2675f550 76d71322 combase!CStdMarshal::DisconnectSwitch_ReleasesLock+0x1d
> 16 2675f578 76d71203 combase!CStdMarshal::DisconnectAndReleaseWorker_ReleasesLock+0x34
> 17 2675f594 76cef085 combase!CStdMarshal::DisconnectAndRelease+0x35
> 18 2675f750 76ceeecb combase!COIDTable::ThreadCleanup+0xd3
> 19 (Inline) -------- combase!FinishShutdown::\_\_l2::&lt;lambda_52cd3ea394b0aaaaa4b6e0872859635a>::operator()+0x5
> 1a 2675f788 76cee8ff combase!ObjectMethodExceptionHandlingAction&lt;&lt;lambda_52cd3ea394b0aaaaa4b6e0872859635a> >+0x15
> 1b 2675f7a8 76cee072 combase!FinishShutdown+0x44
> 1c 2675f7d0 76d410fe combase!ApartmentUninitialize+0x79
> 1d 2675f7f0 76d41629 combase!wCoUninitialize+0xf8
> 1e 2675f86c 01c1dfdc combase!CoUninitialize+0xf9
> 1f 2675f87c 01be4601 Electron!base::win::ScopedCOMInitializer::~ScopedCOMInitializer+0x1c
> 20 2675f8bc 01be5cc7 Electron!base::Thread::ThreadMain+0x171
> 21 2675f8e0 76f96359 Electron!base::`anonymous namespace'::ThreadFunc+0xf7
> 22 2675f8f0 778f7b74 kernel32!BaseThreadInitThunk+0x19
> 23 2675f94c 778f7b44 ntdll!__RtlUserThreadStart+0x2f
> 24 2675f95c 00000000 ntdll!_RtlUserThreadStart+0x1b

FileDialog线程卡死，是在CoUninitialize时在等待某些COM接口的断开连接操作。
### 结论
- 主线程被卡死了，是因为在等待ElectronFileDialogThread线程结束。
- ElectronFileDialogThread线程被卡死了，是因为在COM反初始化CoUninitialize时卡死了。

# 代码分析
## Electron相关代码
根据现有线索，其中一个线程名为ElectronFileDialogThread。使用ElectronFileDialogThread在代码中搜索，未搜索到相关信息。通过FileDialogThread，搜索到了相关代码。
``` C++
bool CreateDialogThread(RunState* run_state) {
  auto thread =
      std::make_unique<base::Thread>(ATOM_PRODUCT_NAME "FileDialogThread");
  thread->init_com_with_mta(false);
  if (!thread->Start())
    return false;

  run_state->dialog_thread = thread.release();
  run_state->ui_task_runner = base::ThreadTaskRunnerHandle::Get();
  return true;
}
```
根据代码往前追溯发现，主线程接收到文件选择弹窗时，会先创建一个新线程，然后把弹窗任务抛到新线程去执行。主线程的相关代码如下：
``` C++
void ShowOpenDialog(const DialogSettings& settings,
                    const OpenDialogCallback& callback) {
  RunState run_state;
  if (!CreateDialogThread(&run_state)) {
    callback.Run(false, std::vector<base::FilePath>());
    return;
  }

  run_state.dialog_thread->task_runner()->PostTask(
      FROM_HERE,
      base::Bind(&RunOpenDialogInNewThread, run_state, settings, callback));
}
```
弹窗的具体实现由FileDailog线程完成，相关步骤有
- 弹窗，等待用户交互，用户操作后返回。
- 把用户选择的路径通过回调返回给主线程。
- 通知主线程可以把当前线程删除。
``` C++
void RunOpenDialogInNewThread(const RunState& run_state,
                              const DialogSettings& settings,
                              const OpenDialogCallback& callback) {
  std::vector<base::FilePath> paths;
  bool result = ShowOpenDialog(settings, &paths);
  run_state.ui_task_runner->PostTask(FROM_HERE,
                                     base::Bind(callback, result, paths));
  run_state.ui_task_runner->DeleteSoon(FROM_HERE, run_state.dialog_thread);
}
```
最后一步调用DeleteSoon的作用，就是通知主线程在callback调用结束之后，删除FileDialog线程。
``` C++
  template <class T>
  bool DeleteSoon(const Location& from_here, const T* object) {
    return DeleteOrReleaseSoonInternal(from_here, &DeleteHelper<T>::DoDelete,
                                       object);

template <class T>
class DeleteHelper {
 private:
  static void DoDelete(const void* object) {
    delete static_cast<const T*>(object);
  }
};
```
## FileDialog线程的COM相关代码
FileDialog线程的卡死跟COM有关，因此，需要分析下与COM有关的代码。其中，FileDialog线程采用的COM线程模型为STA。因此，线程启动和结束时，分别会调用ScopedCOMInitializer封装的COM初始化和反初始化CoUninitalize。
``` C++
  void init_com_with_mta(bool use_mta) {
    DCHECK(!delegate_);
    com_status_ = use_mta ? MTA : STA;
  }

  std::unique_ptr<win::ScopedCOMInitializer> com_initializer;
  if (com_status_ != NONE) {
    com_initializer.reset(
        (com_status_ == STA)
            ? new win::ScopedCOMInitializer()
            : new win::ScopedCOMInitializer(win::ScopedCOMInitializer::kMTA));
  }
```
# 修复方案
这里面有2个问题需要解决
- FileDialog线程的销毁卡死问题
- 主线程不应该同步等待其他线程结束
FileDialog线程COM反初始化卡死问题处理
代码分析与查看文档相结合
由于已经知道跟COM有关，主要从以下两个方面入手：
- 排查FileDialog线程中与COM有关的代码，是否存在误用，如资源未释放的问题。
- 通过MSDN查询与COM有关的文档。
该问题是测试同学发现的问题，有一定复现概率，通过之前录得视频发现，测试同学进行了拖动操作，会跟OLE相关。查询MSDN了解到，在执行文件的拖放操作之前，必须要调用Ole初始化OleInitialize，否则可能会有问题。
> Applications that use the following functionality must call OleInitialize before calling any other function in the COM library:
> - Clipboard
> - Drag and Drop
> - Object linking and embedding (OLE)
> - In-place activation

从代码中可以看出，FileDialog线程是没有进行OLE初始化的，而MSDN说明拖动必然依赖OLE初始化，因此，这是一个矛盾点。于是，把相关操作进行试验发现，如果在打开文件弹窗内进行拖动操作，是可以稳定复现这个卡顿的。
使用OLE初始化替换COM初始化
要解决这个问题，只需要在FileDailog线程中，用OLE替代COM的初始化和反初始化。
通过前面的代码可以看出，Thread的变量com_status_==NONE时，是不会进行COM初始化。因此，修复方案如下：
- 删除CreateDialogThread函数的thread->init_com_with_mta(false)操作。
- 在线程的初始化添加OLE的初始化。
为了对原有的代码浸入较小，本方案对base::Thread类进行了继承。对已有代码，只有2行代码的改动，其他都是新增代码。
``` C++
class FileDialogThead : public base::Thread {
public:
  explicit FileDialogThead(const std::string& name) : base::Thread(name) {
  }
protected:
  virtual void Init() {
    OleInitialize(NULL);
  }
  virtual void CleanUp() {
    OleUninitialize();
  }
}

bool CreateDialogThread(RunState* run_state) {
  auto thread =
      std::make_unique<FileDialogThead>(ATOM_PRODUCT_NAME "FileDialogThread");
  if (!thread->Start())
    return false;
  // thread->init_com_with_mta(false);
  run_state->dialog_thread = thread.release();
  run_state->ui_task_runner = base::ThreadTaskRunnerHandle::Get();
  return true;
}
```
通过实验发现，这种修复方式，把这个必现的bug变成了一个偶现的bug。这就说明，COM反初始化还有其他的坑，暂时还没有找到原因。本次修复策略，只是缓解了问题，没有彻底解决问题。
## 去除主线程的同步等待
由于找到了必现步骤，经过测试发现，Electron最新版和Chrome 66版本都能稳定复现，但Chrome 79内核版本修复了该问题。因此，只需要参考Chrome的修复方案即可。通过查看代码发现，Chrome主要有以下改动：
- 增加了线程池，FileDialog线程不会实时销毁。
``` C++
scoped_refptr<base::SingleThreadTaskRunner> CreateDialogTaskRunner() {
  return CreateCOMSTATaskRunner(
    {base::ThreadPool(), base::TaskPriority::USER_BLOCKING,
     base::TaskShutdownBehavior::CONTINUE_ON_SHUTDOWN, base::MayBlock()},
    base::SingleThreadTaskRunnerThreadMode::DEDICATED);
}
```
- FileDailog线程已经移出了主进程，跟主进程无关。
# 后记
Electron的部分代码是从较早版本的Chromium拷贝过来，没有及时跟上Chromium的步伐。Electron同步Chromium的最新代码，应该是可以为Electron开源社区做的一个事。
# 参考资料
- [MsgCheck说明](https://social.msdn.microsoft.com/Forums/en-US/633d889d-7d4d-4f11-8422-7213cd8c228b/in-wpa-what-is-delay-typemsgcheck-and-where-can-i-get-more-information-about-it?forum=windbg)
- [Chromium源码](https://cs.chromium.org/chromium/src/ui/shell_dialogs/base_shell_dialog_win.cc?q=base_shell&sq=package:chromium&g=0&l=5)
# 关于作者
微信公众号：程序员bingo
![微信公众号：程序员bingo](https://bingoli.github.io/wechat.jpeg)
Blog: https://bingoli.github.io/
GitHub: https://github.com/bingoli