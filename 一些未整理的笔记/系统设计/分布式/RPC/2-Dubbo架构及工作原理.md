# 🙃 Dubbo 架构及工作原理

---

> 💡 Dubbo 除了能够应用在分布式系统中，也可以应用在现在比较火的微服务系统中。不过，由于 Spring Cloud 在微服务中应用更加广泛，所以，我觉得一般我们提 Dubbo 的话，大部分是分布式系统的情况。

## 1. 为什么要使用 Dubbo

随着互联网的发展，网站应用的规模不断扩大，常规的垂直应用架构已无法应对，**分布式服务架构以及流动计算架构势在必行**，亟需一个治理系统确保架构有条不紊的演进。

![](https://cs-wiki.oss-cn-shanghai.aliyuncs.com/img/20201127110803.png)

### ① 单一应用架构

当网站流量很小时，只需一个应用，将所有功能都部署在一起，以减少部署节点和成本。此时，**用于简化增删改查工作量的数据访问框架(ORM)**是关键。

### ② 垂直应用架构

当访问量逐渐增大，单一应用增加机器带来的加速度越来越小，提升效率的方法之一是**将应用拆成互不相干的几个应用，以提升效率**。此时，**用于加速前端页面开发的Web框架(MVC)**是关键。

### ③ 分布式服务架构

当垂直应用越来越多，应用之间交互不可避免，**将核心业务抽取出来，作为独立的服务，逐渐形成稳定的服务中心**，使前端应用能更快速的响应多变的市场需求。此时，**用于提高业务复用及整合的分布式服务框架(RPC)**是关键。

### ④ 流动计算架构

当服务越来越多，容量的评估，小服务资源的浪费等问题逐渐显现，此时需**增加一个调度中心基于访问压力实时管理集群容量，提高集群利用率**。此时，**用于提高机器利用率的资源调度和治理中心(SOA)**是关键。

## 2. Dubbo 架构

### ① 架构图

<img src="https://cs-wiki.oss-cn-shanghai.aliyuncs.com/img/20201204170204.png" style="zoom:67%;" />

### ② 节点角色说明

| 节点        | 角色说明                                                     |
| ----------- | ------------------------------------------------------------ |
| `Provider`  | 暴露服务的**服务提供方**                                     |
| `Consumer`  | <u>调用远程服务</u>的**服务消费方**                          |
| `Registry`  | <u>服务注册与发现</u>的**注册中心**（一般使用 **Zookeeper** 作为注册中心） |
| `Monitor`   | <u>统计服务的调用次数和调用时间</u>的**监控中心**            |
| `Container` | **服务运行容器**                                             |

对于服务提供方，它需要发布服务，而且由于应用系统的复杂性，服务的数量、类型也不断膨胀；

对于服务消费方，它最关心如何获取到它所需要的服务，而面对复杂的应用系统，需要管理大量的服务调用。

而且，对于服务提供方和服务消费方来说，他们还有可能兼具这两种角色，即既需要提供服务，有需要消费服务。

通过**将服务统一管理起来**，可以有效地优化内部应用对服务发布/使用的流程和管理。服务注册中心可以通过特定协议来完成服务对外的统一。

### ③ 调用关系说明

1. 服务容器负责启动，加载，运行服务提供者。
2. **服务提供者在启动时，向注册中心注册自己提供的服务**。
3. **服务消费者在启动时，向注册中心订阅自己所需的服务**。
4. 注册中心返回**服务提供者地址列表**给消费者，如果有变更，注册中心将基于长连接推送变更数据给消费者。
5. 服务消费者 从提供者地址列表中 基于软负载均衡算法 选一台提供者进行调用，如果调用失败，再选另一台调用。
6. **服务消费者和提供者**，在内存中累计调用次数和调用时间，**定时每分钟发送一次统计数据到监控中心**。

![](https://cs-wiki.oss-cn-shanghai.aliyuncs.com/img/20201127110649.png)

### ④ Dubbo 架构特点

Dubbo 架构具有以下几个特点，分别是连通性、健壮性、伸缩性、以及向未来架构的升级性

#### Ⅰ 连通性

- 👍 **注册中心负责服务地址的注册与查找，相当于目录服务，服务提供者和消费者只在启动时与注册中心交互，注册中心不转发请求，压力较小**
- 👍 **监控中心负责统计各服务调用次数，调用时间等，统计先在内存汇总后每分钟一次发送到监控中心服务器，并以报表展示**
- 服务提供者向注册中心注册其提供的服务，并汇报调用时间到监控中心，此时间不包含网络开销
- 服务消费者向注册中心获取服务提供者地址列表，并根据负载算法直接调用提供者，同时汇报调用时间到监控中心，此时间包含网络开销
- 👍 **注册中心，服务提供者，服务消费者三者之间均为长连接，监控中心除外**
- 👍 **注册中心通过长连接感知服务提供者的存在，服务提供者宕机，注册中心将立即推送事件通知消费者**
- 👍 **注册中心和监控中心全部宕机，不影响已运行的提供者和消费者，消费者在本地缓存了提供者列表**
- 👍 **注册中心和监控中心都是可选的，服务消费者可以直连服务提供者**

#### Ⅱ 健壮性

- 监控中心宕掉不影响使用，只是丢失部分采样数据
- 数据库宕掉后，注册中心仍能通过缓存提供服务列表查询，但不能注册新服务
- 注册中心对等集群，任意一台宕掉后，将自动切换到另一台
- 👍 **注册中心 (Zookeeper) 全部宕掉后，服务提供者和服务消费者仍能通过本地缓存通讯**
- 👍 **服务提供者无状态，任意一台宕掉后，不影响使用**
- 👍 **服务提供者全部宕掉后，服务消费者应用将无法使用，并无限次重连等待服务提供者恢复**

#### Ⅲ 伸缩性

- 注册中心为对等集群，可动态增加机器部署实例，所有客户端将自动发现新的注册中心
- 服务提供者无状态，可动态增加机器部署实例，注册中心将推送新的服务提供者信息给消费者

#### Ⅳ 升级性

当服务集群规模进一步扩大，带动IT治理结构进一步升级，需要实现动态部署，进行流动计算，现有分布式服务架构不会带来阻力。下图是未来可能的一种架构：

![](https://cs-wiki.oss-cn-shanghai.aliyuncs.com/img/20201127112104.png)

节点角色说明：

| 节点         | 角色说明                               |
| ------------ | -------------------------------------- |
| `Deployer`   | 自动部署服务的本地代理                 |
| `Repository` | 仓库用于存储服务应用发布包             |
| `Scheduler`  | 调度中心基于访问压力自动增减服务提供者 |
| `Admin`      | 统一管理控制台                         |
| `Registry`   | 服务注册与发现的注册中心               |
| `Monitor`    | 统计服务的调用次数和调用时间的监控中心 |

## 3. Dubbo 工作原理

![](https://cs-wiki.oss-cn-shanghai.aliyuncs.com/img/20201127114803.png)

图中从下至上分为十层，**各层均为单向依赖**，右边的黑色箭头代表层之间的依赖关系，**每一层都可以剥离上层被复用**，其中，Service 和 Config 层为 API，其它各层均为 **SPI**（框架接口规范）。

> 💡 比如你有个接口，现在这个接口有 3 个实现类，那么在系统运行的时候对这个接口到底选择哪个实现类呢？这就需要 SPI 了，需要**根据指定的配置**或者是**默认的配置**，去**找到对应的实现类**加载进来，然后用这个实现类的实例对象。
>
> 举个栗子:
>
> 你有一个接口 A。A1/A2/A3 分别是接口 A 的不同实现。你通过配置 `接口 A = 实现 A2` ，那么在系统实际运行的时候，会加载你的配置，用实现 A2 实例化一个对象来提供服务。

**各层说明**：

- 第一层：**service层**，接口层，给服务提供者和消费者来实现的
- 第二层：**config层**，配置层，主要是对dubbo进行各种配置的
- 第三层：**proxy层**，服务接口透明代理，生成服务的客户端 Stub 和服务器端 Skeleton
- 第四层：**registry层**，服务注册层，负责服务的注册与发现
- 第五层：**cluster层**，集群层，封装多个服务提供者的路由以及负载均衡，<u>将多个实例组合成一个服务</u>
- 第六层：**monitor层**，监控层，对 rpc 接口的调用次数和调用时间进行监控
- 第七层：**protocol层**，远程调用层，封装 rpc 调用
- 第八层：**exchange层**，信息交换层，封装请求响应模式，同步转异步
- 第九层：**transport层**，网络传输层，抽象 mina 和 netty 为统一接口
- 第十层：**serialize层**，数据序列化层，网络传输需要

![](https://cs-wiki.oss-cn-shanghai.aliyuncs.com/img/20201127115243.png)

## 📚 References

- [Apache Dubbo 官方文档](http://dubbo.apache.org/zh/docs/v2.7/user/preface/requirements/)
- [dubbo-dev-book (gitbooks.io)](https://dubbo.gitbooks.io/dubbo-dev-book/content/design.html)
- [Github - Advanced Java](https://doocs.gitee.io/advanced-java/#/./docs/distributed-system/dubbo-operating-principle)