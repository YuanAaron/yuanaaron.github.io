---
layout: post 
author: oshacker
title: 线程池中KeepAliveTime的作用及实现原理
category: jdk
tags: [java,jdk]
excerpt: idea调试
---

**成员变量断点调试实践**

右键断点，选择按Thread挂起，然后勾选Field access。

![image-20211215101357421](https://cdn.jsdelivr.net/gh/YuanAaron/BlogImage/2021/image-20211215101357421.png)

1. 非核心线程等待任务的超时时间
2. 如果核心线程被设置成会超时的，那么该变量也表示核心线程等待任务的超时时间。

调试代码如下所示

```java
public class ThreadPoolTest {
    public static final ThreadPoolExecutor THREAD_POOL =
            new ThreadPoolExecutor(4,
                    8,
                    1,
                    TimeUnit.SECONDS,
                    new ArrayBlockingQueue<>(10));

    public static void main(String[] args) throws IOException {
        for (int i = 0; i < 18; i++) {
            THREAD_POOL.execute(() -> {
                //每个任务等1s再执行
                LockSupport.parkNanos(TimeUnit.SECONDS.toNanos(1));
                System.out.println(1);
            });
        }
        System.in.read();
    }
}
```

execute的源码解析如下：

```java
/**
* Executes the given task sometime in the future.  The task
* may execute in a new thread or in an existing pooled thread.
* 在将来的某一时刻执行给定的任务。这个任务可能在一个新线程中执行，也可能在一个
* 池中已有的线程中执行。
*
* If the task cannot be submitted for execution, either because this
* executor has been shutdown or because its capacity has been reached,
* the task is handled by the current {@code RejectedExecutionHandler}.
* 如果这个任务不能被提交执行，要么因为execute已经被关闭，要么因为它的容量已满，
* 最后按照当前的拒绝策略处理这个任务。
*
* @param command the task to execute
* @throws RejectedExecutionException at discretion of
*         {@code RejectedExecutionHandler}, if the task
*         cannot be accepted for execution
* @throws NullPointerException if {@code command} is null
*/
public void execute(Runnable command) {
    if (command == null)
        throw new NullPointerException();
    /*
     * Proceed in 3 steps:
     *
     * 1. If fewer than corePoolSize threads are running, try to
     * start a new thread with the given command as its first
     * task.  The call to addWorker atomically checks runState and
     * workerCount, and so prevents false alarms that would add
     * threads when it shouldn't, by returning false.
     * 如果正在运行的线程数小于CorePoolSize(核心线程数)，尝试去开启一个新线程，
     * 并将给定的command作为该线程的第一个任务。
     *
     * 2. If a task can be successfully queued, then we still need
     * to double-check whether we should have added a thread
     * (because existing ones died since last checking) or that
     * the pool shut down since entry into this method. So we
     * recheck state and if necessary roll back the enqueuing if
     * stopped, or start a new thread if there are none.
     * 如果一个任务能够成功入队
     *
     * 3. If we cannot queue task, then we try to add a new
     * thread.  If it fails, we know we are shut down or saturated
     * and so reject the task.
     * 如果不能将这个任务入队，我们就尝试添加一个新线程。如果添加失败，意味着所有的线程
     * 被关闭或饱和，就拒绝该任务。
     */
    int c = ctl.get();
    if (workerCountOf(c) < corePoolSize) {
        if (addWorker(command, true))
            return;
        c = ctl.get();
    }
    if (isRunning(c) && workQueue.offer(command)) {
        int recheck = ctl.get();
        if (! isRunning(recheck) && remove(command))
            reject(command);
        else if (workerCountOf(recheck) == 0)
            addWorker(null, false);
    }
    else if (!addWorker(command, false))
        reject(command);
}
```

调试时，首先断在ThreadPoolExecutor构造方法中，然后一直按F9，发现一直断在getTask方法的如下代码处：

![image-20211215101631769](https://cdn.jsdelivr.net/gh/YuanAaron/BlogImage/2021/image-20211215101631769.png)

至此，我们就找到了keepAliveTime的使用地方，因此，只需要分析清楚上图代码中keepAliveTime的作用即可。

线程池中keepAliveTime的作用：

1. 控制非核心线程的超时，让他们自然消亡；
2. 如果开启了允许核心线程超时，它也可以控制核心线程的消亡。

线程池中keepAliveTime的原理：

1. 通过控制线程从阻塞队列中取任务的方式来控制当前线程的消亡。如果允许核心线程超时或工作线程数大于核心线程数，调用bq.poll(keepAliveTime,unit)，即取不到任务就返回null；反之，调用bq.take()方法，即取不到任务，当前线程就阻塞，直到取到任务为止（线程复用）。
2. bq.poll(keepAliveTime,unit)超时后会返回null，即线程池中的这个线程超时了，timeout设置为true，这样getTask的下次for循环就会返回null。
3. 接下来，在runWorker方法中判断(task=getTask()) == null，那么就会跳出while循环，相当于自然销毁这个线程（查看Thread类，发现线程的销毁有stop方法，但是已经不建议使用，而是倾向于让线程自然消亡）。
