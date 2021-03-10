# Sentinel 与 Hystrix 的对比

> 转载：[Sentinel 与 Hystrix 的对比](https://sentinelguard.io/zh-cn/blog/sentinel-vs-hystrix.html)

[Sentinel](https://github.com/alibaba/Sentinel) 是阿里中间件团队开源的，面向分布式服务架构的轻量级高可用流量控制组件，主要以流量为切入点，从流量控制、熔断降级、系统负载保护等多个维度来帮助用户保护服务的稳定性。大家可能会问：Sentinel 和之前常用的熔断降级库 Netflix Hystrix 有什么异同呢？本文将从多个角度对 Sentinel 和 Hystrix 进行对比，帮助大家进行技术选型。

## 1. Overview

先来看一下 [Hystrix](https://github.com/Netflix/Hystrix/wiki) 的官方介绍：

> Hystrix is a library that helps you control the interactions between these distributed services by adding latency tolerance and fault tolerance logic. Hystrix does this by isolating points of access between the services, stopping cascading failures across them, and providing fallback options, all of which improve your system’s overall resiliency.

可以看到 Hystrix 的关注点在于以 *隔离* 和 *熔断* 为主的容错机制，超时或被熔断的调用将会快速失败，并可以提供 fallback 机制。

而 Sentinel 的侧重点在于：

- 多样化的流量控制
- 熔断降级
- 系统负载保护
- 实时监控和控制台

可以看到两者解决的问题还是有比较大的不同的，下面我们来分别对比一下。

## 2. 共同特性

### 2.1 资源模型和执行模型上的对比

Hystrix 的资源模型设计上采用了命令模式，将对外部资源的调用和 fallback 逻辑封装成一个命令对象（`HystrixCommand` / `HystrixObservableCommand`），其底层的执行是基于 RxJava 实现的。每个 Command 创建时都要指定 commandKey 和 groupKey（用于区分资源）以及对应的隔离策略（线程池隔离 or 信号量隔离）。线程池隔离模式下需要配置线程池对应的参数（线程池名称、容量、排队超时等），然后 Command 就会在指定的线程池按照指定的容错策略执行；信号量隔离模式下需要配置最大并发数，执行 Command 时 Hystrix 就会限制其并发调用。

Sentinel 的设计则更为简单。相比 Hystrix Command 强依赖隔离规则，Sentinel 的资源定义与规则配置的耦合度更低。Hystrix 的 Command 强依赖于隔离规则配置的原因是隔离规则会直接影响 Command 的执行。在执行的时候 Hystrix 会解析 Command 的隔离规则来创建 RxJava Scheduler 并在其上调度执行，若是线程池模式则 Scheduler 底层的线程池为配置的线程池，若是信号量模式则简单包装成当前线程执行的 Scheduler。而 Sentinel 并不指定执行模型，也不关注应用是如何执行的。Sentinel 的原则非常简单：根据对应资源配置的规则来为资源执行相应的限流/降级/负载保护策略。在 Sentinel 中资源定义和规则配置是分离的。用户先通过 Sentinel API 给对应的业务逻辑定义资源（埋点），然后可以在需要的时候配置规则。埋点方式有两种：

- try-catch 方式（通过 `SphU.entry(...)`），用户在 catch 块中执行异常处理 / fallback
- if-else 方式（通过 `SphO.entry(...)`），当返回 false 时执行异常处理 / fallback

Sentinel 还支持基于注解的资源定义方式，可以通过 `@SentinelResource` 注解参数指定异常处理函数和 fallback 函数。

```java
// 原本的业务方法.
@SentinelResource(fallback = "fallbackForGetUser")
User getUserById(String id) {
    throw new RuntimeException("getUserById command failed");
}

// fallback 方法，原方法出现异常时被调用（无论是业务异常还是限流异常）.
User fallbackForGetUser(String id) {
    return new User("admin");
}
```

从 0.2.0 版本开始，Sentinel 引入异步调用链路支持，可以方便地统计异步调用资源的数据，维护异步调用链路，同时具备了适配异步框架/库的能力。可以参考 [相关文档](https://github.com/alibaba/Sentinel/wiki/如何使用#异步调用支持)。

Sentinel 提供[多样化的规则配置方式](https://github.com/alibaba/Sentinel/wiki/动态规则扩展)。除了直接通过 `loadRules` API 将规则注册到内存态之外，用户还可以注册各种外部数据源来提供动态的规则。用户可以根据系统当前的实时情况去动态地变更规则配置，数据源会将变更推送至 Sentinel 并即时生效。

### 2.2 隔离设计上的对比

隔离是 Hystrix 的核心功能之一。Hystrix 提供两种隔离策略：线程池隔离（Bulkhead Pattern）和信号量隔离，其中最推荐也是最常用的是线程池隔离。Hystrix 的线程池隔离针对不同的资源分别创建不同的线程池，不同服务调用都发生在不同的线程池中，在线程池排队、超时等阻塞情况时可以快速失败，并可以提供 fallback 机制。线程池隔离的好处是隔离度比较高，可以针对某个资源的线程池去进行处理而不影响其它资源，但是代价就是线程上下文切换的 overhead 比较大，特别是对低延时的调用有比较大的影响。

但是，实际情况下，线程池隔离并没有带来非常多的好处。首先就是过多的线程池会非常影响性能。考虑这样一个场景，在 Tomcat 之类的 Servlet 容器使用 Hystrix，本身 Tomcat 自身的线程数目就非常多了（可能到几十或一百多），如果加上 Hystrix 为各个资源创建的线程池，总共线程数目会非常多（几百个线程），这样上下文切换会有非常大的损耗。另外，线程池模式比较彻底的隔离性使得 Hystrix 可以针对不同资源线程池的排队、超时情况分别进行处理，但这其实是超时熔断和流量控制要解决的问题，如果组件具备了超时熔断和流量控制的能力，线程池隔离就显得没有那么必要了。

Hystrix 的信号量隔离限制对某个资源调用的并发数。这样的隔离非常轻量级，仅限制对某个资源调用的并发数，而不是显式地去创建线程池，所以 overhead 比较小，但是效果不错，也支持超时失败。Sentinel 可以通过并发线程数模式的流量控制来提供信号量隔离的功能。并且结合基于响应时间的熔断降级模式，可以在不稳定资源的平均响应时间比较高的时候自动降级，防止过多的慢调用占满并发数，影响整个系统。

### 2.3 熔断降级对比

Sentinel 和 Hystrix 的熔断降级功能本质上都是基于熔断器模式（Circuit Breaker Pattern）。Sentinel 与 Hystrix 都支持基于失败比率（异常比率）的熔断降级，在调用达到一定量级并且失败比率达到设定的阈值时自动进行熔断，此时所有对该资源的调用都会被 block，直到过了指定的时间窗口后才启发性地恢复。上面提到过，Sentinel 还支持基于平均响应时间的熔断降级，可以在服务响应时间持续飙高的时候自动熔断，拒绝掉更多的请求，直到一段时间后才恢复。这样可以防止调用非常慢造成级联阻塞的情况。

### 2.4 实时指标统计实现对比

Hystrix 和 Sentinel 的实时指标数据统计实现都是基于滑动窗口的。Hystrix 1.5 之前的版本是通过环形数组实现的滑动窗口，通过锁配合 CAS 的操作对每个桶的统计信息进行更新。Hystrix 1.5 开始对实时指标统计的实现进行了重构，将指标统计数据结构抽象成了响应式流（reactive stream）的形式，方便消费者去利用指标信息。同时底层改造成了基于 RxJava 的事件驱动模式，在服务调用成功/失败/超时的时候发布相应的事件，通过一系列的变换和聚合最终得到实时的指标统计数据流，可以被熔断器或 Dashboard 消费。

Sentinel 目前抽象出了 Metric 指标统计接口，底层可以有不同的实现，目前默认的实现是基于 `LeapArray` 的高性能滑动窗口，后续根据需要可能会引入 reactive stream 等实现。

## 3. Sentinel 的特色

除了之前提到的两者的共同特性之外，Sentinel 还提供以下的特色功能：

### 3.1 轻量级、高性能

Sentinel 作为一个功能完备的高可用流量管控组件，其核心 `sentinel-core` 没有任何多余依赖，，非常轻量级。开发者可以放心地引入 `sentinel-core` 而不需担心依赖问题。同时，Sentinel 提供了多种扩展点，用户可以很方便地根据需求去进行扩展，并且无缝地切合到 Sentinel 中。

引入 Sentinel 带来的性能损耗非常小。只有在业务单机量级超过 20W QPS 的时候才会有一些显著的影响（5% - 10% 左右），单机 QPS 不太大的时候损耗几乎可以忽略不计。

### 3.2 流量控制

Sentinel 可以针对不同的调用关系，以不同的运行指标（如 QPS、并发调用数、系统负载等）为基准，对资源调用进行流量控制，将随机的请求调整成合适的形状。

Sentinel 支持多样化的流量整形策略，在 QPS 过高的时候可以自动将流量调整成合适的形状。常用的有：

- 直接拒绝模式：即超出的请求直接拒绝。

- 慢启动预热模式：当流量激增的时候，控制流量通过的速率，让通过的流量缓慢增加，在一定时间内逐渐增加到阈值上限，给冷系统一个预热的时间，避免冷系统被压垮。

  ![2021-03-10-BJPjVp](https://image.ldbmcs.com/2021-03-10-BJPjVp.jpg)

- 匀速器模式：利用 Leaky Bucket 算法实现的匀速模式，严格控制了请求通过的时间间隔，同时堆积的请求将会排队，超过超时时长的请求直接被拒绝。

  ![2021-03-10-T19y6Y](https://image.ldbmcs.com/2021-03-10-T19y6Y.jpg)

Sentinel 还支持 [基于调用关系的限流](https://github.com/alibaba/Sentinel/wiki/流量控制#基于调用关系的流量控制)，包括基于调用方限流、基于调用链入口限流、关联流量限流等，依托于 Sentinel 强大的调用链路统计信息，可以提供精准的不同维度的限流。

同时 Sentinel 也支持更细维度的 [热点参数限流](https://github.com/alibaba/Sentinel/wiki/热点参数限流)，能够实时的统计热点参数并针对热点参数的资源调用进行流量控制。

### 3.3 系统负载保护

Sentinel 对系统的维度提供自适应保护策略，负载保护算法借鉴了 TCP BBR 的思想。当系统负载较高的时候，如果仍持续让请求进入，可能会导致系统崩溃，无法响应。在集群环境下，网络负载均衡会把本应这台机器承载的流量转发到其它的机器上去。如果这个时候其它的机器也处在一个边缘状态的时候，这个增加的流量就会导致这台机器也崩溃，最后导致整个集群不可用。针对这个情况，Sentinel 提供了对应的保护机制，让系统的入口流量和系统的负载达到一个平衡，保证系统在能力范围之内处理最多的请求。

![2021-03-10-GsuNJw](https://image.ldbmcs.com/2021-03-10-GsuNJw.jpg)

### 3.4 实时监控与控制面板

Sentinel 提供 HTTP API 用于获取[实时的监控信息](https://github.com/alibaba/Sentinel/wiki/实时监控)，如调用链路统计信息、簇点信息、规则信息等。如果用户正在使用 Spring Boot/Spring Cloud 并使用了 [Spring Cloud Alibaba](https://github.com/spring-cloud-incubator/spring-cloud-alibaba)，还可以方便地通过其暴露的 Actuator Endpoint 来获取运行时的一些信息，如动态规则等。未来 Sentinel 还会支持标准化的指标监控 API，可以方便地整合各种监控系统和可视化系统，如 Prometheus、Grafana 等。

[Sentinel 控制台](https://github.com/alibaba/Sentinel/wiki/控制台)（Dashboard）提供了机器发现、配置规则、查看实时监控、查看调用链路信息等功能，使得用户可以非常方便地去查看监控和进行配置。

![2021-03-10-4H7pOy](https://image.ldbmcs.com/2021-03-10-4H7pOy.jpg)

### 3.5 生态

Sentinel 目前已经针对 Servlet、Dubbo、Spring Boot/Spring Cloud、gRPC、Reactor、Quarkus 等进行了适配，用户只需引入相应依赖并进行简单配置即可非常方便地享受 Sentinel 的高可用流量防护能力。未来 Sentinel 还会对更多常用框架进行适配，并且会为 Service Mesh 提供集群流量防护的能力。

## 4. 总结

最后用表格来进行对比总结：

| Sentinel       | Hystrix                                        |                               |
| -------------- | ---------------------------------------------- | ----------------------------- |
| 隔离策略       | 信号量隔离                                     | 线程池隔离/信号量隔离         |
| 熔断降级策略   | 基于慢调用比例或异常比例                       | 基于失败比率                  |
| 实时指标实现   | 滑动窗口                                       | 滑动窗口（基于 RxJava）       |
| 规则配置       | 支持多种数据源                                 | 支持多种数据源                |
| 扩展性         | 多个扩展点                                     | 插件的形式                    |
| 基于注解的支持 | 支持                                           | 支持                          |
| 限流           | 基于 QPS，支持基于调用关系的限流               | 有限的支持                    |
| 流量整形       | 支持慢启动、匀速排队模式                       | 不支持                        |
| 系统自适应保护 | 支持                                           | 不支持                        |
| 控制台         | 开箱即用，可配置规则、查看秒级监控、机器发现等 | 不完善                        |
| 常见框架的适配 | Servlet、Spring Cloud、Dubbo、gRPC 等          | Servlet、Spring Cloud Netflix |

若有 Sentinel 设计上的疑问或讨论，欢迎来提 [issue](https://github.com/alibaba/Sentinel/issues)。若对 Sentinel 的开发感兴趣，不要犹豫，欢迎加入我们，我们随时欢迎贡献！