# dubbo

## 一、高级特性

### 1.序列化

消费者和提供者传递的实体对象必须实现序列化。

### 2.地址缓存

注册中心挂了之后，服务是否可以正常访问？

可以，dubbo消费者在第一次调用时会把服务方的地址做本地缓存，后续的调用不会再访问注册中心。当服务提供者地址发生变化时会通知消费者。

### 3.超时和重试

超时：服务调用时会计时，超过时间则断开，可以在消费者端使用，@service(timeout=?)，或者在服务端进行配置，@reference(timeout=?)，同时配置时以服务端为准。

重试：为了防止网络抖动导致的超时，可以为接口配置重试次数，可以在消费者端使用，@service(retries=?)，或者在服务端进行配置，@reference(retries=?)，同时配置时以服务端为准。

### 4.多版本

灰度发布：当出现新功能的时候先让一部分用户使用新功能，当用户反馈没问题时，再让所有用户使用新功能。

使用：@service(version="v1.0")， @reference(version="v1.0")

### 5.负载均衡

#### 5.1 random随机

按照权重进行随机，@service(weight=100)，@reference(loadbalance="random")

#### 5.2 roundRobin按权重轮询

@service(weight=100)，@reference(loadbalance="roundRobin")

#### 5.3 LeastActive最少活跃调用数

服务端会记录每个服务调用的调用时间，会调用最后耗时的服务者。

#### 5.4 ConsitentHash一致性哈希

相同的参数请求总是发到相同的服务者

### 6.集群容错模式

##### 6.1 Failover Cluster

失败重试，默认值。当出现失败的时候重试其他的服务器，使用retries进行配置，一般用于读操作。

##### 6.2 Failfast Cluster

快速失败，只发起一次调用，失败立即报错，通常用于写操作。

##### 6.3 Failsafe Cluster

出现异常直接忽略，返回一个空结果

##### 6.4 Failback

失败后自动恢复，后台记录失败请求，定时重发

##### 6.5 forking

并行调用多个服务器，只要一个成功就返回

##### 6.6 broadcast

广播调用所有服务者，逐个调用，一个失败则报错。

### 7.服务降级

mock=force:return null，对该服务的方法都直接返回null，不发起远程调用，用来屏蔽不重要服务对调用方的影响。

mock=fail:return null,调用失败后返回null，不抛出异常。用来容忍不重要服务不稳定时对调用方的影响。