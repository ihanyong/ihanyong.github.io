spring cloud eureka

region   zone

使用 Ribbon 实现服务调用时， 对于 Zone 的设置可以在负载均衡时实现区域亲和特性： Ribbon 的默认策略会优先访问同客户端牌一个 Zone 中的服务端实例，只有当同一个 Zone 中没有可用服务端实例的时候才会访问其他 Zone 中的实例。 所以通过 Zone 属性的定义， 配合实际部署的物理结构， 我们就可以有效地设计出对区域性故障的容错集群。


