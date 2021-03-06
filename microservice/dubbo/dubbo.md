Dubbo是Alibaba开源的一款RPC框架，官方架构图如下：  
![架构图](http://dubbo.apache.org/docs/zh-cn/user/sources/images/dubbo-architecture.jpg)  
dubbo各个节点作用如下，它们也是整个dubbo服务的核心：  
|节点|角色说明|
|----|-------|
|Provider|暴露服务的服务提供方|
|Consumer|调用远程服务的服务消费方|
|Registry|服务注册与发现的注册中心|
|Monitor|统计服务的调用次数和调用时间的监控中心|
|Container|服务运行容器|  

本次主要从Dubbo的以下几个模块进行源码分析：  

|模块|作用|默认实现|
|---|----|--------|
|[SPI](SPI.md)|开发者可以根据自己需要拓展dubbo组件，如自定义过滤器|无|
|[自适应扩展](Adaptive.md)|代理类需要时才被加载|无|
|[服务导出](server.md)|服务导出|Netty|
|[服务引入](client.md)|服务引入|Netty|
|[服务字典](dictionary.md)|存储服务提供者信息|RegistryDirectory|
|[负载均衡](lb.md)|负载均衡|RandomLoadBalance|
|[服务路由](route.md)|路由策略|MockInvokersSelector&&ConditionRouter|
|[服务集群](cluster.md)|容错|FailoverCluster|
|[服务编码](codec.md)|Netty解码编码|DubboCodec|
|[过滤器](filter.md)|Invoker增强|FilterChain|
|[服务调用](invoke.md)|服务调用|无|