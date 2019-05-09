---
layout: post
title:  "【todo】【Java基础知识梳理】AQS源码阅读分析"
date:   2019-05-09 12:30:00 +0800
tags:
        - 技术
        - java
---
说到并发， 不能不深入了解一下 AbstractQueuedSynchronizer （AQS）。
ReentrantLock， ReentrantReadWriteLock， Semaphore，CountDownLatch 都是基于AQS实现的。 
其核心是基于一个先进先出(FIFO)的等待队列和一个表示状态的支持原子操作的整数值。 子类需要实现几个方法来改变这个状态，来表达资源的占用和释放。


# 代码框架
## AQS类的组成
AQS的主要组件是 一个volatile int 类型的 state, 和一个Node的双向链表。

### State
AbstractQueuedSynchronizer 定义了一个 **volatile int state** 来表示资源。 在不同的实现中对state的定义可能不一样，比如 ReentrantLock中，用state为0来判断锁是否被占用， 如果为0说明没有线程占用锁。 state 只能通过下面三个方法来进行访问：
- getState()： 获取当前state的值
- setState(int)： 直接设置state值， 一般在能保证没有线程并发修改的情况下可以使用这个方法
- compareAndSetState(expect, update)： CAS的方式修改State, 一般用于多个线程并发获取资源时使用

在自定义同步器中只需要实现下面几个方法，使用上面三个state访问方法来实现资源的获取与释放。 对资源的占用分为两种模式：
独占模式
- tryAcquire(int)： 尝试以独占的方式获取资源。 如果获取失败，调用方可以将当前线程推入资源等待队列，直到其它线程释放资源时通知这个线程。
- tryRelease(int)： tryAcquire的反向操作。 返回一个boolean, true表示已经完全释放了占用的资源，其它等待线程可以尝试获取资源；如果为false，表示还没有将资源完全释放。

共享模式
- tryAcuqireShared(int)： 尝试以共享的模式获取资源。 返回负数：失败； 0：表示获取成功，但没有剩余可用的资源供其它线程获取了；正数： 表示获取成功，并且还有剩余的资源供其它的线程获取。
- tryReleaseShared(int)： 尝试释放资源， tryAcuqireShared的反向操作。 

其它
- isHeldExclusively(): 判断资源是否被当前线程独占。只在 condition中使用。　如果不使用condition就不需要实现这个方法。

以 ReentrantLock 中的非公平锁的实现为例:
```java
        protected final boolean tryAcquire(int acquires) {
            return nonfairTryAcquire(acquires);
        }
        final boolean nonfairTryAcquire(int acquires) {
            final Thread current = Thread.currentThread();
            int c = getState(); // 获取当前资源状态
            // 如果为0，没有其它线程占用资源（锁）
            if (c == 0) { 
                 // 有过CAS的方式占用资源（获取锁）
                if (compareAndSetState(0, acquires)) {
                    setExclusiveOwnerThread(current);
                    return true;
                }
            }
            // 如果已经资源已经被占用，判断是不是当前线程占用的（可重入锁）
            else if (current == getExclusiveOwnerThread()) {
                int nextc = c + acquires;
                if (nextc < 0) // overflow
                    throw new Error("Maximum lock count exceeded");
                // 如果是当前线程占用的锁，可直接使用setState方式来更新state(获取锁的次数)
                // 注意，state 的数值是多少，之后就要release 多少次，state为0时才算是锁被当前线程释放
                setState(nextc);
                return true;
            }
            return false;
        }

        protected final boolean tryRelease(int releases) {
            int c = getState() - releases;
            if (Thread.currentThread() != getExclusiveOwnerThread())
                throw new IllegalMonitorStateException();
            boolean free = false;
            if (c == 0) {
                free = true;
                setExclusiveOwnerThread(null);
            }
            setState(c);
            return free;
        }
```

### Node
AbstractQueuedSynchronizer 维护了一个是一个双向链表队列。节点是内部类 Node 来定义的。
AbstractQueuedSynchronizer 中保存了对链表的 head tail 引用。

等待队列是"CLH" (Craig, Landin, and Hagersten)的一个变种。 CLH 一般用于自旋锁。 

Node 中的主要属性为：
- volatile int waitStatus： 这个值在使用时一般只需要确定它的范围值。负数值表示节点不需要通知，所以多数代码不需要去判断这个值的精确值。对于同步节点初始值是0，conditio节点是 CONDITION(-2)。使用CAS的方式来修改。取值范围是：[CANCELLED(1),SIGNAL(-1),CONDITION(-2),PROPAGATE(-3),0]
- volatile Node prev： 指向前一个节点
- volatile Node next： 指向下一个节点。  为null时当前节点不一定为tail。 所以当next为null时，可能需要从tail向前遍历。 为了便于isOnSyncQueue处理，取消节点的next会指向自己。
- volatile Thread thread： 绑定到当前节点的线程。构建时初始化，使用后设置为null
- Node nextWaiter： 指向在condition上等待的节点，或一个特定的SHARED值。 只有在独占模式下才可能会有条件队列。 


Node 中的状态waitStatus的取值：
- SIGNAL ： 本节点的后续节点是被阻塞的，当前节点释放资源或者取消时，必须唤醒后续节点。 其实就是标识一个节点是否为活跃状态。为了避免并发问题，获取方法必须要先设置 SIGNAL。然后再重试原子性的获取，再次获取失败后才进入阻塞。
- CANCELLED ： 本节点关联的线程因为超时或中断而取消。这是一个终止状态，不能再转变为其它状态。 取消后，其关联的线程不再被阻塞。
- CONDITION ： 此节点当前在条件队列中。 当在其所等待的condtion 上调用了single()方法后， 节点从条件队列移到同步队列中去，参与资源的获取
- PROPAGATE ： 在共享模式中标识节点处于可运行状态。
- 0 : 初始化状态。

头节点永远不会是取消节点的，因为只有获取资源成功的节点才能成为一个头节点。 取消的线程不可能成功获取到资源， 且线程只能取消自己所在的节点，不能取消其它节点。


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


![AQS结构示意图](http://pr31ptshd.bkt.clouddn.com/image/java_basic/AQS_structure.JPG)

# AQS代码框架详解

下面以 acquire，release, acquireShared, releaseShared 的顺序来详细看一下实现的源码。 主要在代码中进行注释的方式来说明。


## 独占模式





### acquire
acquire方法流程图
![acquire流程图](http://pr31ptshd.bkt.clouddn.com/image/java_basic/aqs-exclusive-accquire-flow.png)

acquireQueued方法流程图
![acquireQueued流程图](http://pr31ptshd.bkt.clouddn.com/image/java_basic/aqs-exclusive-acquireQueued-flow.png)


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
             * 此时本节点不能直接进入waiting 状态，需要调用者应该再尝试一次获取资源
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
     * 唤醒后续节点（如果有的话）
     *
     * @param node the node
     */
    private void unparkSuccessor(Node node) {
        /*
         * 如果状态是负数（需要通知），尝试把通知状态给清除掉。 失败也没有关系~
         */
        int ws = node.waitStatus;
        if (ws < 0)
            compareAndSetWaitStatus(node, ws, 0);

        /*
         * 一般来说需要唤醒的下一个线程是next节点。
         * 如果下一个节点是取消节点或者是null， 
         * 就需要从tail 开始向前找到第一个未取消节点作为下一个唤醒的线程。
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


### release(int)
独占模式的资源释放方法

![流程图](http://pr31ptshd.bkt.clouddn.com/image/java_basic/aqs-exclusive-release-flow.png)

```java
    public final boolean release(int arg) {
        // 尝试释放资源， 子类自定义实现
        if (tryRelease(arg)) {
        // 若当前节点已经将资源全部释放，
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




## 共享模式

### acquireShared(int)
共享模式的资源获取方法


```java
    public final void acquireShared(int arg) {
        // 先尝试直接获取所需资源
        if (tryAcquireShared(arg) < 0)
            // 如果不能直接获取到资源，排队获取
            doAcquireShared(arg);
    }

    private void doAcquireShared(int arg) {
        // 加入等待队列
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
                if (shouldParkAfterFailedAcquire(p, node) && // 判断是否可以直接进入休眠
                    parkAndCheckInterrupt()) // 挂起线程并在唤醒时检查中断标志
                    interrupted = true;
            }
        } finally {
            if (failed)
                cancelAcquire(node);
        }
    }


    private void setHeadAndPropagate(Node node, int propagate) {
        Node h = head; // Record old head for check below
        // 将 node 设置为 head
        setHead(node);
        // 如果资源还有余量，唤醒下一个线程来获取资源
        if (propagate > 0 || h == null || h.waitStatus < 0 ||
            (h = head) == null || h.waitStatus < 0) {
            Node s = node.next;
            if (s == null || s.isShared())
                doReleaseShared();
        }
    }


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



### releaseShared(int)

```java

    public final boolean releaseShared(int arg) {
        // 如果tryReleaseShared返回true, 则解除后面一个或若干线程的阻塞状态
        if (tryReleaseShared(arg)) {
            doReleaseShared();
            return true;
        }
        return false;
    }

    protected boolean tryReleaseShared(int arg) {
        throw new UnsupportedOperationException();
    }

```



## 子类的实现

### 各方法实现时的注意点
### 实现示例