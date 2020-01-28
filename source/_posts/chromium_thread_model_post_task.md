---
title: Chromium的无锁线程模型示例之PostTask
date: 2020-1-27 20:11:34
tags:
- chromium
- Electron
- 线程
---

# 引言

多线程一直是软件开发中最容易出问题的环节，很多的崩溃、卡死问题都与多线程有关。在常用的线程模型中，一般会使用线程锁保证线程数据安全，但是，在实践中，这种模式很容易造成漏加锁、锁粒度太大、死锁等问题。

要解决这类问题，一种比较好的方式是采用无锁的线程模型，Chromium就是采用了这种线程模型。本文通过Electron基于Chromium线程模型实现的打开文件对话框功能，介绍无锁线程模型的思路。

# 无锁线程模型简介

## 应用层数据不加锁

chromium的无锁线程模型，不是指完全的不使用线程锁，因为底层的Task队列是有加锁的，而是指在应用层使用时，不需要添加线程锁。

## 不同线程不会同时访问数据

无锁线程模型，主要是保证在同一时间，不同线程在同一时间不会同时访问相同的数据。下面的例子要用到的方法，主要是对不同线程访问数据的能力进行隔离。对数据访问能力隔离方式主要有

* 拷贝，在不同线程传递数据时，对数据进行一份拷贝，让两个线程访问的是不同数据。
* 移动，在不同线程传递数据时，使用std::move进行右值转移，让原线程无法访问这个数据。

# 无锁线程模型示例

下面以Electron的dialog.showOpenDialog实现代码为例，说明Chromium的无锁线程模型的使用原理。

## dialog.showOpenDialog的调用

Electron的接口函数dialog.showOpenDialog是一个异步的JavaScript函数，会返回一个promise。showOpenDialog会弹出一个文件选择对话框，用户选择文件之后，把文件路径通过result.filePaths返回。

``` JavaScript
dialog.showOpenDialog(mainWindow, {
  properties: ['openFile']
}).then(result => {
  console.log(result.canceled)
  console.log(result.filePaths)
}).catch(err => {
  console.log(err)
})
```

## Chromium线程几个基本概念

* TaskRunner：每一个线程有一个TaskRunner，主要通过PostTask把任务投放到线程的任务队列，通过线程安全的引用技术管理生命周期，可配合scoped_refptr在不同线程使用。
* PostTask：TaskRunner的一个函数，可向该线程的任务队列中发送一个闭包，闭包会在该线程中执行。

相关代码如下：

``` C++
class BASE_EXPORT TaskRunner
    : public RefCountedThreadSafe<TaskRunner, TaskRunnerTraits> {
 public:
  bool PostTask(const Location& from_here, OnceClosure task);
}
```

## C++代码响应JavaScript调用

在JavaScript调用dialog.showOpenDialog之后，会在UI线程中，调用到C++代码的ShowOpenDialog函数。ShowOpenDialog函数主要是创建一个新的Dialog线程，然后通过PostTask把RunOpenDialogInNewThread函数抛到这个Dialog线程去运行。这个过程中，需要处理所有权的参数有3个，run_state、settings和promise。

* run_state、settings是通过拷贝方式隔离不同线程的访问权。run_state除了保存有Dialog线程的指针外，还有UI线程的TaskRunner，用于后续Dialog线程往UI线程发送回调函数。
* promise是通过std::move进行所有权转移，转移之后，就只有Dialog线程的函数RunOpenDialogInNewThread有权访问，UI线程暂时无权限访问。

相关代码如下：

``` C++
struct RunState {
  base::Thread* dialog_thread;
  scoped_refptr<base::SingleThreadTaskRunner> ui_task_runner;
};

bool CreateDialogThread(RunState* run_state) {
  auto thread =
      std::make_unique<base::Thread>(ELECTRON_PRODUCT_NAME "FileDialogThread");
  thread->init_com_with_mta(false);
  if (!thread->Start())
    return false;

  run_state->dialog_thread = thread.release();
  run_state->ui_task_runner = base::ThreadTaskRunnerHandle::Get();
  return true;
}

void ShowOpenDialog(const DialogSettings& settings,
                    gin_helper::Promise<gin_helper::Dictionary> promise) {
  gin_helper::Dictionary dict = gin::Dictionary::CreateEmpty(promise.isolate());
  RunState run_state;
  if (!CreateDialogThread(&run_state)) {
    dict.Set("canceled", true);
    dict.Set("filePaths", std::vector<base::FilePath>());
    promise.Resolve(dict);
  } else {
    run_state.dialog_thread->task_runner()->PostTask(
        FROM_HERE, base::BindOnce(&RunOpenDialogInNewThread, run_state,
                                  settings, std::move(promise)));
  }
}
```

## Dialog线程执行任务，并把结果返回给UI线程

RunOpenDialogInNewThread函数会在Dialog线程中运行，它通过ShowOpenDialogSync函数获取到选中的文件路径paths，并通过拷贝的方式，返回结果result和paths。

这时，promise的所有权再次通过std::move进行了转移，转移之后，只有UI线程的OnDialogOpened函数有权访问。

相关代码如下：

``` C++
void RunOpenDialogInNewThread(
    const RunState& run_state,
    const DialogSettings& settings,
    gin_helper::Promise<gin_helper::Dictionary> promise) {
  std::vector<base::FilePath> paths;
  bool result = ShowOpenDialogSync(settings, &paths);
  run_state.ui_task_runner->PostTask(
      FROM_HERE,
      base::BindOnce(&OnDialogOpened, std::move(promise), !result, paths));
  run_state.ui_task_runner->DeleteSoon(FROM_HERE, run_state.dialog_thread);
}
```

## UI线程处理返回结果

此时，回到了UI线程，OnDialogOpened函数对promise进行处理后，返回给JavaScript的promise的处理结果，最终会回到JavaScript的promise的then函数。

相关代码如下：

``` C++
void OnDialogOpened(gin_helper::Promise<gin_helper::Dictionary> promise,
                    bool canceled,
                    std::vector<base::FilePath> paths) {
  gin_helper::Dictionary dict = gin::Dictionary::CreateEmpty(promise.isolate());
  dict.Set("canceled", canceled);
  dict.Set("filePaths", paths);
  promise.Resolve(dict);
}
```

## 处理流程

promise的可访问权，是跟着showOpenDialog处理顺序进行变化，执行属性如下：

* 在UI线程运行函数：ShowOpenDialog
* 在Dialog线程运行函数：RunOpenDialogInNewThread
* 在UI线程运行函数：OnDialogOpened

相关流程图如下：

![微信公众号：程序员bingo](https://bingoli.github.io/electron_open_file_dialog_sequence_chart.jpg)

# 小结

上面通过Electron的dialog.showOpenDailog，介绍了Chromium的无锁线程模型的一些使用思路。这个例子是通过拷贝和移动语义来保证不同线程无法同时对同一变量进行访问，从而不需要加锁。如果能够正确使用这种线程模型，是可以消除因为数据锁带来的一些线程同步问题。

# 关于作者
微信公众号：程序员bingo
![微信公众号：程序员bingo](https://bingoli.github.io/wechat.jpeg)
Blog: https://bingoli.github.io/
GitHub: https://github.com/bingoli