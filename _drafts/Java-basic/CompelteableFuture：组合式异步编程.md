学习笔记：CompelteableFuture-组合式异步编程

主要内容：
- 创建异步计算， 并获取计算结果
- 使用非阻塞式操作提升吞吐量
- 设计和实现异步API
- 以异步的方式使用同步的API（同步操作异步化）
- 将多个异步操作合并为一个流水线操作
- 处理异步操作的完成状态

# 引子
近年，两种趋势推动我们不断的反思软件设计的方式。 第一和硬件平台相关，第二和程序架构相关，尤其是它们之间的交互。

硬件： 随着多核处理器的出现，提升程序处理速度最有效的方式是发挥多核的能力。通过将大型任务切分为多个子任务并行运行，这一目标是可以实现的； 我们已经可以通过直接使用线程、分支\合并框架（java1.7）、并行流（java1.8） 能以简单有效的方式来实现。

架构：互联网应用的公司API日益增长。 大厂纷纷提供了各种公共的API，如地图服务商的地理位置信息服务、 社交网络的社交信息服务、新闻服务等。现在很少有网站在独力地隔离地提供服务了， 更多的是以“混聚”的方式：使用多个来源的内容并整合在一起提供给用户。

外部网络服务响应时间相对于CPU时间来来说是极慢的。如果一个任务中以阻塞串行的方式调用多个网络服务，对CPU时钟周期来说是极大的浪费。

分支/合并框架与并行流是实现并行处理的工具；它们将操作切分为多个子操作，在多个不同的核、CPU甚至是机器上并行执行这些子操作。

有时我们真正想要是的并发而不是并行，主要目标是充分利用CPU，使其足够忙碌，从而最大化吞吐量，真正想要的是避免一些耗时操作阻塞线程的执行，而浪费计算资源。 Future 和 CompletableFuture 是处理这种情况的利器。

并行与并发的区别
```
并发
处理器核心1： --> 任务1 --> 任务2 --> 任务1 -->
处理器核心1： ------------------------------>

并行
处理器核心1： --> 任务1   ------------------->
处理器核心1： --> 任务2   ------------------->

```


# Future 接口
Future 的设计意图是为未来的结果建模。 建模一个种异步计算： 返回一个运算未来结果的引用，当运算结束后，通过这个引用将结果返回给调用方。Future 把触发了耗时操作的调用线程从等待中解放出来，让调用线程可以做一些其它有价值的工作（不依赖该耗时操作）。 就像去洗衣店洗衣衣服，店员会给一个小票并告诉我们什么时候洗好，这样洗衣服的同时我们就可以去多写几段代码。这个小票就是Future, 我们就是调用线程，洗衣店就是执行耗时操作(洗衣服)的线程。 我们写好一段代码后凭小票去洗衣店取我们的衣服（get 结果）。 Future 比 Thread 更易用，需要将耗时操作封装在 Callable 对象中提交给 ExecutorService 就OK了。
获取 Future 结果时最好使用带超时参数的get方法，这样就可以定义等待结果的最长时间而不是一直阻塞等待。


## Future 接口的局限性
仅使用Future并不能写出简洁的并发代码。比如，我们很难表述Future结果间的依赖有关系：任务A完成后， 将任务A的结果通知任务B，两个任务都完成后将结果与另一个查询操作C的结果合并。

我们期待的描述能力：
- 将两个异步计算合并为一个
- 等待Future集合中所有的任务都完成
- 只需等待Future集合中最快完成的结果
- 通过代码手动地设定一个Future的结果
- notify的方式响应Future 的完成事件

CompletableFuture 利用Java8的新特性以更直观的方式 将上述需要变为可能。 Stream 和 CompletableFuture 的设计都遵循了类似的模式：Lambda表达式和流水线的思想。 从这个角度看， CompletableFuture 和 Future 的关系就和 Stream 与  Collection 的关系一样。


## 使用 CompletableFuture 构建异步应用
要学习的几点技能
- 提供异步API
- 将同步API 封装为异步API
- 响应式的方式处理异步操作的完成事件

什么是同步API，什么是异步API
同步API就是传统的阻塞式的方法调用， 异步API会直接返回或者至少在被调用方法的计算完成之前返回，把剩下的计算放到另外一个工作线程处理（异步）-- 非阻塞式调用。工作线程会在计算完成后把结果返回给调用方。返回方式要么是通过回调函数，要么是调用方再发起一个“等待，真的到完成得到结果”的查询方法调用。

下面构建一个“最佳价格查询器”来展示CompletableFuture的用法。它会查询多个在线商店，根据给定的产品或服务找出最低的价格。

# 实现异步API
下面定义一个耗时长达一秒的 getPrice()方法。

```
public class Shop {
    public static void delay() {
        try {
            Thread.sleep(1000L);
        } catch (TnterruptedException e) {
            throw new RuntimeException(e);
        }
    }

    public double getPrice(String product) {
        return calculatePrice(product);
    }

    public double calculatePrice(String product) {
        // 人为引入一秒的延时，模拟耗时处理
        delay();
        // 模拟一个商品的价格
        return random.nextDouble() * product.charAt(0) + product.charAt(1);
    }

}
```

## 将同步方法转化为异步方法
```
public CompletableFuture<Double> getPriceAsync(String product) {
    CompletableFuture<Double> future = new CompletableFuture<>();
    new Thread(() -> {     // 新建一个线程来执行获取价格的逻辑              
            double price = getPrice(product);
            future.compelete(price);   // 设置future的返回值
        }).start();        // 启动一个新线程异步获取price

    return future;         // 不用等到计算出price 即返回一个CompletableFuture对象
}
```

## 错误处理
CompletableFuture 的 completeExceptionally()方法将异步子线程发生的异常传递给调用方。 调用方在调用get方法时会抛出同样的异常。
```
public CompletableFuture<Double> getPriceAsync(String product) {
    CompletableFuture<Double> future = new CompletableFuture<>();
    new Thread(() -> {
            try {
                double price = getPrice(product);
                future.compelete(price);
            }  catch () {
                future.completeExceptionally(e); // 将异常设置到future结果中，传递给调用方
            }          

        }).start();

    return future;
}
```

### 使用CompletableFuture的 supplyAsyncf 工厂方法
上面的例子可以使用 CompletableFuture 的工厂方法来重写。 CompletableFuture 提供了不少类似的精巧的工厂方法， 我们不用自己来实现一些模板代码。
```
public CompletableFuture<Double> getPriceAsync(String product) {
    return CompletableFuture.supplyAsync(() -> getPrice(product)); // 结合lambda 一行搞定，何止是简洁。 9行变 1行，使用的由于很充分！
}
```








