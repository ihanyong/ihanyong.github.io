---
layout: post
title:  "【Netty源码解析】Netty线程模型详解-EventLoop-EventLoopGroup"
date:   2019-04-23 20:00:00 +0800
tags:
        - netty
---


# 几个重要的接口
- EventLoopGroup/EventLoop
- EventExecutorGroup/EventExecutor

![接口的类图](https://raw.githubusercontent.com/ihanyong/ihanyong.github.io/master/img/netty/eventloop/eventloopsinterfaces.png)

### EventExecutorGroup
EventExecutorGroup 主要用途通过next()方法来提供EventExecutor，并负责管理它们的生命周期（全局性的shutdown）。 继承了ScheduledExecutorService接口，所以可以理解它就是一个线程池，可以接受任务来处理。

### EventExecutor  
EventExecutor 是 EventExecutorGroup 的子接口，也是一个线程池。 增加了一些便利方法来判断一个线程与一个事件循环是不是绑定的关系。



### EventLoopGroup
一个EventExecutorGroup， 允许注册之后在事件循环中处理的Channel。

### EventLoop
用于处理注册的 Channel 的所有的 I/O 处理。 Channel根据具体的实现，一个EventLoop通常可以处理多个 Channel。



# 从Nio的EventLoopGroup实现来解析

![实现类的类图](https://raw.githubusercontent.com/ihanyong/ihanyong.github.io/master/img/netty/eventloop/eventloopimpls.png)

如上面的类图， 黄色框内是EventLoopGroup相关的实现， 红色框内是EventLoop相关的实现

通过实现类的名称可以知道，EventExecutorGroup与EventLoopGroup的实现的是多线程实现的；EventExecutor与EventLoop是单线程实现。下面我们挨个的看一下各个类的实现源码。

## EventExecutorGroup/EventLoopGroup

### AbstractEventExecutorGroup
AbstractEventExecutorGroup 是所有 EventExecutorGroup 实现的基类。 主要实现了线程池任务提交管理的方法(invokeAll(), invokeAny(), submit(), executor())， 如下所示都是将对应的方法调用委托给 next() 取得的 EventExecutor。

```java
    // ...
    @Override
    public Future<?> submit(Runnable task) {
        return next().submit(task);
    }
    @Override
    public ScheduledFuture<?> schedule(Runnable command, long delay, TimeUnit unit) {
        return next().schedule(command, delay, unit);
    }
    @Override
    public <T> List<java.util.concurrent.Future<T>> invokeAll(Collection<? extends Callable<T>> tasks)
            throws InterruptedException {
        return next().invokeAll(tasks);
    }
    @Override
    public <T> T invokeAny(Collection<? extends Callable<T>> tasks, long timeout, TimeUnit unit)
            throws InterruptedException, ExecutionException, TimeoutException {
        return next().invokeAny(tasks, timeout, unit);
    }

    @Override
    public void execute(Runnable command) {
        next().execute(command);
    }
    // ...
```

### MultithreadEventExecutorGroup
MultithreadEventExecutorGroup 是EventExecutorGroup多线程实现的基类。

先看一下定义了哪些属性

```java
    // EventExecutorGroup 管理的 EventExecutor
    private final EventExecutor[] children;
    private final Set<EventExecutor> readonlyChildren;

    private final AtomicInteger terminatedChildren = new AtomicInteger();
    private final Promise<?> terminationFuture = new DefaultPromise(GlobalEventExecutor.INSTANCE);

    // EventExecutor选择器， 用于next()方法， 从children中选择一个 EventExecutor
    private final EventExecutorChooserFactory.EventExecutorChooser chooser;
```

在构造方法， 调用抽象方法newChild()获取EventExecutor,来始化children数组。

```java

    protected MultithreadEventExecutorGroup(int nThreads, Executor executor,
                                            EventExecutorChooserFactory chooserFactory, Object... args) {
        if (nThreads <= 0) {
            throw new IllegalArgumentException(String.format("nThreads: %d (expected: > 0)", nThreads));
        }

        if (executor == null) {
            executor = new ThreadPerTaskExecutor(newDefaultThreadFactory());
        }

        // 生成指定线程数的children 数组
        children = new EventExecutor[nThreads];

        // 调用抽象方法newChild()， 生成EventExecutor存入children数组
        for (int i = 0; i < nThreads; i ++) {
            boolean success = false;
            try {
                children[i] = newChild(executor, args);
                success = true;
            } catch (Exception e) {
                // TODO: Think about if this is a good exception type
                throw new IllegalStateException("failed to create a child event loop", e);
            } finally {
                if (!success) {
                    for (int j = 0; j < i; j ++) {
                        children[j].shutdownGracefully();
                    }

                    for (int j = 0; j < i; j ++) {
                        EventExecutor e = children[j];
                        try {
                            while (!e.isTerminated()) {
                                e.awaitTermination(Integer.MAX_VALUE, TimeUnit.SECONDS);
                            }
                        } catch (InterruptedException interrupted) {
                            // Let the caller handle the interruption.
                            Thread.currentThread().interrupt();
                            break;
                        }
                    }
                }
            }
        }

        // 初始化选择器
        chooser = chooserFactory.newChooser(children);

        final FutureListener<Object> terminationListener = new FutureListener<Object>() {
            @Override
            public void operationComplete(Future<Object> future) throws Exception {
                if (terminatedChildren.incrementAndGet() == children.length) {
                    terminationFuture.setSuccess(null);
                }
            }
        };

        for (EventExecutor e: children) {
            e.terminationFuture().addListener(terminationListener);
        }

        Set<EventExecutor> childrenSet = new LinkedHashSet<EventExecutor>(children.length);
        Collections.addAll(childrenSet, children);
        readonlyChildren = Collections.unmodifiableSet(childrenSet);
    }


    @Override
    public EventExecutor next() {
        // 委托给chooser选取下一个 EventExecutor
        return chooser.next();
    }


    /**
     * Create a new EventExecutor which will later then accessible via the {@link #next()}  method. This method will be
     * called for each thread that will serve this {@link MultithreadEventExecutorGroup}.
     *
     */
    protected abstract EventExecutor newChild(Executor executor, Object... args) throws Exception;

```

EventExecutorChooser的实现有两个， EventExecutorChooserFactory会根据线程数来选择不同的实现。
```java
    private static final class PowerOfTwoEventExecutorChooser implements EventExecutorChooser {
        private final AtomicInteger idx = new AtomicInteger();
        private final EventExecutor[] executors;

        PowerOfTwoEventExecutorChooser(EventExecutor[] executors) {
            this.executors = executors;
        }

        @Override
        public EventExecutor next() {
            return executors[idx.getAndIncrement() & executors.length - 1];
        }
    }

    private static final class GenericEventExecutorChooser implements EventExecutorChooser {
        private final AtomicInteger idx = new AtomicInteger();
        private final EventExecutor[] executors;

        GenericEventExecutorChooser(EventExecutor[] executors) {
            this.executors = executors;
        }

        @Override
        public EventExecutor next() {
            return executors[Math.abs(idx.getAndIncrement() % executors.length)];
        }
    }
```

### MultithreadEventLoopGroup
MultithreadEventLoopGroup 是继承了 MultithreadEventExecutorGroup 类，并实现了 EventLoopGroup 接口。
实现上，具体化了 MultithreadEventExecutorGroup 的 next()与newChild()方法的返回值为 EventLoop (EventExecutor的子接口)。  并实现了 EventLoopGroup 接口定义的 resigter()方法： 通过 next() 选择一个 EventLoop， 并将参数 Channel 注册到该 EventLoop。
```java
    @Override
    public ChannelFuture register(Channel channel) {
        return next().register(channel);
    }
```

### NioEventLoopGroup
上面相关的都是一些抽象类， NioEventLoopGroup则是一个具体的实现类， 实现了 newChild() 方法： 实例化一个 NioEventLoop()。

```java
    @Override
    protected EventLoop newChild(Executor executor, Object... args) throws Exception {
        return new NioEventLoop(this, executor, (SelectorProvider) args[0],
            ((SelectStrategyFactory) args[1]).newSelectStrategy(), (RejectedExecutionHandler) args[2]);
    }
```


## EventExecutor/EventLoop
下面再来看EventExecutor/EventLoop相关的实现。

### AbstractEventExecutor

AbstractEventExecutor 继承自J.U.C 包的AbstractExecutorService， 主要实现如下：

- 定义一个parent属性，指向管理自己的 EventExecutorGroup实例。
- 实现 next()方法， 并返回自身(this)。
- 将 submit() 等方法的返回值由JUC的 Future 具体化为netty自己实现的 Future。
- 定义一个 safeExecute(Runnable task) 方法， 直接执行task.run()，并捕获打印异常（不向上抛出）

### AbStractScheduledEventExecutor
继承了 AbstractEventExecutor， 通过优先级队列 scheduledTaskQueue 来实现 ScheduledExecutorService 接口定义的 schedule 相关的方法。

### SingleThreadEventExecutor
EventExecutor单线程实现抽象基类。 继承了 AbstractScheduledEventExecutor 。
定义了一个 任务队列 LinkedBlockingQueue<Runnable> taskQueue 来管理提交的任务。


实现了 execute(Runnable task) 方法， 接受一个任务(addTask(task)) 将任务添加到任务队列，并尝试启动线程(startThread()) 调用子类的 run()方法开始处理任务队列中的任务。

execute(Runnable task) 实现：
```java
    @Override
    public void execute(Runnable task) {
        if (task == null) {
            throw new NullPointerException("task");
        }

        boolean inEventLoop = inEventLoop();
        // 将任务添加到任务队列
        addTask(task);
        if (!inEventLoop) {
            // 启动线程，处理任务队列中的任务
            startThread();
            if (isShutdown() && removeTask(task)) {
                reject();
            }
        }

        if (!addTaskWakesUp && wakesUpForTask(task)) {
            wakeup(inEventLoop);
        }
    }
```


startThread() 实现：

```java

    private void startThread() {
        // 判断是否已经启动，确保只启动一次。
        if (state == ST_NOT_STARTED) {
            if (STATE_UPDATER.compareAndSet(this, ST_NOT_STARTED, ST_STARTED)) {
                try {
                    doStartThread();
                } catch (Throwable cause) {
                    STATE_UPDATER.set(this, ST_NOT_STARTED);
                    PlatformDependent.throwException(cause);
                }
            }
        }
    }

    private void doStartThread() {
        assert thread == null;
        // 户传可以在声明 EventLoopGroup时自定义 executor ，
        // 没有的话MultithreadEventExecutorGroup构造方法中默认创建一个 ThreadPerTaskExecutor
        executor.execute(new Runnable() {
            @Override
            public void run() {
                // 保存当前线程引用，用于 inEventLoop 方法判断。
                thread = Thread.currentThread();
                if (interrupted) {
                    thread.interrupt();
                }

                boolean success = false;
                updateLastExecutionTime();
                try {
                    // 调用对应子类的 run()方法
                    SingleThreadEventExecutor.this.run();
                    success = true;
                } catch (Throwable t) {
                    logger.warn("Unexpected exception from an event executor: ", t);
                } finally {
                    for (;;) {
                        int oldState = state;
                        if (oldState >= ST_SHUTTING_DOWN || STATE_UPDATER.compareAndSet(
                                SingleThreadEventExecutor.this, oldState, ST_SHUTTING_DOWN)) {
                            break;
                        }
                    }

                    // Check if confirmShutdown() was called at the end of the loop.
                    if (success && gracefulShutdownStartTime == 0) {
                        if (logger.isErrorEnabled()) {
                            logger.error("Buggy " + EventExecutor.class.getSimpleName() + " implementation; " +
                                    SingleThreadEventExecutor.class.getSimpleName() + ".confirmShutdown() must " +
                                    "be called before run() implementation terminates.");
                        }
                    }

                    try {
                        // Run all remaining tasks and shutdown hooks.
                        for (;;) {
                            if (confirmShutdown()) {
                                break;
                            }
                        }
                    } finally {
                        try {
                            cleanup();
                        } finally {
                            STATE_UPDATER.set(SingleThreadEventExecutor.this, ST_TERMINATED);
                            threadLock.release();
                            if (!taskQueue.isEmpty()) {
                                if (logger.isWarnEnabled()) {
                                    logger.warn("An event executor terminated with " +
                                            "non-empty task queue (" + taskQueue.size() + ')');
                                }
                            }

                            terminationFuture.setSuccess(null);
                        }
                    }
                }
            }
        });
    }
```


定义抽象方法, 用于子类定义执行逻辑：
```java
protected abstract void run();
```


并于任务的获取在 takeTask() 方法中实现， 会尝试将到时的计划任务压入到执队列中，然后按顺序从执行队列中获取任务。

```java

    protected Runnable takeTask() {
        assert inEventLoop();
        if (!(taskQueue instanceof BlockingQueue)) {
            throw new UnsupportedOperationException();
        }

        BlockingQueue<Runnable> taskQueue = (BlockingQueue<Runnable>) this.taskQueue;
        for (;;) {
            ScheduledFutureTask<?> scheduledTask = peekScheduledTask();
            if (scheduledTask == null) {
                // 没有 scheduledTask 直接获取 taskQueue 中的任务
                Runnable task = null;
                try {
                    task = taskQueue.take();
                    if (task == WAKEUP_TASK) {
                        task = null;
                    }
                } catch (InterruptedException e) {
                    // Ignore
                }
                return task;
            } else {
                // 如果有 scheduledTask 看一下是否有任务到了执行时间。

                long delayNanos = scheduledTask.delayNanos();
                Runnable task = null;
                if (delayNanos > 0) {
                    // 没有任务到执行时间，直接获取 taskQueue 中的任务
                    try {
                        task = taskQueue.poll(delayNanos, TimeUnit.NANOSECONDS);
                    } catch (InterruptedException e) {
                        // Waken up.
                        return null;
                    }
                }
                if (task == null) {
                    // 有任务到执行时间，
                    // 尝试将 scheduledTaskQueue 到到执行时间的任务
                    // 移动到 taskQueue中, 再从 taskQueue 中获取任务
                    fetchFromScheduledTaskQueue();
                    task = taskQueue.poll();
                }

                if (task != null) {
                    return task;
                }
            }
        }
    }

    private boolean fetchFromScheduledTaskQueue() {
        // 从 scheduledTaskQueue 中获取所有超过delay时间的任务，放入taskQueue
        // 如果 taskQueue 已经无法放入更多的任务，刚将任务还还到 scheduledTaskQueue 并返回

        long nanoTime = AbstractScheduledEventExecutor.nanoTime();
        Runnable scheduledTask  = pollScheduledTask(nanoTime);
        while (scheduledTask != null) {
            if (!taskQueue.offer(scheduledTask)) {
                // No space left in the task queue add it back to the scheduledTaskQueue so we pick it up again.
                scheduledTaskQueue().add((ScheduledFutureTask<?>) scheduledTask);
                return false;
            }
            scheduledTask  = pollScheduledTask(nanoTime);
        }
        return true;
    }
```


### SingleThreadEventLoop
继承了 SingleThreadEventExecutor， 并实现接口 EventLoop。

实现 register 方法， 将channel 的 unsafe 注册到eventloop: 
```java
    @Override
    public ChannelFuture register(final ChannelPromise promise) {
        ObjectUtil.checkNotNull(promise, "promise");
        promise.channel().unsafe().register(this, promise);
        return promise;
    }
```

另外提供了一个 tailTasks， 可用于提交一些任务队列执行完毕后的再执行的任务。
```java
    @UnstableApi
    public final void executeAfterEventLoopIteration(Runnable task) {
        ObjectUtil.checkNotNull(task, "task");
        if (isShutdown()) {
            reject();
        }

        if (!tailTasks.offer(task)) {
            reject(task);
        }

        if (wakesUpForTask(task)) {
            wakeup(inEventLoop());
        }
    }
    @UnstableApi
    final boolean removeAfterEventLoopIterationTask(Runnable task) {
        return tailTasks.remove(ObjectUtil.checkNotNull(task, "task"));
    }
    @Override
    protected void afterRunningAllTasks() {
        runAllTasksFrom(tailTasks);
    }
```


### NioEventLoop
这个一个具体的 EventLoop 实现类， 与NioEventLoopGroup配套使用（newChild()中实例化）。

看看run方法的实现：
主要思想是在一个死循环不断地调用 
1. Channel内部封装的Unsafe对象的read()方法，进行底层的IO处理。 
2. runAllTasks(), 处理任务队列中待处理的任务。

```java

    @Override
    protected void run() {
        // 循环处理， 确保服务关闭前，线程一直处理。
        for (;;) {
            try {
                switch (selectStrategy.calculateStrategy(selectNowSupplier, hasTasks())) {
                    case SelectStrategy.CONTINUE:
                        continue;

                    case SelectStrategy.BUSY_WAIT:
                        // fall-through to SELECT since the busy-wait is not supported with NIO

                    case SelectStrategy.SELECT:
                         // select(boolean oldWakenUp)方法 触发 NIO selector.selectNow()
                        // 会将select出来的结果 放入 selectedKeys
                        select(wakenUp.getAndSet(false));
                        if (wakenUp.get()) {
                            selector.wakeup();
                        }
                        // fall through
                    default:
                }

                cancelledKeys = 0;
                needsToSelectAgain = false;
                final int ioRatio = this.ioRatio;
                if (ioRatio == 100) {
                    try {
                        // 进行IO处理
                        processSelectedKeys();
                    } finally {
                        // 确保调用执行任务队列中所有的任务）
                        runAllTasks();
                    }
                } else {
                    final long ioStartTime = System.nanoTime();
                    try {
                        // 
                        processSelectedKeys();
                    } finally {
                        // 确保调用执行任务队列中所有的任务）
                        final long ioTime = System.nanoTime() - ioStartTime;
                        runAllTasks(ioTime * (100 - ioRatio) / ioRatio);
                    }
                }
            } catch (Throwable t) {
                handleLoopException(t);
            }
            // Always handle shutdown even if the loop processing threw an exception.
            try {
                if (isShuttingDown()) {
                    closeAll();
                    if (confirmShutdown()) {
                        return;
                    }
                }
            } catch (Throwable t) {
                handleLoopException(t);
            }
        }
    }
```

```java 

    private void processSelectedKeys() {
        // openSelector() 方法中， 会尝试将 selectedKeys 设置为 NIO Selector 内部的 selectedKeys， publicSelectedKeys， 
        // 设置不成功的话，selectedKeys会被重置为null
        if (selectedKeys != null) { 
            // select(boolean oldWakenUp)方法 触发 NIO selector.selectNow()
            // 会将select出来的结果 放入 selectedKeys，这里直接处理就行了
            processSelectedKeysOptimized();
        } else {
            processSelectedKeysPlain(selector.selectedKeys());
        }
    }

    private void processSelectedKeysOptimized() {
        for (int i = 0; i < selectedKeys.size; ++i) {
            final SelectionKey k = selectedKeys.keys[i];
            // null out entry in the array to allow to have it GC'ed once the Channel close
            // See https://github.com/netty/netty/issues/2363
            selectedKeys.keys[i] = null;

            final Object a = k.attachment();

            if (a instanceof AbstractNioChannel) {
                processSelectedKey(k, (AbstractNioChannel) a);
            } else {
                @SuppressWarnings("unchecked")
                NioTask<SelectableChannel> task = (NioTask<SelectableChannel>) a;
                processSelectedKey(k, task);
            }

            if (needsToSelectAgain) {
                // null out entries in the array to allow to have it GC'ed once the Channel close
                // See https://github.com/netty/netty/issues/2363
                selectedKeys.reset(i + 1);

                selectAgain();
                i = -1;
            }
        }
    }



    private void processSelectedKeysOptimized() {
        for (int i = 0; i < selectedKeys.size; ++i) {
            final SelectionKey k = selectedKeys.keys[i];
            // 从数组取出后就设置为null，以便后面Channel 关闭后进行GC
            selectedKeys.keys[i] = null;

            final Object a = k.attachment();

            if (a instanceof AbstractNioChannel) {
                // 处理Channel I/O
                processSelectedKey(k, (AbstractNioChannel) a);
            } else {
                @SuppressWarnings("unchecked")
                NioTask<SelectableChannel> task = (NioTask<SelectableChannel>) a;
                processSelectedKey(k, task);
            }

            if (needsToSelectAgain) {
                // null out entries in the array to allow to have it GC'ed once the Channel close
                // See https://github.com/netty/netty/issues/2363
                selectedKeys.reset(i + 1);

                selectAgain();
                i = -1;
            }
        }
    }



    private void processSelectedKey(SelectionKey k, AbstractNioChannel ch) {
        final AbstractNioChannel.NioUnsafe unsafe = ch.unsafe();
        if (!k.isValid()) {
            final EventLoop eventLoop;
            try {
                eventLoop = ch.eventLoop();
            } catch (Throwable ignored) {

                return;
            }

            if (eventLoop != this || eventLoop == null) {
                return;
            }
            unsafe.close(unsafe.voidPromise());
            return;
        }

        try {
            int readyOps = k.readyOps();

            if ((readyOps & SelectionKey.OP_CONNECT) != 0) {

                int ops = k.interestOps();
                ops &= ~SelectionKey.OP_CONNECT;
                k.interestOps(ops);

                unsafe.finishConnect();
            }

            if ((readyOps & SelectionKey.OP_WRITE) != 0) {

                ch.unsafe().forceFlush();
            }

            if ((readyOps & (SelectionKey.OP_READ | SelectionKey.OP_ACCEPT)) != 0 || readyOps == 0) {
                // 调用Channel中Unsafe对象的read()方法，进行处理 IO处理
                //
                unsafe.read();
            }
        } catch (CancelledKeyException ignored) {
            unsafe.close(unsafe.voidPromise());
        }
    }
```

