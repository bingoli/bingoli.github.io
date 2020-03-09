---
title: rust tokio库interval的时间处理的坑
date: 2020-3-5 21:03:11
tags:
- rust
- tokio
---

# 引言

虽然rust代码写得不多，但使用C++的性能分析方法论和工具链，也能分析rust代码的性能。这次，就是通过WPT（Windows Performance Tools）和Instruments分析了rust的第三方库tokio的Interval的一个性能问题。

# 问题的发现

在Win10系统（CPU 12核）休眠恢复之后，发现rust进程的CPU占比超过6%，相当于1个CPU核的70%。于是，赶紧用WPT抓了一个现场。通过WPT分析，发现CPU主要被其中一个线程消耗。

![微信公众号：程序员bingo](https://bingoli.github.io/tokio_interval_windows_cpu_high.png)

# 其他平台的测试验证

由于rust代码是跨平台的SDK，因此，也需要验证下其他平台是否有同样的问题。

## iOS

使用Instruments抓了iOS与Windows类似场景，消耗CPU最多的也是这个线程。

![微信公众号：程序员bingo](https://bingoli.github.io/tokio_interval_ios_cpu_high.png)

## Mac

经过反复测试，未发现Mac有这个问题。

## Android

经过反复测试，未发现Android有这个问题。（后来发现，是因为Android连上电脑是充电模式，进程不会自动挂起，没有进入到可复现这个问题的路径。）

# tokio的Interval的Demo实验

经过多平台的测试验证，发现Windows和iOS有这个问题，但是Mac没有发现这个问题，这就感觉到很迷糊了。通过WPT和Instruments的分析数据，已定位到相关代码，使用tokio的Interval运行一个定时任务，每隔1S执行1次。其中，tokio版本0.1.13。排查了自有代码，没有找到可能引起这个问题的代码。经过现象分析，推测可能跟tokio有关，于是，写了一个简化的Demo进行验证。

## 实验代码说明

创建一个循环定时器任务，每隔1S运行1次。在每次定时器任务触发时，输出以下调试信息：

- 当前任务的序号
- 任务触发的绝对时间
- Instant::now()获取的时间
- 当前任务的到期时间

``` rust
pub fn test_interval() {
    let interval = Duration::from_millis(1000);
    let task = Interval::new_interval(interval)
        .for_each(move |deadline| {
            let count = {
                let mut count = TIME_COUNT.write().unwrap();
                *count += 1;
                *count
            };
            println!("{}, {}, now: {:?}, deadline: {:?}", count, Utc::now().format("%T"), Instant::now(), deadline);
            Ok(())
        })
        .then(|_| {
            Ok(())
        });
    
        MY_RUNTIME.executor().spawn(task);
}
```

## 系统休眠实验

由于Windows平台发现的问题，就是通过系统休眠的方式来发现的，因此，对Windows、Mac、Linux都进行了系统休眠实验。

实验方法：先让系统休眠一段时间（超过2分钟），然后重新激活系统，观察激活后的任务执行次数。如果系统重新激活后1S内，任务执行次数只有1-3次，属于没有问题；如果有大量任务执行，则是有问题的。

### Windows系统休眠实验结果

- 系统开始休眠时间：14:08:33。
- 激活系统时间：14:15:05。
- 系统激活后1S内，执行任务394次，存在问题。

> 1, 14:08:18, now: Instant { t: 1287693.9087257s }, deadline: Instant { t: 1287693.9079829s }
> 2, 14:08:19, now: Instant { t: 1287694.9086322s }, deadline: Instant { t: 1287694.9079829s }
> ***
> 16, 14:08:33, now: Instant { t: 1287708.9085874s }, deadline: Instant { t: 1287708.9079829s }
> 17, 14:15:05, now: Instant { t: 1288101.05249s }, deadline: Instant { t: 1287709.9079829s }
> 18, 14:15:05, now: Instant { t: 1288101.0528517s }, deadline: Instant { t: 1287710.9079829s }
> ***
> 409, 14:15:06, now: Instant { t: 1288101.9091369s }, deadline: Instant { t: 1288101.9079829s }
> 410, 14:15:07, now: Instant { t: 1288102.9145323s }, deadline: Instant { t: 1288102.9079829s }
> 411, 14:15:08, now: Instant { t: 1288103.9128121s }, deadline: Instant { t: 1288103.9079829s }

### Mac系统休眠实验结果

- 系统开始休眠时间：14:10:19。
- 激活系统时间：14:18:39。
- 结束休眠后1S内，执行任务1次，没有问题。

> 1, 14:09:50, now: Instant { t: 306133377041330 }, deadline: Instant { t: 306133373882499 }
> 2, 14:09:51, now: Instant { t: 306134375199583 }, deadline: Instant { t: 306134373882499 }
> ***
> 30, 14:10:19, now: Instant { t: 306162380111871 }, deadline: Instant { t: 306162373882499 }
> ***
> 31, 14:18:39, now: Instant { t: 306164208046139 }, deadline: Instant { t: 306163373882499 }
> 32, 14:18:40, now: Instant { t: 306164378924112 }, deadline: Instant { t: 306164373882499 }
> 33, 14:18:41, now: Instant { t: 306165379182917 }, deadline: Instant { t: 306165373882499 }
> 34, 14:18:42, now: Instant { t: 306166375919484 }, deadline: Instant { t: 306166373882499 }

## Linux系统休眠实验结果

使用的测试系统为ubuntu 18.04。

- 进程开始休眠时间：11:10:11。
- 进程激活时间：11:31:52。
- 进程激活后1S内，执行任务2次，没有问题。

> 1, 11:09:58, now: Instant { tv_sec: 3437, tv_nsec: 200974533 }, deadline: Instant { tv_sec: 3437, tv_nsec: 199134350 }
> ***
> 14, 11:10:11, now: Instant { tv_sec: 3450, tv_nsec: 200868087 }, deadline: Instant { tv_sec: 3450, tv_nsec: 199134350 }
> 15, 11:31:52, now: Instant { tv_sec: 3452, tv_nsec: 388860753 }, deadline: Instant { tv_sec: 3451, tv_nsec: 199134350 }
> 16, 11:31:52, now: Instant { tv_sec: 3452, tv_nsec: 389004878 }, deadline: Instant { tv_sec: 3452, tv_nsec: 199134350 }
> 17, 11:31:53, now: Instant { tv_sec: 3453, tv_nsec: 201303263 }, deadline: Instant { tv_sec: 3453, tv_nsec: 199134350 }
> 18, 11:31:54, now: Instant { tv_sec: 3454, tv_nsec: 201269682 }, deadline: Instant { tv_sec: 3454, tv_nsec: 199134350 }

## 进程挂起实验

通过iOS之前的测试分析，iOS出现问题时，并不是系统休眠，而是进程挂起，于是，对移动端进行了进程挂起的实验。

### iOS进程挂起实验结果

先运行进程，然后把进程切换到后台，等待一段时间后，把进程切换到前台，观察运行结果。

- 进程开始休眠时间：09:40:07。
- 进程激活时间：09:41:56。
- 进程激活后1S内，运行了109次任务，存在问题。

> 1, 09:39:10, Instant { t: 8575538386338 }, Instant { t: 8575538224636 }
> ***
> 58, 09:40:07, Instant { t: 8576906237923 }, Instant { t: 8576906224636 }
> ***
> 59, 09:41:56, Instant { t: 8579507571088 }, Instant { t: 8576930224636 }
> 60, 09:41:56, Instant { t: 8579507573627 }, Instant { t: 8576954224636 }
> ***
> 167, 09:41:56, Instant { t: 8579522286057 }, Instant { t: 8579522224636 }
> 168, 09:41:57, Instant { t: 8579546357653 }, Instant { t: 8579546224636 }
> 169, 09:41:58, Instant { t: 8579570335229 }, Instant { t: 8579570224636 }

### Android进程挂起实验结果

在测试Android时，发现连上电脑后，手机处于充电模式，App切换到后台之后，仍然会运行，因此，切到后台的方式不能测试进程挂起。下面就通过打断点的方式，测试进程挂起恢复后的情况。

- 进程开始挂起时间：08:51:01。
- 进程激活时间：08:55:25。
- 进程激活后1S内，运行任务264次，存在问题。

> 1, 08:50:48, now: Instant { tv_sec: 16422, tv_nsec: 449018405 }, deadline: Instant { tv_sec: 16422, tv_nsec: 446896405 }
> ***
> 14, 08:51:01, now: Instant { tv_sec: 16435, tv_nsec: 449331405 }, deadline: Instant { tv_sec: 16435, tv_nsec: 446896405 }
> ***
> 15, 08:55:25, now: Instant { tv_sec: 16698, tv_nsec: 727410405 }, deadline: Instant { tv_sec: 16436, tv_nsec: 446896405 }
> ***
> 278, 08:55:25, now: Instant { tv_sec: 16699, tv_nsec: 449296405 }, deadline: Instant { tv_sec: 16699, tv_nsec: 446896405 }
> 279, 08:55:26, now: Instant { tv_sec: 16700, tv_nsec: 450704405 }, deadline: Instant { tv_sec: 16700, tv_nsec: 446896405 }
> 280, 08:55:27, now: Instant { tv_sec: 16701, tv_nsec: 449870405 }, deadline: Instant { tv_sec: 16701, tv_nsec: 446896405 }
> 281, 08:55:28, now: Instant { tv_sec: 16702, tv_nsec: 449879405 }, deadline: Instant { tv_sec: 16702, tv_nsec: 446896405 }

## 小结

通过以上的实验结果证明，测试代码能复现相同场景问题，因此，这个问题跟tokio的Interval有关。

* 系统休眠恢复后，Windows有此问题，Mac和Linux无此问题。
* 进程挂起恢复后，iOS和Android都有此问题。同时，经过推测，Windows、Mac、Linux的进程挂起，也会有此问题。

# tokio源码分析

下面就研究tokio的相关源码来分析问题原因。

## Interval的实现原理

Interval处理定时器到期的函数为poll，主要作用是设置下一次定时器到期的时间。这里是通过本次到期时间加上定时器时间间隔，作为下一次到期的时间。这里使用的是相对时间，因此，每一次循环都需要执行。

``` rust
impl Stream for Interval {
    fn poll(&mut self) -> Poll<Option<Self::Item>, Self::Error> {
        let _ = try_ready!(self.delay.poll());
        // 这里使用的当前时间是上次定时器到期的时间
        let now = self.delay.deadline();
        // 然后用上次到期时间加上时间间隔，作为下一次触发定时器的时间
        self.delay.reset(now + self.duration);

        Ok(Some(now).into())
    }
}
```

定时器的等待时间和唤醒的处理，是由Timer类的park和unpark函数实现。park是处理定时器等待的函数，通过当前实时时间和下一次定时器到期时间对比，判断定时器是否到期。如果定时器未到期，则会进入park_timeout进行剩余时间的等待。

``` rust
impl<T, N> Park for Timer<T, N>
where
    T: Park,
    N: Now,
{
    type Unpark = T::Unpark;
    type Error = T::Error;

    fn unpark(&self) -> Self::Unpark {
        self.park.unpark()
    }

    fn park(&mut self) -> Result<(), Self::Error> {
        self.process_queue();

        match self.wheel.poll_at() {
            Some(when) => {
                // 获取当前的实时时间,调用的是Instant::now()
                let now = self.now.now();
                // 本次定时器到期的时间
                let deadline = self.expiration_instant(when);
                // 如果定时器未到期，会进行等待
                if deadline > now {
                    self.park.park_timeout(deadline - now)?;
                } else {
                    self.park.park_timeout(Duration::from_secs(0))?;
                }
            }
            None => {
                self.park.park()?;
            }
        }

        self.process();

        Ok(())
    }
}
```

通过上面几个函数的逻辑判断，Timer的等待和唤醒的处理使用的是实时时间，而Interval的定时器到期处理使用了相对时间。系统休眠或者进程挂起会导致相对时间暂停，休眠恢复后，就可能导致休眠期间的任务累加到休眠恢复后执行，从而导致短期内CPU消耗高。这个逻辑推断，正好验证了前面在Windows平台中发现的问题。但是，为什么Mac平台下没有和Windows一样的问题呢？这个跟Instant的实现原理有关。

## Instant的实现原理

Timer的park函数获取实时时间是调用的Instant::now()函数，这个函数是获取了系统当前时间，通过查看源码和测试发现，各平台的Instant::now()的实现原理不一样。

### Instant::now()的Windows相关源码

Windows系统中，Instant::now()调用的是QueryPerformanceCounter函数。经过测试，这个函数的时间计算包括系统休眠时间。

``` rust
impl Instant {
    pub fn now() -> Instant {
        perf_counter::PerformanceCounterInstant::now().into()
    }
}
mod perf_counter {
    impl PerformanceCounterInstant {
        pub fn now() -> Self {
            Self { ts: query() }
        }
    }
    fn query() -> c::LARGE_INTEGER {
        let mut qpc_value: c::LARGE_INTEGER = 0;
        cvt(unsafe { c::QueryPerformanceCounter(&mut qpc_value) }).unwrap();
        qpc_value
    }
}
```

### Instant::now()的Mac和iOS相关源码

Mac和iOS系统中，Instant::now()调用的是mach_absolute_time函数。查看苹果的开发者文档得知，mach_absolute_time的时间不包括系统休眠的时间。

``` rust
impl Instant {
    pub fn now() -> Instant {
        extern "C" {
            fn mach_absolute_time() -> u64;
        }
        Instant { t: unsafe { mach_absolute_time() } }
    }
}
```

### mach_absolute_time在苹果开发者文档中的说明如下：

> mach_absolute_time
> Returns current value of a clock that increments monotonically in tick units (starting at an arbitrary point), this clock does not increment while the system is asleep.

### Instant::now()的Android和Linux相关源码

Android和Linux系统中，Instant::now()调用的是clock_gettime函数。查看相关资料得知，clock_gettime的CLOCK_MONOTONIC的时间不包括系统休眠的时间。

``` rust
impl Instant {
    pub fn now() -> Instant {
        Instant { t: now(libc::CLOCK_MONOTONIC) }
    }
}

fn now(clock: clock_t) -> Timespec {
    let mut t = Timespec { t: libc::timespec { tv_sec: 0, tv_nsec: 0 } };
    cvt(unsafe { libc::clock_gettime(clock, &mut t.t) }).unwrap();
    t
}
```

在StackOverflow查到的关于clock_gettime的问题回答如下：

> Note that on Linux, CLOCK_MONOTONIC does not measure time spent in suspend, although by the POSIX definition it should. You can use the Linux-specific CLOCK_BOOTTIME for a monotonic clock that keeps running during suspend.
> https://stackoverflow.com/questions/3523442/difference-between-clock-realtime-and-clock-monotonic


### 小结

Windows的实时时间计算会包括系统休眠时间。Mac、iOS、Android、Linux的实时时间计算不包括系统休眠时间。

|  系统            | Instant调用函数                  | 是否包括系统休眠时间 |
|  -------------- | ------------------------------- |------------------ |
| Windows         | QueryPerformanceCounter         | 是                |
| Mac、iOS        | mach_absolute_time              | 否                |
| Android、Linux  | clock_gettime (Monotonic Clock) | 否                |

# 问题处理

通过以上的分析，得出结论：

* Windows的系统休眠恢复后有问题，是因为Windows平台的Instant的时间计算包括系统休眠时间。
* 所有系统的进程挂起都有问题，是因为Instant的时间计算包括进程挂起时间。

要想解决这个问题，要么弃用tokio::timer::Interval，或者修改Interval的代码。

## Interval的修改建议

要解决这个问题，重点是要使用系统时间来计算下一次定时器的到期时间。但是这种修复不一定符合tokio库对Interval的设计，只适合修改之后自己使用。

``` rust
impl Stream for Interval {
    fn poll(&mut self) -> Poll<Option<Self::Item>, Self::Error> {
        let _ = try_ready!(self.delay.poll());

        // let now = self.delay.deadline();
        let now = Instant::now();

        self.delay.reset(now + self.duration);

        Ok(Some(now).into())
    }
}
```

# 关于作者
微信公众号：程序员bingo
![微信公众号：程序员bingo](https://bingoli.github.io/wechat.jpeg)
Blog: https://bingoli.github.io/
GitHub: https://github.com/bingoli