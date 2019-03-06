---
layout: post
title:  "【日常学习笔记】Java的fork-join的用法"
date:   2019-03-06 22:30:00 +0800
tags:
        - 技术
        - java
        - 并发编程
---

Fork/join 本质上是一个并行版分治法实现。 将一个大任务递归地拆解为若干个小的子任务进行并行计算，在子任务完成后将子任务的结果合并为父任务的结果。 也就是说比较适合那些可以递归处理的任务。

一般的代码套路如下：

```
Result solve (Problem problem) {
    if ( problem is small) {
        solve problem
     else {
        *split* proble into independent parts
        *fork* new subtasks to solve each part
        *join* all subtasks
        *compose* result form subresults
    }
}
```


# 工作窃取

线程池里每个线程对应一个工作队列， 一个线程只处理自己队列的任务。 如果有的线程队列处理较快，把自己的队列的任务处理完毕之后就会空闲下来。而其它的线程处理较慢还有任务积压。这样空闲的线程就是一种浪费。  工作窃取就是让空闲了的线程从别的线程工作队列中窃取任务进行处理，既能充分利用线程也能提高整体的处理速度~

fork/join 的线程池为 ForkJoinPool。 可以通过构造方法new 出来，也可以使用Executors.newWorkStealingPool()来创建（返回为 ExecutorService， 需要的话自己 cast 为 ForkJoinPool）。  通过API可以看出来， ForkJoinPool也可以单纯地做为一个带有工作窃取特性的线程池来用。


# 任务 ForkJoinTask 
我们需要把任务封装为一个 ForkJoinTask 来提交给 ForkJoinPool。 但一般不用直接继承ForkJoinTask 来写。 如果需要有返回值的话直接继承 RecursiveTask ， 没有返回值直接继承 RecursiveAction。


# 示例
如下例子可以看就是把一个递归方法给并行化了。

1. 没有引入delay 的情况下， 反而是单线程处理的快。（fork join 引入了其它开销，创建线程，切换上下文……）
2. 引入delay的情况下（模拟耗时操作如IO）， fork join 对整体处理速度的提升还是可观的，但与线程数和任务数也有关系
3. 写多线程的代码不能太凭直觉， 最好有基准来做对比参考~


```java

/**
 * ForkJoinExample
 * 通过forkjoin实现 1+2+3+…+n 求和， 并和最基本单线程方式做对比
 * @author yong.han
 * 2019/3/6
 */
public class ForkJoinExample {

    public static void testForkJoin(long begin, long end, long threshold) {

        ForkJoinPool forkJoinPool = (ForkJoinPool) Executors.newWorkStealingPool();

        ForkJoinTask<Long> task = new CountTask(begin, end, threshold);

        long start = System.currentTimeMillis();
        // 提交任务
        forkJoinPool.submit(task);
        // 获取结果
        long result = task.join();

        long stop = System.currentTimeMillis();
        System.out.println("forkjoin ("+threshold+ ") cost time is " + (stop - start) + "ms");
        System.out.println(result);

    }

    // 继承 RecursiveTask 来返回一个结果
    public static class CountTask extends RecursiveTask<Long> {
        private final long begin;
        private final long end;
        private final long threshold;
        public CountTask(long begin, long end, long threshold) {
            this.begin = begin;
            this.end = end;
            this.threshold = threshold;
        }
        @Override
        protected Long compute() {
            if (end - begin <= threshold) {
                // 任务量小于指定值时直接求结果
                long r = 0;
                for (long i = begin; i < end; i++) {
                    r += i;
                    delay();
                }
                return r;
            } else {
                // 任务量大于指定值时(递归地)分解任务
                long mid = (begin + end) / 2;
                ForkJoinTask<Long> left = new CountTask(begin, mid, this.threshold);
                ForkJoinTask<Long> right = new CountTask(mid, end, this.threshold);
                // fork 子任务
                left.fork();
                right.fork();

                // 等待子任务结果并合并结果
                long lr = left.join();
                long rr = right.join();
                return lr + rr;
            }
        }
    }


    /**
     * 单线程版本
     * @param begin
     * @param end
     * @return
     */
    public static long basicImpl(long begin, long end) {
        long start = System.currentTimeMillis();

        long r = 0;
        for (long i = begin; i < end; i++) {
            r += i;
            delay();
        }
        long stop = System.currentTimeMillis();
        System.out.println("basic cost time is " + (stop - start) + "ms.");
        System.out.println(r);
        return r;
    }


    /* 通过Thread.sleep 模拟耗时操作， 为0不sleep   */
    public static long delay = 5;

    /**
     *
     *
     * @param args
     */
    public static void main(String[] args) {
        long count = 1_000;

        //
        for (int i = 0; i < 10; i++) {
            System.out.println("========================================");
            testForkJoin(1, count+1, 10);
            testForkJoin(1, count+1, 50);
            testForkJoin(1, count+1, 1_000);
            testForkJoin(1, count+1, 10_000);
            basicImpl(1, count+1);
        }
    }

    public static void delay() {
        if (delay>0) {
            try {
                Thread.sleep(10);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }

```


