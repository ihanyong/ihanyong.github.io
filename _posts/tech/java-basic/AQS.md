AQS

说到并发， 不能不深入了解一下 AbstractQueuedSynchronizer （AQS）。
ReentrantLock， ReentrantReadWriteLock， Semaphore，CountDownLatch 都是基于AQS实现的。 
其核心是基于一个先进先出(FIFO)的等待队列和一个表示状态的支持原子操作的整数值。 子类需要实现几个方法来改变这个状态，来表达资源的占用和释放。


# 代码框架
## AQS类的组成

AbstractQueuedSynchronizer 定义了一个 **volatile int state** 来表示资源。 state 只能通过 getState() setState(int) compareAndSetState(expect, update)来进行原子性的访问。
在自定义同步器中只需要实现下面几个方法，使用上面state的gettersetter方法来实现资源的获取与释放。， 

独占模式
- tryAcquire(int)： 尝试以独占的方式获取资源。 如果获取失败，调用方可以将当前线程推入资源等待队列，直到其它线程释放资源时通知这个线程。
- tryRelease(int)： tryAcquire的反向操作。 返回一个boolean, true表示已经完全释放了占用的资源，其它等待线程可以尝试获取资源；如果为false，表示还没有将资源完全释放。

共享模式
- tryAcuqireShared(int)： 尝试以共享的模式获取资源。 返回负数：失败； 0：表示获取成功，但没有剩余可用的资源供其它线程获取了；正数： 表示获取成功，并且还有剩余的资源供其它的线程获取。
- tryReleaseShared(int)： 尝试释放资源， tryAcuqireShared的反向操作。 

其它
- isHeldExclusively(): 判断资源是否被当前线程独占。只在 condition中使用。　如果不使用condition就不需要实现这个方法。


AbstractQueuedSynchronizer 维护了一个是一个双向链表队列。节点是 内部类 Node 来定义的。



## Node
等待队列是"CLH" (Craig, Landin, and Hagersten)的一个变种。 CLH 一般用于自旋锁。 

Node 中的状态waitStatus的取值：
- SIGNAL ： 本节点的后续节点是被阻塞的，当前节点释放资源或者取消时，必须唤醒后续节点。 其实就是标识一个节点是否为活跃状态。为了避免并发问题，获取方法必须要先设置 SIGNAL。然后再重试原子性的获取，再次获取失败后才进入阻塞。
- CANCELLED ： 本节点关联的线程因为超时或中断而取消。这是一个终止状态，不能再转变为其它状态。 取消后，其关联的线程不再被阻塞。
- CONDITION ： 此节点当前在条件队列中。 当在其所等待的condtion 上调用了single()方法后， 节点从条件队列移到同步队列中去，参与资源的获取
- PROPAGATE ： 在共享模式中标识节点处于可运行状态。
- 0 : 初始化状态。

Node 定义了两种模式
- SHARD 共享模式： state 代码的资源可以同时被多个线程使用。
- EXCLUSIVE 独占模式 : state 代码的资源同一时间只能被一个线程使用。


此节点当前位于条件队列中。

在传输之前，它不会用作同步队列节点，此时状态将设置为0。（此处使用该值与字段的其他用途无关，但简化了力学。） 


```java
    static final class Node {

        /** Marker to indicate a node is waiting in shared mode */
        static final Node SHARED = new Node();
        /** Marker to indicate a node is waiting in exclusive mode */
        static final Node EXCLUSIVE = null;

        /** waitStatus value ： 线程已取消 */
        static final int CANCELLED =  1;
        /** waitStatus value ： 需要通知后继节点 */  
        static final int SIGNAL    = -1;
        /** waitStatus value ： 节点在条件上等待 */
        static final int CONDITION = -2;
        /**
         * waitStatus value to ： 共享模式下使用，表示（todo）
         */
        static final int PROPAGATE = -3;

    
        volatile int waitStatus;

        /** 前一个节点 */
        volatile Node prev;

        /** 后继节点 */
        volatile Node next;

        /**
         * 节点关联的线程（插入本节点的线程）
         */
        volatile Thread thread;

        /**
         * SHARED 表示节点为共享模式。 独占模式时，要么为null, 要么指向等待队列（condition 只能是独占模式的）。
         */
        Node nextWaiter;

        final boolean isShared() {
            return nextWaiter == SHARED;
        }

        /**
         * 获取前一个节点
         * @return the predecessor of this node
         */
        final Node predecessor() throws NullPointerException {
            Node p = prev;
            if (p == null)
                throw new NullPointerException();
            else
                return p;
        }

        Node() {    // 用于构建一个初始化的head 或者 SHARED 标识对象
        }

        Node(Thread thread, Node mode) {     // Used by addWaiter
            this.nextWaiter = mode;
            this.thread = thread;
        }

        Node(Thread thread, int waitStatus) { // Used by Condition
            this.waitStatus = waitStatus;
            this.thread = thread;
        }
    }
```

# 代码详解
下面以 acquire，release, acquireShared, releaseShared 的顺序来详细看一下实现的源码。 主要在代码中进行注释的方式来说明。


## acquire

```java
    public final void acquire(int arg) {
        // 直接尝试获取资源，成功直接返回。子类根据需求实现
        if (!tryAcquire(arg) &&  
            // 将线程以独占模式，加入等待队列尾部。 
            //返回值：true 等待加入队列的过程中被中断过（但没有处理中断）
            acquireQueued(addWaiter(Node.EXCLUSIVE), arg)) 
            // 补偿处理中断
            selfInterrupt(); 
    }


    /**
     * 根据的模式，为当前线程创建一个node并加入队列
     * 
     * @param 模式 Node.EXCLUSIVE ： 独占模式, Node.SHARED ： 共享模式
     * @return 新创建的node
     */
    private Node addWaiter(Node mode) {
        Node node = new Node(Thread.currentThread(), mode);
        // 先假设队列已经初始化，尝试不自旋快速将新节点加入队列尾部
        Node pred = tail;
        if (pred != null) {
            node.prev = pred;
            if (compareAndSetTail(pred, node)) {
                pred.next = node;
                return node;
            }
        }
        // 如果不能以快速方式加入队列，调用enq来加入队列（自旋+队列初始化）
        enq(node);
        return node;
    }

    /**
     * 将节点插入队列，有必要的话初始化队列
     * @param node the node to insert
     * @return node's predecessor
     */
    private Node enq(final Node node) {
        // 自旋的方式，直接到节点插入队列
        for (;;) {
            Node t = tail;
            if (t == null) {
                // 尾节点为null 说明队列还未初始化，第一次自旋进行队列初始化
                if (compareAndSetHead(new Node()))
                    // CAS 的方式设置head节点为一个空节点
                    // 成功的话将head设置给tail, 
                    //失败的话说明被别的线程抢先了， 可以直接进入下一轮自旋
                    tail = head;
            } else {
                // 尝试用CAS的方式将node插入到tail, 成功返回，不成功自旋重试。
                node.prev = t;
                if (compareAndSetTail(t, node)) {
                    t.next = node;
                    return t;
                }
            }
        }
    }


    /**
     * 为等待队列中的线程不断地尝试获取资源。
     * 可以 condition wait 方法和 acquire 方法调用。
     *
     * @param node the node
     * @param arg the acquire argument
     * @return {@code true} if interrupted while waiting
     */
    final boolean acquireQueued(final Node node, int arg) {
        // 是否成功获取到资源
        boolean failed = true;
        try {
            // 等待过程中是否被中断过
            boolean interrupted = false;
            // 自旋
            for (;;) {
                final Node p = node.predecessor();
                // 如果在队列中排到第二个了，尝试获取资源
                if (p == head && tryAcquire(arg)) {
                    // 获取到资源了，将节点设置为head
                    setHead(node);
                    // 前一个头节点的 next 置null, for GC
                    p.next = null; // help GC
                    // 成功获取到资源，并返回线程中断状态
                    failed = false;
                    return interrupted;
                }
                // 本轮获取资源失败后，判断node关联的线程是否需要进入等待状态
                // 如果不需要，自旋回去再次尝试获取资源
                if (shouldParkAfterFailedAcquire(p, node) &&
                    // 需要进入 waiting 状态，等待被 unpark() 唤醒，并返回当前线程是否被中断过
                    parkAndCheckInterrupt())
                    // 记录当前线程是否被中断过，以便返回出去让调用者处理中断
                    interrupted = true;
            }
        } finally {
            if (failed)
                // 获取资源失败且退出了自旋，从队列中取消node
                cancelAcquire(node);
        }
    }


    /**
     * 判断一个获取资源失败的node所关联的线程是否可以进入等待状态。
     *
     * @param pred 前一个节点  pred == node.prev
     * @param node 当前节点
     * @return {@code true} 线程是否可以进入阻塞状态
     */
    private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
        int ws = pred.waitStatus;
        if (ws == Node.SIGNAL)
            /*
             * 如果前节点的 waitStatus 已经是 SIGNAL 了，可以本节点可以安全地进入 waiting 状态
             */
            return true;
        if (ws > 0) {
            /*
             * 前节点取消排队。 忽略掉本节点前面所有取消的节点。
             * 本节点不直接进行waiting 状态，调用者应该再尝试一次获取资源
             */
            do {
                node.prev = pred = pred.prev;
            } while (pred.waitStatus > 0);
            pred.next = node;
        } else {
            /*
             * 前节点的 waitStatus 是 0 或 PROPAGATE， 
             * 需要将前节点的 waitStatus 设为 SIGNAL（前一节点释放资源后给本节点发送信号）
             * 此时本节点不能直接进行waiting 状态，需要调用者应该再尝试一次获取资源
             * 如果再次失败通过  if (ws == Node.SIGNAL) 分支进入休眠
             */
            compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
        }
        return false;
    }


    /**
     * 使当前线程进入休眠状态node
     *
     * @return {@code true} if interrupted
     */
    private final boolean parkAndCheckInterrupt() {
        LockSupport.park(this);
        return Thread.interrupted();
    }


    private void cancelAcquire(Node node) {
        // Ignore if node doesn't exist
        if (node == null)
            return;

        node.thread = null;

        // 略过前面所有的取消节点
        Node pred = node.prev;
        while (pred.waitStatus > 0)
            node.prev = pred = pred.prev;

        Node predNext = pred.next;

        // 这一步过后，其它线程的对队列的操作都会略过本节点
        node.waitStatus = Node.CANCELLED;

        // 如果节点在队尾，直接从队列中移除本节点
        if (node == tail && compareAndSetTail(node, pred)) {
            compareAndSetNext(pred, predNext, null);
        } else {
            // 如果后续节点需要通知，就将前节点的 next 指向后节点
            // 否则就唤醒后节点的线程
            int ws;
            if (pred != head &&
                ((ws = pred.waitStatus) == Node.SIGNAL ||
                 (ws <= 0 && compareAndSetWaitStatus(pred, ws, Node.SIGNAL))) &&
                pred.thread != null) {
                Node next = node.next;
                if (next != null && next.waitStatus <= 0)
                    compareAndSetNext(pred, predNext, next);
            } else {
                unparkSuccessor(node);
            }

            node.next = node; // help GC
        }
    }

    /**
     * Wakes up node's successor, if one exists.
     *
     * @param node the node
     */
    private void unparkSuccessor(Node node) {
        /*
         * If status is negative (i.e., possibly needing signal) try
         * to clear in anticipation of signalling.  It is OK if this
         * fails or if status is changed by waiting thread.
         */
        int ws = node.waitStatus;
        if (ws < 0)
            compareAndSetWaitStatus(node, ws, 0);

        /*
         * Thread to unpark is held in successor, which is normally
         * just the next node.  But if cancelled or apparently null,
         * traverse backwards from tail to find the actual
         * non-cancelled successor.
         */
        Node s = node.next;
        if (s == null || s.waitStatus > 0) {
            s = null;
            for (Node t = tail; t != null && t != node; t = t.prev)
                if (t.waitStatus <= 0)
                    s = t;
        }
        if (s != null)
            LockSupport.unpark(s.thread);
    }
```


## release(int)
独占模式的资源释放方法

```java
    public final boolean release(int arg) {
        // 尝试释放资源， 子类自定义实现
        if (tryRelease(arg)) {
        // 若当前节点已经将资源全部释放，
            // 将当前头节点移出队列，
            Node h = head;
            if (h != null && h.waitStatus != 0)
                // 尝试唤醒头节点后续的可唤醒节点（线程）
                unparkSuccessor(h);
            return true;
        }

        // 当前节点未释放全部资源，
        return false;
    }

    // 子类需要实现这个方法
    // 因为是独占模式，也就是调用 release 的线程应该是已经获取到资源的独占线程,  
    // tryRelease(arg) 一般都会成功，不用考虑线程安全的问题。
    protected boolean tryRelease(int arg) {
        throw new UnsupportedOperationException();
    }
```


## acquireShared(int)
共享模式的资源获取方法


```java
    public final void acquireShared(int arg) {
        if (tryAcquireShared(arg) < 0)
            doAcquireShared(arg);
    }

    /**
     * Attempts to acquire in shared mode. This method should query if
     * the state of the object permits it to be acquired in the shared
     * mode, and if so to acquire it.
     *
     * <p>This method is always invoked by the thread performing
     * acquire.  If this method reports failure, the acquire method
     * may queue the thread, if it is not already queued, until it is
     * signalled by a release from some other thread.
     *
     * <p>The default implementation throws {@link
     * UnsupportedOperationException}.
     *
     * @param arg the acquire argument. This value is always the one
     *        passed to an acquire method, or is the value saved on entry
     *        to a condition wait.  The value is otherwise uninterpreted
     *        and can represent anything you like.
     * @return a negative value on failure; zero if acquisition in shared
     *         mode succeeded but no subsequent shared-mode acquire can
     *         succeed; and a positive value if acquisition in shared
     *         mode succeeded and subsequent shared-mode acquires might
     *         also succeed, in which case a subsequent waiting thread
     *         must check availability. (Support for three different
     *         return values enables this method to be used in contexts
     *         where acquires only sometimes act exclusively.)  Upon
     *         success, this object has been acquired.
     * @throws IllegalMonitorStateException if acquiring would place this
     *         synchronizer in an illegal state. This exception must be
     *         thrown in a consistent fashion for synchronization to work
     *         correctly.
     * @throws UnsupportedOperationException if shared mode is not supported
     */
    protected int tryAcquireShared(int arg) {
        throw new UnsupportedOperationException();
    }

    /**
     * Acquires in shared uninterruptible mode.
     * @param arg the acquire argument
     */
    private void doAcquireShared(int arg) {
        final Node node = addWaiter(Node.SHARED);
        boolean failed = true;
        try {
            boolean interrupted = false;
            for (;;) {
                final Node p = node.predecessor();
                if (p == head) {
                    int r = tryAcquireShared(arg);
                    if (r >= 0) {
                        setHeadAndPropagate(node, r);
                        p.next = null; // help GC
                        if (interrupted)
                            selfInterrupt();
                        failed = false;
                        return;
                    }
                }
                if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt())
                    interrupted = true;
            }
        } finally {
            if (failed)
                cancelAcquire(node);
        }
    }


    /**
     * Sets head of queue, and checks if successor may be waiting
     * in shared mode, if so propagating if either propagate > 0 or
     * PROPAGATE status was set.
     *
     * @param node the node
     * @param propagate the return value from a tryAcquireShared
     */
    private void setHeadAndPropagate(Node node, int propagate) {
        Node h = head; // Record old head for check below
        // 将 node 设置为 head
        setHead(node);
        // 如果资源还有余量，唤醒下一个线程
        if (propagate > 0 || h == null || h.waitStatus < 0 ||
            (h = head) == null || h.waitStatus < 0) {
            Node s = node.next;
            if (s == null || s.isShared())
                doReleaseShared();
        }
    }



    /**
     * Release action for shared mode -- signals successor and ensures
     * propagation. (Note: For exclusive mode, release just amounts
     * to calling unparkSuccessor of head if it needs signal.)
     */
    private void doReleaseShared() {
        for (;;) {
            Node h = head;
            if (h != null && h != tail) {
                int ws = h.waitStatus;
                if (ws == Node.SIGNAL) {
                    if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0))
                        continue;            // loop to recheck cases
                    // 唤醒后继线程
                    unparkSuccessor(h);
                }
                else if (ws == 0 &&
                         !compareAndSetWaitStatus(h, 0, Node.PROPAGATE))
                    continue;                // loop on failed CAS
            }
            if (h == head)                   // loop if head changed
                break;
        }
    }
```

## releaseShared(int)

```java

    /**
     * Releases in shared mode.  Implemented by unblocking one or more
     * threads if {@link #tryReleaseShared} returns true.
     *
     * @param arg the release argument.  This value is conveyed to
     *        {@link #tryReleaseShared} but is otherwise uninterpreted
     *        and can represent anything you like.
     * @return the value returned from {@link #tryReleaseShared}
     */
    public final boolean releaseShared(int arg) {
        // 如果tryReleaseShared返回true, 则解除后面一个或若干线程的阻塞状态
        if (tryReleaseShared(arg)) {
            doReleaseShared();
            return true;
        }
        return false;
    }

    /**
     * Attempts to set the state to reflect a release in shared mode.
     *
     * <p>This method is always invoked by the thread performing release.
     *
     * <p>The default implementation throws
     * {@link UnsupportedOperationException}.
     *
     * @param arg the release argument. This value is always the one
     *        passed to a release method, or the current state value upon
     *        entry to a condition wait.  The value is otherwise
     *        uninterpreted and can represent anything you like.
     * @return {@code true} if this release of shared mode may permit a
     *         waiting acquire (shared or exclusive) to succeed; and
     *         {@code false} otherwise
     * @throws IllegalMonitorStateException if releasing would place this
     *         synchronizer in an illegal state. This exception must be
     *         thrown in a consistent fashion for synchronization to work
     *         correctly.
     * @throws UnsupportedOperationException if shared mode is not supported
     */
    protected boolean tryReleaseShared(int arg) {
        throw new UnsupportedOperationException();
    }

```