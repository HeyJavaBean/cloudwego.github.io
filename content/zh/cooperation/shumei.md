---
type: docs
title: "Kitex 在数美科技可用性治理的落地实践践"
linkTitle: "数美科技"
weight: 5
---

> 
> 3月，CloudWeGo Day  邀请了贪玩游戏、数美科技、字节跳动业务部门的一线架构师，为大家分享，在Java 转 go 场景下、企业高可用性挑战等场景下，如何通过 CloudWeGo 来落地微服务架构。本文为 **数美科技架构师 代俊凯** 分享内容。
> 
> **🔗 回放链接：** https://juejin.cn/live/cloudwegoday002
> 


## 嘉宾介绍

![shumei1](/img/users/shumei/shumei1.png)

本次分享主要分为以下三部分内容：

1.  数美科技在线架构的业务挑战
1.  稳定性问题治理策略
1.  Kitex 落地实践与效果


去年，我们公司暴露了许多与可用性相关的问题，因此急需进行可用性相关治理工作。由于公司的基础设施与业界存在较大差距，我们调研并引进了 Kitex 开源解决方案，以降低研发成本。在过去一年中，我们取得了一些成果，在此借这个机会向大家分享一些相关的经验。我将针对可用性方面展开讨论，为大家提供一些可复制的参考经验。

Kitex 文档提供了很多参数设置和参考意见，同时我会提供一些参考文献链接，帮助大家了解微服务在发展过程中的沉淀下来的治理特性及其解决方案的历史。可以说，Kitex 是这些微服务治理特性的集大成者。

## 在线架构的业务挑战

### 业务挑战

首先介绍一下我们公司的业务场景，我们公司对外主要提供 SaaS 服务，我们大概有数千家客户、数十个集群，所有的客户都是使用共享的集群资源。而且我们是一个典型的机器学习系统，资源消耗比较高的是 GPU 相关的资源。

**多租户共享集群**

-   租户众多，文本、音频、图像模型预测占成本大头。大约占总成本的 80% 以上。
-   如果每个租户独立部署 GPU 集群，总体资源利用率将会很低。所以我们必然有一个选择，就是通过多个租户通过共享部署，可以共享冗余降低成本。

大部分情况下，比如在文本识别中，一条文本通常处理时间为十几毫秒左右。对于音频和图像，整理的处理时延要长很多。

-   对于音频，要经过 ASR 服务，ASR 服务对音频的响应时间通常在几十毫秒左右。
-   对于图像，会经过很多各种检测的模型，以及经过 OCR 识别出文字，再去过这些文本的风险模型，整体的延时是计算密集型的，延时比较高，图像处理通常涉及多个检测模型和OCR服务，例如OCR服务可能需要几十毫秒的时间，再经过后续不同精度的检测模型，如色情模型等， 处理时间可能会超过百毫秒。

因此，在多租户共享时，扩容速度不够快，可能会导致性能问题，并且所有风控请求都与用户行为相关，是无法缓存的。

![shumei2](/img/users/shumei/shumei2.png)

  

**多租户共享集群问题**

1.  **无法缓存**：风控请求每个都和用户行为相关，不可缓存。
1.  **性能问题**：共享硬件基础设施集群，性能问题容易导致租户间互相影响。当一个租户需要使用更多资源时候，其他租户可能受到影响，导致延迟或者性能下降。
1.  **管理维护成本**：共享数据库等基础设施，导致更高的管理成本和更复杂的系统架构。

  

**流量特征**

**1、流量整体天级别规律波**

介绍一下我们的流量特性：流量按天规律波动，有明显的波峰波谷，晚高峰用户多流量大，凌晨流量下降。

为了降低整体的成本，如果全天将这些 GPU 的机器都启动，那么每天的成本可能会较大。根据波峰波谷来启动 GPU 机器，会比全天启动所有的 GPU 机器的成本降低接近 2/ 3 。

![shumei3](/img/users/shumei/shumei3.png)

我们系统需要具备的一个功能是弹性伸缩，通过弹性伸缩，动态增加或减少系统资源来适应不同的负载需求，在高峰期，通过增加服务器实例来处理更多的请求，以防止客户的突增流量。在低峰期，通过缩减服务器实例数量来节省成本。然而，因为我们的客户数量众多，我们相当于要服务各种流量特性的用户，如果用户是互联网客户，则大部分情况下波峰波谷流量可以满足他们的需求。

**2、存在无法预测的流量突增**

流量受不同租户活动影响较大，客户业务活动或者突发的测试流量，就会出现较大的流量波动。我们很多其他的客户会进行许多活动，或者经常会发送一些测试请求，例如某段时间想将历史数据过最新的检测结果，例如对于一个网盘用户，他们经常会重新检测服务器上的文本、图像和音频，来确定当前是否满足当前监管要求，预期排除业务风险。但是，这种情况会导致我们服务器产生较大的流量波动。

因此，首先我们针对无法预测的、突增的流量进行限制，但我们的限制对于每个客户并不是一个经常动态调整的过程，而且也不会根据不同天、每天的不同时间有不同的 QPS 限制。因此，它全天都可以做到流量的突增。如果我们在低峰期预留的服务器较少，就无法处理这么多请求，造成服务器雪崩。

![shumei4](/img/users/shumei/shumei4.png)

### **微服务架构**

我们公司是一个典型的三层微服务业务架构：**接入层-逻辑业务层-基础服务层**。

我们基础服务，接入层通过 HTTP 请求接入用户的请求，内部服务使用 thrift 协议做 RPC ，进行微服务间的调用。业务层处理大部分业务逻辑，基础服务大部分是线上的机器学习预测服务，几乎都是计算密集型的服务，大部分都是 GPU 的预测服务，然后我们大概线上有数百个不同的类型的服务，以及线上数百个不同类型的服务，大概几千个实例，每天要承受接近 30 亿多的日请求量。

我们公司是 15 年左右成立的，一般认为，一个公司的基础架构需要5年进行一次更新。我们公司刚好需要进行下一次基础设施相关的更新，因为我们的基础服务架构缺少很多现在主流的治理特性。在高并发的情况下，暴露出了许多稳定性问题。因此，我们亟需一个高性能、集成多种治理特性解决稳定性问题、方便定制可以融入当前基础设施。因此，我们通过一些调研，选择了 Kitex 作为服务框架。

![shumei5](/img/users/shumei/shumei5.png)

大家可以看这张图，在引入新框架的过程中，由于我们的服务非常多且历史问题突出，我们是主要是在接入层通过替换 thrift 的客户端，在整个业务层将所有的 server 端以及 client 端都进行了替换。而基础服务由于大部分是C++服务，我们没有对其进行调整，但是因为 Kitex 的客户端支持了很多功能，已经可以解决我们很多稳定性问题。后面我们将展开介绍。

### 框架定制

CloudWeGo开源了许多相关的组件，我们使用了 Kitex 。它虽然是一个开源框架，并不像许多全家桶式地将各种服务都集成进来的框架那样，将与公司基础设施耦合较紧密的这部分作为扩展进行接入。例如并没有将字节跳动的服务发现、日志监控、tracing 和 mesh等组件直接开源出来，以拓展的方式提供支持，外部用户可以按需集成。因此，需要我们这些使用Kitex 的进行一些自我的定制，根据自己的基础设施部分自研，部分使用现成的开源拓展，为了方便内部使用还进行了易用性封装。

Kitex 有一个社区贡献的代码仓库，提供了我们开源出来的治理组件，大家可以根据其中进行选用，然后结合一部分自研，就可以搭建一套完整的微服务框架来方便使用。

下图是我们公司的一些选型，我们公司之前使用 ZooKeeper 进行服务发现，所以我们复用了原来的服务发现的机制。

![shumei6](/img/users/shumei/shumei6.png)

**基于开源组件定制**

-   DI: samber/do
-   Config: viper
-   Discovery: zookeeper
-   Logging: zap & lumberjack(rolling)
-   Metrics: prometheus
-   Tracing: opentelemetry

### 依赖注入

首先，我们需要通过比较原始日志打印以及分割和运维等处理方式，选择一些开源框架进行日志的定制，其他选用了社区比较通用常用的方案，直接进行了一些接入。

由于我们公司很多基础设施都以拓展的形式进行注入，但是我们要注入的拓展其实是非常多的，而且在不同模块间进行拓展。也就是说，比如每次开发时，都需要初始化一个客户端把所有的扩展进行注入，其实是很麻烦的。

于是，我们使用了一个依赖注入框架，该框架需要 Go 1.18 或更高版本才能使用，这个开源库要求必须支持泛型的 Golang 版本。通过这一层封装，我们可以解决不同模块间扩展组装的问题，简化组装过程。

可以看下方我提供的代码片段，演示了将这个服务发现的组件在一个公共的地方进行注册，然后其他所有需要使用该组件的地方，都可以通过单例的方式获取对应的实例，解决跨代码库跨模块依赖的问题。

![shumei7](/img/users/shumei/shumei7.png)

配置和服务发现组件共享

  

> 易用性： 使用依赖注入解决扩展组装问题，简化跨模块依赖共享 
> 依赖注入组件（支持范型，使用方便）：https://github.com/samber/do 
> 参考文献 Dependency injection ： https://en.wikipedia.org/wiki/Dependency_injection

## 稳定性问题治理策略

### 稳定性策略

我们公司使用的策略大致分为三个方面：限流、熔断和过载保护：

1.  **限流** **（Rate Limiting）** ：通过控制请求的速率来保护系统免受过多请求的技术手段。限流可以避免系统被过多的请求压垮，提高系统的稳定性和可用性。
1.  **熔断** **（Circuit Breaking）** ：通过切断不稳定服务的调用来保护系统免受服务雪崩效应的技术手段。当服务的调用出现故障时，熔断器会立即停止对该服务的调用，并通过降级处理来保证系统的可用性。
1.  **过载保护（Overload Protection）** ：通过拒绝过多的请求来保护系统免受过载的技术手段。当系统的负载达到一定程度时，过载保护会拒绝新的请求并返回错误码或者错误信息，以避免系统被压垮。

  

我们公司是一个多租户的系统，拥有上千个用户。为了针对不同的客户进行 QPS 限制，我们在服务的接入层已经做了 QPS 限制。但实际上，在用户请求进入系统时，在服务的各个层也需要进行限流的控制，以避免系统受到过多的请求影响，导致系统被过多的请求压垮。这个是一个从避免方面就可以解决系统稳定性和可用性的方式。

另外，我们的限流并不是很精准，因为客户的访问请求 QPS 并不严格等于我们下游的负载。我们针对不同 QPS 的处理方式不同，对于音频请求、文本请求或图像请求，QPS 相同的服务压力也不一样。而且，如果我们只是根据入口 QPS 的话，就无法保护整个系统。

我们进入系统后，还会结合熔断和过载保护，让系统能够自适应地解决可能带来稳定性问题的流量。因为当系统无法处理这些流量时，一个很好的解决办法是直接丢弃请求，以防止造成严重影响。

### 1、限流

首先，Kitex 本身自带了一个限流的策略，但无法满足高度定制的需求，因为需要针对不同客户的不同类型请求设置不同的QPS限制，并且需要经常变更。因为我们很多的客户一段时间流量很大，但是另外一段时间可能又切到了其他的 SaaS 供应商，这时流量又很小，所以我们要及时调整他的 QPS 限制，防止他的突增对我们系统政策带来影响。

并且这个限制还需要去根据我们自己的集群容量，去做集群整体的限制，这部分是需要我们定制一个自己方便的配置中心来做限流策略的变更。所以，这部分我们选择自研，没有直接使用 Kitex 自带的限流策略。但是，Kitex 的限流策略在扩展时具有很好的特性，可以动态地更新限流的定制，大家可以参考以下文档，具体看限流策略是如何定制的。

> 限流策略：Kitex 自带的限流策略无法满足我们高度定制的需求，我们自研了分布式限流组件解决多租户限流问题，针对不同租户不同应用定制了更细分的限流策略。
>
> 参考文档：Kitex 的限流策略定制 https://www.cloudwego.io/zh/docs/kitex/tutorials/basic-feature/limiting/

关于如何自研分布式的限流组件，不展开描述。

**限流策略**：

-   保证客户的合理 QPS 需求得到满足
-   限制突增客户的 QPS
-   优化弹性扩缩能力，提升用户体验

首先，**保证客户的合理** **QPS** **需求得到满足**。什么是客户的合理QPS？如果这个是客户，我们需要根据这个客户的流量特性来看他什么是合理的。比如很多的互联网客户，他们的流量特性有明显的波峰、波谷，所以在不同的时间的流量几乎是恒静的，除非有一些活动会产生突增的流量。这种情况下，他也会跟我们去提前报备，预留机械就可以解决临时活动产生的突增流量。

另外，我们还有很多类似网盘这样的客户，他们的流量会呈现明显的峰值和谷值。所以，我们在进行 QPS 限制时，还开发了一套系统来预测客户的流量。而且，很多客户的流量突增是有规律的，即使全天没有明显的波控和波谷，很多时候的突增也是有时间的特性。例如在周五客户端发版的时候，经常会出现流量突增，并且在在周六、周日一些节假日也会出现流量突增。我们通过针对几千家的客户学习流量的特性，并根据其特性计算系统应该保留多少合理的冗余，以满足当前客户的流量需求。

![shumei8](/img/users/shumei/shumei8.png)

我们会在这套限制的情况下，如果流量与之前预测的 QPS 不一致，我们将限制这部分突增的 QPS，但是这样用户体验是不好的，这个时候我们应该解决的问题是：根据流量反馈进行弹性扩缩，以优化用户体验并满足大部分用户的请求 QPS。如果弹性扩缩的速度越快，那这部分突增的 QPS 可以在弹性扩缩之后，取消对客户的 QPS 限制。这样可以优化用户体验，保证大部分用户的请求 QPS 可以得到满足。

**基本可用性保证**

对于基本的可用性保证，因为我们是典型的分布式系统，对于分布式的微服务来说，很多以前的做法，比如相对于单体服务来说，我们的调用方式发生了变化，变成了许多本地函数的调用，这些调用都变成了网络调用。而且，一旦变成了网络调用，就存在各种失败的可能性。

因为分布式服务与单体服务最大的区别在于，我们很多调用并不能百分之百成功，所以在整个可用性的治理上，首要的事情是针对远程的调用应该建立自己的重试机制，前提是请求是可重试的，必须保证重试是幂等的，才能定制自己的重试策略。如果请求不能做到幂等，重试策略可能不是最佳选择。这里提供一种设置重试次数的公式，假设单次请求的成功率是99%，一次重试用去100%减去两次失败的概率，系统可以在大概率下达到4个9的可用性，两次进行重试可以达到6个9的可用性。

**重试策略**

-   假设单次请求成功率 99%
-   一次重试 1-(1-99%）* (1-99%）=99.99%
-   两次重试 1-(1-99%）* (1-99%） * (1-99%） =99.9999%

> 参考文档：Kitex重试定制 https://www.cloudwego.io/zh/docs/kitex/tutorials/basic-feature/retry/

![shumei9](/img/users/shumei/shumei9.png)

另外针对延时，假设 99% 的请求都可以在 10 毫秒的情况下去完成，产生一次重试，有一部分的请求时间可能就变成了 20 毫秒。如果有两次重试，同时有一部分请求的延时就变成了 30 毫秒。

首先，重试可以提高分布式系统在多个模块配合下的可用性，因此我们可以增加重试次数。然而，在某些情况下，重试也可能是有害的。例如，当服务请求量非常大时，如果某个服务出现了大量失败，仍然进行重试，可能会导致整个服务器后端的压力瞬间增加为整个系统的一倍，而很多时候对整个系统保留的冗余不可能达到一倍，此时就会引发服务器雪崩。假设入口的请求是 600 QPS 到某一层服务，只要失败就进行重试，此时的 QPS 可能要接近 1200 QPS。此外，我们系统中处理的大部分请求可能都是有害的重试请求，客户真实的请求并没有得到有效的反馈。

### 2、熔断

**重试熔断**

Kitex 提供了一个很好的机制，叫**重试熔断**的机制，其中有两个参数：一个是重试的失败率错误阈值，目前系统框架允许的阈值范围是 10% - 30%，默认的阈值为 10%。也就是说，如果重试次数设置为 1，当遇到失败时将进行重试，此时某一层服务的流量将翻倍。如果开启了熔断重试，熔断只会增加 10% 左右的请求，因此只要某一层服务保留了至少 10% 以上的冗余服务器，就可以保证服务不发生雪崩。

![shumei10](/img/users/shumei/shumei10.png)

  


针对于重试，还有另外一个机制，即重试多次会导致延时增加，比如 99% 分位的请求耗时是 10 毫秒，针对于请求设置的超时是 500 毫秒的话，很多时候如果碰见一个超时，它将在 500 毫秒之后才进行一次重试。

但是 Kitex 带了一个功能叫 backup request，可以做到在第一次请求时，比如 99% 分位是10毫秒，如果在 10 毫秒之内没有返回结果的话，就立即去发出另一次的请求，这样不需要等到 500 秒的超时再进行重试，这是一个降低系统延时的很好方式。 2013 年， Google 发了一篇文章叫 the tail at scale，这篇文章里就详细了介绍这种机制是如何生效的，感兴趣可以了解。

> the tail at scale
>
> https://research.google/pubs/pub40801/

**服务粒度/实例粒度熔断**

针对于熔断，还提供了两种机制：服务粒度熔断和实例粒度熔断。之前在线上服务出现问题，如 ASR 服务或 OCR 服务挂了，需要手动将该服务摘除，然后等待扩容满足流量需求再将服务器重新挂回来。但是Kitex自带了一个熔断机制，可以自动解决问题，当某个服务发生过载时，它会自动摘除这个服务的调用，避免运维手动进行一些运维操作。

-   服务粒度熔断自动摘除异常服务进行系统降级。
-   实例粒度熔断统计，主要用于解决单实例异常问题，例如部分实例负载过高或者云上经常碰到的网络问题。

针对于服务粒度这种熔断，其实有的时候是比较有风险的。比如说因为一些网络的问题或者参数的设置，导致说这个服务被异常的摘掉的话，其实会导致我们大量系统失败，此时也是一种异常。

在线上，我们经常会遇到单实例问题，例如某个实例的负载过高。由于不同QPS的负载并不是完全一致的，因此我们还会遇到网络问题，例如按需机器拉起后上游网络和当前机器网络需要预热，这就会产生很多超时情况。下面是 Kitex 如何定制熔断策略以及如何设置相关参数的文档。

> 参考文档：Kitex 熔断定制 https://www.cloudwego.io/zh/docs/kitex/tutorials/basic-feature/circuitbreaker/

### 3、过载保护

**Kitex** **服务的过载保护**

**业务逻辑层大多是 IO 密集型服务，性能测试较大的开销是序列化和反序列化的开销。** 我们线上大部分服务都是成本比较高的，都是计算密集型的服务。并且业务层的服务它的单机 QPS 是相对比较高的，至少也有几百或者是上千的 QPS。我们很多时候过载保护会针对于不同的服务去使用不同的策略，比如说计算密集可能服务，可能它单击只有 50 QPS，它的过载保护设置就需要更激进一点。而像这种有几百和几千的这些服务的话，我们可以设置一些不太激进的策略。然后这里Kitex它本身提供的过载保护的机制是通过去设置一个最大连接数，或者是设置它最大的 QPS 这个机制来实现的。

但是，对于拥有大量不同服务的组织来说，这两个机制可能并不是特别好维护。设置一个最大 QPS 可能会面临一个问题，如果设置得比较小，可能会导致服务器资源的浪费。如果服务经过多次的迭代，其性能可能会下降，而最大 QPS 没有更新，则可能导致该服务产生过载的风险。因此，更好的选择可能是提供一个可以自适应过载保护的系统，而不是手动设置一个固定的 QPS 或连接数。

我们的需求是需要一个能够**自适应过载保护的系统**，不需要手动设置一个固定的 QPS 或连接数。为了实现过载保护，我们使用了另一个开源软件 Sentinel。通过使用 Sentinel，我们可以让服务器最大限度地利用系统的资源，承受更多的吞吐量。

**过载保护目标：**

-   保证系统不被拖垮
-   在系统稳定的前提下，保持系统的吞吐量

![shumei11](/img/users/shumei/shumei11.png)

> 参考文档：https://sentinelguard.io/zh-cn/docs/golang/system-adaptive-protection.html

在处理线上问题时，第一步是找到一种应急措施，以确保客户的请求得到满足。例如，当服务器产生大量 5XX 服务时，可以采取一些措施来缓解服务压力。此外，当服务不正常时，需要及时处理，避免影响客户的使用体验。对于 ChatGPT 服务，可能会遇到服务过载的情况，此时需要采取相应的降级措施。

我们出现问题的时候，分布式的系统一定要有超时这个机制来保护大家。另外一个当出现问题的时候要及时地进行线上系统的降级。为什么说像 ChatGPT 或者是说我们现在这些 AI 的服务经常会提示说我现在某个功能不可用？这很大一部分原因是我们过去的服务对整个算力的要求其实是比较低的，但是像我们现在这些机器学习的系统，它算力的要求非常高，很多时候我们是无法满足峰值的资源需求。

我们是去云厂商那里租用GPU，当云厂商的资源可能无法满足我们如此高的并发流量的时候，我们只能选择降低服务等级，只服务于某些重要的客户，或者与客户商量，将流量切换到其他客户，并且同时尝试购买有余量集群的资源，并且做多机房切换求。例如，目前新出的GPT服务GPU的消耗非常高，且单卡无法进行Inference，需要使用多卡。如果全球所有用户同时涌入该系统，则很难满足这种系统的需要。相比之下，传统的算力需求较小的服务，如IO密集型服务，对资源的消耗相对较低，可能相对容易的在集群中快速增加 计算和存储资源，可以为全部用户提供高质量的服务，服务的可用性虽然可能存在问题，但持续时间较短。

对于机器学习服务，由于难以协调大量的 GPU计算资源。一些特殊时期，这时我们去所有的云厂商都买不到足够多的服务器，只能对整个服务进行降级，只满足部分服务的用户的需求。有关过载保护相关的工作，也是为了在服务器发生一些资源不足的时候，首先过载保护第一件事尽可能在我们资源的利用率更高的情况下，保证服务不会出现一些事故。

对于 C++的 GPU 的服务，我们希望它们能达到相同的效果，但它们可能本身能承受的 QPS 就比较小，稍微多一些请求就会导致很多的问题。就像我这里的 IO 密集型服务的过载保护和 CPU 密集型过载保护一样，它们实现的区别就在于它们俩的负载能承受的范围，其实承受的范围的波动差距非常大。像我们这种 IO 密集型的服务，QPS 的范围可能是几千，波动个几百对它影响不是特别大。但是对于 GPU 的服务，它本身可能连 100 QPS 都承受不到，稍微多 10 QPS，立马 GPU 的队列就堵住了，就基本就挂掉了。

为了避免服务过载，我们需要一些更精细的机制来控制它的过载保护。对于IO密集型服务，通过选型我们使用 Sentinel 系统自适应流控从整体维度对应用入口流量进行控制，结合系统的 Load、CPU 使用率以及应用的入口 QPS、平均响应时间和并发量等几个维度的监控指标，通过自适应的流控策略，让系统的入口流量和系统的负载达到一个平衡，让系统尽可能跑在最大吞吐量的同时保证系统整体的稳定性。

![shumei12](/img/users/shumei/shumei12.png)

例如，假设一个请求的平均响应时间是20毫秒，我们的QPS大概是100QPS。但是对于像我们的C++类服务来说，我们需要一个更精细的机制来快速响应过载保护。

### 队列

在微服务系统中，我们通常存在一个无处不在的队列，无论是网卡、CPU，还是RPC服务器，都需要存在于该队列中。该队列的响应时间等于排队时间加上服务响应时间，如果请求可以立即处理，则几乎不存在排队时间。但如果下游请求不能及时响应，则需要在队列中等待处理。

> 响应时间 = 排队时间 + 服务时间 Response Time = Queue Time + Process Time

![shumei13](/img/users/shumei/shumei13.png)

**Thrift** **（C++）/ Fbthrift 服务的过载保护**

> 基础服务层大多是计算密集型服务，CPU 利用率等不完全反应程序的负载，服务过载可能是 GPU 负载过高。 使用请求在队列中的平均等待时间（“queue time”）来计算服务的负载。之所以不选择请求的平均处理时间（“procss time”），是为了去除下游服务调用的影响。有时 process time 的增加并不代表当前服务过载了，而是请求依赖的下游服务过载。另一方面，请求的 process time 增加到一定程度，当前服务的资源也会逐渐耗尽，最终反应在 queue time 的增加上。

我们可以看下图，当我们整个 CPU 或者是 GPU 整体的负载较高的时候，它的排队时间就会去无限长。所以这里给我们一种启发的机制，也就是说我们设置一个最长的队列等待时间。因为刚才也说了，在整个微服务的框架下，我们一定要干的一件事情就是一定要有超时的机制，因为没有超时的机制最终可能导致整个系统都被焊住。所以我们这里针对于这个 Queue Time设置一个 Timeout 的话，那就可以去保证我们这些计算密集能服务在它的排队时间，等到一定时间的时候我们就立即丢弃请求。

![shumei14](/img/users/shumei/shumei14.png)
  


> 相关理论可以参考排队论 https://mp.weixin.qq.com/s/YSvvLhBYvif5ykqBXYY5xA https://arthurchiao.art/blog/traffic-control-from-queue-to-edt-zh/

但是我们实践了这种固定的 Queue timeout 的情况下，其实产生了另外一个问题，就是我们设置固定的 Queue timeout，如果设置过大，会发现保护不了我们的系统；如果设置过小，偶尔的 CPU 突增又会导致请求丢弃。我们之前实验了这个设置，我们请求是稳定地发 QPS，但是每隔一段时间会起一些计算密集的服务，会产生 CPU 的扰动。这个时候我们就会发现，只要 CPU 产生了突增，它就会立即丢弃请求，也就是说它这个时候很难去抢占 CPU 的资源，导致很多请求在队列中排队。

但因为只是在某个时间段出现突发的流量增加，并不能代表整个服务器的负载，所以这种方式作为一种负载保护机制，可能会导致我们丢弃许多请求。为了解决这个问题，我们进行了一些调研，并使用了 C ++ 使用的Fbthrift 方案中的启发式算法。如果队列在过去很长时间没有被清空，就将 Queue timeout 设置得更短；如果队列一直没有堆积，则可以将 Queue timeout 设置得更长。我们比较了这两种机制，之前其实参考了微信的 Queue time 设置时间，他们的超时是 500 毫秒， Queue time 设置为 20 毫秒，可以保证大多数情况下，在服务器发生过载的时候尽快地丢弃请求。

![shumei15](/img/users/shumei/shumei15.png)

Control Delay

> 相比固定的 queue timeout，可自适应的调整 queue timeout。 上图是设置固定的较短 queue timeout（参考微信的20ms），小幅的 CPU 抖动都将导致请求失败，但服务并不是持续的过载。

我们把 Queue timeout 设置了一个相对长的时间，靠自适应的 queue timeout 机制，会在我们服务器的队列，比如在 200 毫秒以内没有被清空的时候，就会把 queue timeout 设置为一个更短的时间，达到可以快速地丢弃请求的机制。

![shumei16](/img/users/shumei/shumei16.png)

> 一般来说服务器会有内存或者资源池数量的限制，并将没有来得及处理的请求放在缓冲区。一旦处理请求的速度跟不上到来的请求，队列将会越来越大并且最终超过使用闲置。 Facebook 根据 CoDel 的启发设计了一套算法：当内存缓冲队列在过去的N毫秒内没有被清空，则 queue 中请求的 timeout 则被设置为M毫秒（一般为10-30ms）；(Optional) 当内存缓冲队列在过去的N毫秒内被清空，则 queue 中请求的 timeout 被设置成N毫秒。

另外，大部分的系统其实都是有一个消费队列的，比如很多 RPC 的框架，可能是一个多生产者单消费者的队列，也有一些框架是一个多生产者、多消费者的队列，然后很多请求一定会在队列中去堆积。比如 Kitex ，其实之前它有宣称自己平均处理时间的时候，其实就是处理的延时以及长尾的延时，就是针对于这种多生产者、多消费者或者多生产者单消费者的队列，它降低延时的一个方式就是当有多个队列的时候，尽快地去消费请求，让大家在队列里堆积的时间足够短，因为一个请求实际处理的时间并不完全等于他处理的时间，还有他在队列中等待的时间。

这些 RPC 框架通过优化请求在队列中等待的时间，可以优化实际处理的延时。我们使用的 C++ 框架可以实现先进先出的请求模式，即使在请求堆积的情况下，也会将整个队列变为后进先出。这意味着新到来的请求会被尽快处理，而堆积的请求则可以在这些队列清空之后才进行处理。结合自适应 queue timeout机制，可以解决整体请求成功率的问题。

我们可以想象一种场景，即如果我们的 GPU 服务处理时延很长，当请求在队列中堆积时间过长时，可能会导致上游超时，此时应该立即将请求丢弃，而不是让堆积的请求去消耗 GPU 资源。此外，可以参考 “Fail at Scale” 这篇文章，了解 queue 时间的机制。对于 Facebook 发布的 fbthrift 框架来说，网上很难找到很多资料，但通过实践和线上问题，找到了这两个参数的设置，即codel_enabled=true和 queue_timeout，可以自动实现自适应的机制。通过开启 gflag，并设置合理的 queue timeout，可以保护 GPU 服务免受过载。

![shumei17](/img/users/shumei/shumei17.png)
Adaptive LIFO

> 大部分系统处理请求遵循 FIFO (First In Last Out) 原则。在峰值流量太大时，后来的请求可能会因为先来请求的阻塞而导致请求耗时更长。对此 Facebook 提出的方案是 adaptive LIFO (Last In First Out) ，当系统出现队列请求积压的时候，将队列模式自动切换为 LIFO，后到的请求首先执行，最大限度上增加了请求成功的可能性。 Adaptive LIFO 与 CoDel 能够非常好的兼容，如下图所示。CoDel 设置较短 timeout，防止队列积压过多请求，adaptive LIFO 将后入的请求率先处理，最大限度增加请求成功的概率。

## Kitex落地实践与效果

### 效果对比

另外，是我们上线了 Kitex 以及对比上线前后的效果。可以看到，上图是上线之前，因为我们经常有各种各样的突增流量，突增流量的影响会持续一段较长的时间。

![shumei18](/img/users/shumei/shumei18.png)

但是换了 Kitex 以及结合之前对服务的保护策略之后，发现可能只有在突增的一刹那会有一部分的失败，但是它的影响的范围已经被缩到了很小。

可用性图是对数的坐标，为了让大家比较明显地看出来失败率，实际失败率其实并不高，实际失败率小于千分之一。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/6c86d33220b84693a556b9c67861aa58~tplv-k3u1fbpfcp-zoom-1.image)

> **可用性对比**：
>
> 上图对比为 Kitex 调用 C++ 基础服务服务失败数/成功数监控。 Kitex 框架可以很好的解决弹性扩缩，以及机器负载导致的可用性问题，将影响都限定在很小范围内。

### 相似业务的参考意见

-   **做好客户** **QPS** **的限制和集群预留足够的冗余。** 对于具有多租户的 SaaS 系统，一定要做好客户 QPS 的限制，因为资源是有限的，不可能给每个客户都预留足够多的 QPS 这种限额，因为实在是太贵了。此外，要预留足够的冗余，但是在平衡成本时，需要相对精确地预测客户的流量。
-   **请求处理速率跟不上请求到达速率，尽早丢弃请求。** 在整个微服务架构下，处理请求的哲学。
-   **选择合适的熔断** **、过载保护策略。** Kitex 提供了很好的熔断过载保护的机制，用户可以选择自己熔断或过载保护的策略。流量是检查效果的唯一标准。经过足够多流量验证的 categ 真是非常好的选择，因为像熔断、国宅保护等这些机制，有很多边界case，需要将这些边界case都通过自己的流量去验证。
-   **流量是检验效果的唯一标准，经过足够多流量验证的** **Kitex** **是很好的选择。** 在字节跳动，经过了足够多的验证，很多边界 case 已经解决了，所以我们选择 Kitex 开源框架一定是非常好的选择。

> Kitex Github：https://github.com/cloudwego/kitex
