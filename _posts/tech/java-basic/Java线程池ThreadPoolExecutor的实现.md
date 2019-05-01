Java线程池ThreadPoolExecutor的实现.md


# AbstractExecutorService
实现了 submit、 invokeAll、 invokeAny方法。

所有的 submit 方法都是通过 newTaskFor 方法 将 Runnable Callable 等类型的任务封装为 FutureTask 类型，以统一交给 execute 需要个体的子类来实现。 方法处理， execute 需要个体的子类来实现。 如对于Callable类型的任务：

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
先看几个主要的属性


ctl

线程池的控制状态， 一个包含了两个逻辑字段的AtomicInteger: 
- workerCount: 线程数
- runState: 线程状态，如 RUNNING， SHUTDOWN 等 
有点Map的意思（runState -> workerCount）。 

为了将两个字段整合到一个整数中，  workerCount 最大值只有(2^29)-1 大约5亿，完整的整数应该(2^31)-1大概有20亿 。 如果这个限制有什么问题的话， 可以将数据改为 AtomicLong ， 并调整下shift/mask方法即可。 但是以当前的需求来说，AtomicInteger　完全够了，而且稍微快些且简单些。

workerCount　是允许启动的但不允许被停止的线程数。　在某一个瞬间上，　这个值可能与实际上存活的线程数不同，比如，当已经申请了创建线程但ThreadFactory创建失败时，在退出前这个线程还是算在工作线程数内的。　用户可见的线程池大小的工作线程集合的当前大小。

```java
   /**
     * The main pool control state, ctl, is an atomic integer packing
     * two conceptual fields
     *   workerCount, indicating the effective number of threads
     *   runState,    indicating whether running, shutting down etc
     *
     * In order to pack them into one int, we limit workerCount to
     * (2^29)-1 (about 500 million) threads rather than (2^31)-1 (2
     * billion) otherwise representable. If this is ever an issue in
     * the future, the variable can be changed to be an AtomicLong,
     * and the shift/mask constants below adjusted. But until the need
     * arises, this code is a bit faster and simpler using an int.
     *
     * The workerCount is the number of workers that have been
     * permitted to start and not permitted to stop.  The value may be
     * transiently different from the actual number of live threads,
     * for example when a ThreadFactory fails to create a thread when
     * asked, and when exiting threads are still performing
     * bookkeeping before terminating. The user-visible pool size is
     * reported as the current size of the workers set.
     *
     * The runState provides the main lifecycle control, taking on values:
     *
     *   RUNNING:  Accept new tasks and process queued tasks
     *   SHUTDOWN: Don't accept new tasks, but process queued tasks
     *   STOP:     Don't accept new tasks, don't process queued tasks,
     *             and interrupt in-progress tasks
     *   TIDYING:  All tasks have terminated, workerCount is zero,
     *             the thread transitioning to state TIDYING
     *             will run the terminated() hook method
     *   TERMINATED: terminated() has completed
     *
     * The numerical order among these values matters, to allow
     * ordered comparisons. The runState monotonically increases over
     * time, but need not hit each state. The transitions are:
     *
     * RUNNING -> SHUTDOWN
     *    On invocation of shutdown(), perhaps implicitly in finalize()
     * (RUNNING or SHUTDOWN) -> STOP
     *    On invocation of shutdownNow()
     * SHUTDOWN -> TIDYING
     *    When both queue and pool are empty
     * STOP -> TIDYING
     *    When pool is empty
     * TIDYING -> TERMINATED
     *    When the terminated() hook method has completed
     *
     * Threads waiting in awaitTermination() will return when the
     * state reaches TERMINATED.
     *
     * Detecting the transition from SHUTDOWN to TIDYING is less
     * straightforward than you'd like because the queue may become
     * empty after non-empty and vice versa during SHUTDOWN state, but
     * we can only terminate if, after seeing that it is empty, we see
     * that workerCount is 0 (which sometimes entails a recheck -- see
     * below).
     */
private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING, 0));
```

workers 是当前

```java
    /**
     * 线程池中所有工作线程的集合，只有在取得 mainLock 锁时才能访问
     */
    private final HashSet<Worker> workers = new HashSet<Worker>();

    /**
     * Lock held on access to workers set and related bookkeeping.
     * While we could use a concurrent set of some sort, it turns out
     * to be generally preferable to use a lock. Among the reasons is
     * that this serializes interruptIdleWorkers, which avoids
     * unnecessary interrupt storms, especially during shutdown.
     * Otherwise exiting threads would concurrently interrupt those
     * that have not yet interrupted. It also simplifies some of the
     * associated statistics bookkeeping of largestPoolSize etc. We
     * also hold mainLock on shutdown and shutdownNow, for the sake of
     * ensuring workers set is stable while separately checking
     * permission to interrupt and actually interrupting.
     */
    private final ReentrantLock mainLock = new ReentrantLock();

    /**
     * The queue used for holding tasks and handing off to worker
     * threads.  We do not require that workQueue.poll() returning
     * null necessarily means that workQueue.isEmpty(), so rely
     * solely on isEmpty to see if the queue is empty (which we must
     * do for example when deciding whether to transition from
     * SHUTDOWN to TIDYING).  This accommodates special-purpose
     * queues such as DelayQueues for which poll() is allowed to
     * return null even if it may later return non-null when delays
     * expire.
     */
    private final BlockingQueue<Runnable> workQueue;
```



下面 execute 方法为切入点进行分析

```java

    /**
     * Executes the given task sometime in the future.  The task
     * may execute in a new thread or in an existing pooled thread.
     *
     * If the task cannot be submitted for execution, either because this
     * executor has been shutdown or because its capacity has been reached,
     * the task is handled by the current {@code RejectedExecutionHandler}.
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
         *
         * 2. If a task can be successfully queued, then we still need
         * to double-check whether we should have added a thread
         * (because existing ones died since last checking) or that
         * the pool shut down since entry into this method. So we
         * recheck state and if necessary roll back the enqueuing if
         * stopped, or start a new thread if there are none.
         *
         * 3. If we cannot queue task, then we try to add a new
         * thread.  If it fails, we know we are shut down or saturated
         * and so reject the task.
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