Java线程池ThreadPoolExecutor的实现.md


[线程池类框架](http://pr31ptshd.bkt.clouddn.com/image/java_basic/AbstractExecutorService-class.jpg)

- interface Executor : 基本的线程池接口， 只定义了一个execute 方法，接收 Runnable 类型的任务，没有返回值
- interface ExecutorService ： 增加了生命周期控制的shutdown()方法。 并增加了可以接收 Callable类型任务的submint/invokeAll/invokeAny等方法，并返回一个Future用于获取任务结果或控制参与任务的生命周期。
- abstract class AbstractExecutorService ： 实现了submit* invoke* 系列方法。  



# AbstractExecutorService
AbstractExecutorService 定义了线程池实现的基本框架， 我们自己实现自己的线程池时可以直接从继承AbstractExecutorService开始。 它实现了submit、 invokeAll、 invokeAny方法。 

所有的 submit 方法都是通过 newTaskFor 方法 将 Runnable Callable 等类型的任务封装为 FutureTask 类型，以统一交给 execute来执行任务处理， 具体的实现类只需要实现execute 方法即可。  如对于Callable类型的任务：

```java
    public <T> Future<T> submit(Callable<T> task) {
        if (task == null) throw new NullPointerException();
        // 将task 封装为一个 RunnableFuture
        RunnableFuture<T> ftask = newTaskFor(task);
        // 调用子类 execute 实现
        execute(ftask);
        // 将封装的  RunnableFuture 返回， 可用于 任务结果的获取，取消等
        return ftask;
    }
    protected <T> RunnableFuture<T> newTaskFor(Callable<T> callable) {
        return new FutureTask<T>(callable);
    }
```

invokeAll 与 submit 其实是类似是，主要逻辑也是将 task 封装为一个 FutureTask 并调用 execute 方法处理。 不同在于接受的一个任务集合，需要将集合内所有的任务都执行一遍，当所有的任务都完成后才返回。 

```java
    public <T> List<Future<T>> invokeAll(Collection<? extends Callable<T>> tasks)
        throws InterruptedException {
        if (tasks == null)
            throw new NullPointerException();

        // 声明一个大小与提交任务数一样的结果 futures 列表
        ArrayList<Future<T>> futures = new ArrayList<Future<T>>(tasks.size());
        boolean done = false;
        try {
            // 对提交的任务集合进行遍历，处理与submit一样
            for (Callable<T> t : tasks) {
                RunnableFuture<T> f = newTaskFor(t);
                futures.add(f);
                execute(f);
            }

            // 因为需要所有的任务都完成后才能返回，总的耗时是执行时长最长的那个任务
            // 不需要考虑任务执行的快慢，这里直接对futures进行遍历即可
            for (int i = 0, size = futures.size(); i < size; i++) {
                Future<T> f = futures.get(i);
                if (!f.isDone()) { 
                    try {
                        // 如果任务没有完成就阻塞直到完成
                        f.get();
                    // 阻塞的结果如果是取消和执行异常，这是不做处理，
                    // 而是正常迭代futures, 将异常留给调用方进行处理
                    // 如果是其它异常（中断），则会中止执行（跳到 finally 代码块中 done = false;）
                    } catch (CancellationException ignore) {
                    } catch (ExecutionException ignore) {
                    }
                }
            }
            done = true;
            return futures;
        } finally {
            // 如果有任务未完成就退出，尝试取消所有任务（不一定取消成功）
            if (!done)
                for (int i = 0, size = futures.size(); i < size; i++)
                    futures.get(i).cancel(true);
        }
    }
```

invokeAny, 顾名思义， 接受一组任务进行执行，当任务一个任务有完成就取消其它任务的执行并返回完成的结果。 其核心逻辑在 doInvokeAny()  方法中。

```java
    private <T> T doInvokeAny(Collection<? extends Callable<T>> tasks,
                              boolean timed, long nanos)
        throws InterruptedException, ExecutionException, TimeoutException {
        if (tasks == null)
            throw new NullPointerException();
        int ntasks = tasks.size();
        if (ntasks == 0)
            throw new IllegalArgumentException();
        ArrayList<Future<T>> futures = new ArrayList<Future<T>>(ntasks);


        // 声明一个 ExecutorCompletionService 来管理任务执行结果， 真正执行的线程池还是 this
        ExecutorCompletionService<T> ecs =
            new ExecutorCompletionService<T>(this);

        // 为了执行效率，尤其是并行度有限的情况下， 
        // 在提交新的任务之前先检查之前提交的任务有没有完成的。

        try {
            // 记录异常，如果最终一个正常结果也没有获取到，就将最后一个异常抛出
            ExecutionException ee = null;
            final long deadline = timed ? System.nanoTime() + nanos : 0L;
            Iterator<? extends Callable<T>> it = tasks.iterator();

            // 先提交一个任务
            futures.add(ecs.submit(it.next()));
            // 未提交的任务数
            --ntasks;
            // 执行中的任务数
            int active = 1;

            // 随处可见的自旋
            for (;;) {
                // 从ecs获取执行完成任务的future
                Future<T> f = ecs.poll();
                // 如果没有任务完成， 就再提交一个任务
                if (f == null) {
                    // 如果还有未提交的任务，再提交
                    if (ntasks > 0) {
                        --ntasks;
                        futures.add(ecs.submit(it.next()));
                        ++active;
                    }
                    // 如果所有的任务都已经提交并执行完成，退出自旋
                    else if (active == 0)
                        break;
                    else if (timed) {
                        // 任务已经全部提交，这里就阻塞地获取完成的任务future(再带超时)
                        f = ecs.poll(nanos, TimeUnit.NANOSECONDS);
                        if (f == null)
                            throw new TimeoutException();
                        nanos = deadline - System.nanoTime();
                    }
                    else
                        // 任务已经全部提交，这里就阻塞地获取完成的任务future（无超时）
                        f = ecs.take();
                }

                // 如果有任务完成了
                if (f != null) {
                    // 执行完成一个任务
                    --active;
                    try {
                        // 获取任务结果，如果不是异常就直接返回， 
                        // 是异常则记录下来，继续查看其它的future 结果
                        return f.get();
                    } catch (ExecutionException eex) {
                        ee = eex;
                    } catch (RuntimeException rex) {
                        ee = new ExecutionException(rex);
                    }
                }
            }
            // 如何走到这里？   else if (active == 0) break 会走到这里但 ee应该不为null

            if (ee == null)
                ee = new ExecutionException();
            throw ee;

        } finally {
            // 返回前，尝试取消其它还正在执行的任务
            for (int i = 0, size = futures.size(); i < size; i++)
                futures.get(i).cancel(true);
        }
    }
```

# ThreadPoolExecutor
先看ThreadPoolExecutor的属性

- ctl : 
- workQueue
- workers
- mainLock

- termination
- largestPoolSize
- completedTaskCount
- threadFactory
- handler
- keepAliveTime
- allowCoreThreadTimeOut
- corePoolSize
- maximumPoolSize
- defaultHandler


ctl

线程池的控制状态， 一个包含了两个逻辑字段的AtomicInteger: 
- workerCount: 线程数
- runState: 线程状态，如 RUNNING， SHUTDOWN 等 

为了将两个字段整合到一个整数中，  workerCount 最大值只有(2^29)-1 大约5亿，完整的整数应该(2^31)-1大概有20亿 。 如果这个限制有什么问题的话， 可以将数据改为 AtomicLong ， 并调整下shift/mask方法即可。 但是以当前的需求来说，AtomicInteger　完全够了，而且稍微快些且简单些。

workerCount　是允许启动的但不允许被停止的线程数。　在某一个瞬间上，　这个值可能与实际上存活的线程数不同，比如，当已经申请了创建线程但ThreadFactory创建失败时，在退出前这个线程还是算在工作线程数内的。　用户可见的线程池大小的工作线程集合的当前大小。


线程池的生命周期状态有：

- RUNNING ： 接收并处理队列中的任务
- SHUTDOWN ：  不再接收新任务，但会继续处理队列中的任务。
- STOP ： 不再接收新任务，也不再处理队列中的任务，并中断处理中的任务。
- TIDYING ： 所有的任务都已经终止， workerCount 为0， 过渡到 TIDYING状态的线程会调用钩子方法 terminated()
- TERMINATED ： terminated()已经完成

![状态迁移图](http://pr31ptshd.bkt.clouddn.com/image/java_basic/threadpool-state-transitions.png)


```java
private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING, 0));
```


## ThreadPoolExecutor 的 execute 方法实现
下面 execute 方法为切入点进行分析

```java

    public void execute(Runnable command) {
        if (command == null)
            throw new NullPointerException();
        // 分3步处理：
        // 1. 如果当前运行线程数小于 corePoolSize， 尝试启动一个新的线程，
        // 传入的任务作为这个线程的第一个任务。 
        // 
        // 2.  如果任务可以成功加入到队列中， 
        // 还是需要进行二次检查一下是否需要增加一个新的线程， 或者线程池有没有关闭
        // 根据检查状态，线程池关闭时需要将压入队列的任务撤回；
        // 线程池里没有线程的需要启动一个新的线程
        // 
        // 3. 如果没有成功加入队列， 再尝试一下启动新的线程来执行任务。
        //  如果再次失败了，说明线程池关闭或饱和了，需要拒绝任务。
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




    private boolean addWorker(Runnable firstTask, boolean core) {
        retry:
        for (;;) {
            int c = ctl.get();
            int rs = runStateOf(c);

            // Check if queue empty only if necessary.
            if (rs >= SHUTDOWN &&
                ! (rs == SHUTDOWN &&
                   firstTask == null &&
                   ! workQueue.isEmpty()))
                return false;

            for (;;) {
                int wc = workerCountOf(c);
                if (wc >= CAPACITY ||
                    wc >= (core ? corePoolSize : maximumPoolSize))
                    return false;
                if (compareAndIncrementWorkerCount(c))
                    break retry;
                c = ctl.get();  // Re-read ctl
                if (runStateOf(c) != rs)
                    continue retry;
                // else CAS failed due to workerCount change; retry inner loop
            }
        }

        boolean workerStarted = false;
        boolean workerAdded = false;
        Worker w = null;
        try {
            w = new Worker(firstTask);
            final Thread t = w.thread;
            if (t != null) {
                final ReentrantLock mainLock = this.mainLock;
                mainLock.lock();
                try {
                    // Recheck while holding lock.
                    // Back out on ThreadFactory failure or if
                    // shut down before lock acquired.
                    int rs = runStateOf(ctl.get());

                    if (rs < SHUTDOWN ||
                        (rs == SHUTDOWN && firstTask == null)) {
                        if (t.isAlive()) // precheck that t is startable
                            throw new IllegalThreadStateException();
                        workers.add(w);
                        int s = workers.size();
                        if (s > largestPoolSize)
                            largestPoolSize = s;
                        workerAdded = true;
                    }
                } finally {
                    mainLock.unlock();
                }
                if (workerAdded) {
                    t.start();
                    workerStarted = true;
                }
            }
        } finally {
            if (! workerStarted)
                addWorkerFailed(w);
        }
        return workerStarted;
    }

    private final class Worker
        extends AbstractQueuedSynchronizer
        implements Runnable
    {
        Worker(Runnable firstTask) {
            setState(-1); // inhibit interrupts until runWorker
            this.firstTask = firstTask;
            this.thread = getThreadFactory().newThread(this);
        }
        /** Delegates main run loop to outer runWorker  */
        public void run() {
            runWorker(this);
        }
    }



    final void runWorker(Worker w) {
        Thread wt = Thread.currentThread();
        Runnable task = w.firstTask;
        w.firstTask = null;
        w.unlock(); // allow interrupts
        boolean completedAbruptly = true;
        try {
            while (task != null || (task = getTask()) != null) {
                w.lock();
                // If pool is stopping, ensure thread is interrupted;
                // if not, ensure thread is not interrupted.  This
                // requires a recheck in second case to deal with
                // shutdownNow race while clearing interrupt
                if ((runStateAtLeast(ctl.get(), STOP) ||
                     (Thread.interrupted() &&
                      runStateAtLeast(ctl.get(), STOP))) &&
                    !wt.isInterrupted())
                    wt.interrupt();
                try {
                    beforeExecute(wt, task);
                    Throwable thrown = null;
                    try {
                        task.run();
                    } catch (RuntimeException x) {
                        thrown = x; throw x;
                    } catch (Error x) {
                        thrown = x; throw x;
                    } catch (Throwable x) {
                        thrown = x; throw new Error(x);
                    } finally {
                        afterExecute(task, thrown);
                    }
                } finally {
                    task = null;
                    w.completedTasks++;
                    w.unlock();
                }
            }
            completedAbruptly = false;
        } finally {
            processWorkerExit(w, completedAbruptly);
        }
    }

```









参考
- [Java中的多线程你只要看这一篇就够了](https://www.cnblogs.com/wxd0108/p/5479442.html)
