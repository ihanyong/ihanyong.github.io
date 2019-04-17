Flink的Batch（DataSetAPI）FaultTolerance

Flink的容错机制可以在出现故障时恢复程序并继续执行它们。这些故障包括机器硬件故障、网络故障、瞬时程序故障等。

## Batch Processing Fault Tolerance (DataSet API)

DataSet API 程序通过重试来容错。 通过execution retries parameter 来配置程序的重试次数。 0值意味没有开启容错。

通过设置大于0的会是来开启容错。 一般设置为3。

可以在代码中配置
```java
ExecutionEnvironment env = ExecutionEnvironment.getExecutionEnvironment();
env.setNumberOfExecutionRetries(3);
```


也可以在flink-conf.yaml中配置
```yaml
execution-retries.default: 3
```

## Retry Delays
重试可以配置延时。 延时重试的意思是故障出现后，并不立即进行重试，还是等到一定的延时后再重试。 

当程序与外部系统交互时，延迟重试会很有帮助，例如，连接或挂起的事务应在尝试重新执行之前达到超时。

可以在代码中配置
```java
ExecutionEnvironment env = ExecutionEnvironment.getExecutionEnvironment();
env.getConfig().setExecutionRetryDelay(5000); // 5000 milliseconds delay

```

也可以在flink-conf.yaml中配置
```yaml
execution-retries.delay: 10 s
```