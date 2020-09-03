# Zuul & Spring Cloud Gateway选型比对

SpringCloudGateway是Spring官方基于Spring 5.0，Spring Boot 2.0和Project Reactor等技术开发的网关，Spring云网关旨在提供一种简单而有效的路由API的方法。

## 1. 简介

### 1.1 关于 Zuul

Zuul是Netflix开源的微服务网关，他可以和Eureka，Ribbon，Hystrix等组件配合使用。Zuul组件的核心是一系列的过滤器，这些过滤器可以完成以下功能：

* 身份认证和安全: 识别每一个资源的验证要求，并拒绝那些不符的请求
* 审查与监控：在边缘位置追踪有意义的数据和统计结果，从而带来精确的生产视图。
* 动态路由：动态将请求路由到不同后端集群
* 压力测试：逐渐增加指向集群的流量，以了解性能
* 负载分配：为每一种负载类型分配对应容量，并弃用超出限定值的请求
* 静态响应处理：边缘位置进行响应，避免转发到内部集群
* 多区域弹性：跨域AWS Region进行请求路由，旨在实现ELB\(ElasticLoad Balancing\)使用多样化

Spring Cloud对Zuul1.0进行了整合和增强。目前，Zuul使用的默认是Apache的HTTP Client，也可以使用Rest Client，可以设置ribbon.restclient.enabled=true.

Zuul已经发布了Zuul 2.x，基于Netty，也是非阻塞的，支持长连接，但Spring Cloud暂时还没有整合计划 。

### 1.2 关于 Spring Cloud Gateway （下面简称Gateway）

SpringCloudGateway是Spring官方基于Spring 5.0，Spring Boot 2.0和Project Reactor等技术开发的网关，Spring云网关旨在提供一种简单而有效的路由API的方法。

Spring Cloud Gateway作为Spring Cloud生态系中的网关，目标是替代Netflix ZUUL，其不仅提供统一的路由方式，并且基于Filter链的方式提供了网关基本的功能，例如：安全，监控/埋点，和限流等。

## 2. 选型

### 2.1 产品对比

|  | Gateway | Zuul 1.0 | Zuul 2.0 |
| :--- | :--- | :--- | :--- |
| 开源时间 | 2017年7月 | 2013年6月 | 2017年11月 |
| 架构 | 基于netty | 基于servlet | 基于netty |
| 运行方式 | 异步非阻塞 | 同步阻塞 | 异步非阻塞 |
| Spring Cloud集成 | √ | √ | × |
| 支持长连接、Web Socket | √ | × | √ |
| 限流、限速 | √ | 第三方 | 即将推出 |
| 监控 | √ | √ | √ |
| 上手难度 | 中等 | 简单 | 复杂 |

### 2.2 性能测试

| 组件 | 平均延迟 | RPS \(每秒请求数\) |
| :--- | :--- | :--- |
| gateway | 6.61ms | 3.24k |
| zuul1.0 | 12.56ms | 2.09k |
| zuul2.0 | ? | ? |
| none | 2.09ms | 11.77k |

这个没什么好比的，要比就和 Zuul 2.x 比，Zuul 2.x 在底层上有了很大的改变，使用了异步无阻塞式的 API，性能改善明显，不过现在 Spring Cloud 也没集成 Zuul 2.x，所以就没什么好比的。

### 2.3 开源组织

Spring Cloud Gateway 是 Spring Cloud 微服务平台的一个子项目，属于 Spring 开源社区，依赖名叫：spring-cloud-starter-gateway。

> [https://spring.io/projects/spring-cloud-gateway](https://spring.io/projects/spring-cloud-gateway)

Zuul 是 Netflix 公司的开源项目，Spring Cloud 在 Netflix 项目中也已经集成了 Zuul，依赖名叫：spring-cloud-starter-netflix-zuul。

> [https://github.com/Netflix/zuul](https://github.com/Netflix/zuul)

### 2.4 底层实现

> [https://stackoverflow.com/questions/47092048/how-is-spring-cloud-gateway-different-from-zuul](https://stackoverflow.com/questions/47092048/how-is-spring-cloud-gateway-different-from-zuul)

据 Spring Cloud Gateway 原作者的解释： Zuul构建于 Servlet 2.5，兼容 3.x，使用的是阻塞式的 API，不支持长连接，比如 websockets。另外 Spring Cloud Gateway构建于 Spring 5+，基于 Spring Boot 2.x 响应式的、非阻塞式的 API。同时，它支持 websockets，和 Spring 框架紧密集成，开发体验相对来说十分不错。

## 3. 结论

从性能测试结果可知，Spring Cloud Gateway的RPS是Zuul1.0的1.55倍，平均延迟是Zuul1.0的一半！

Spring Cloud Gateway、Zuul2.0都是刚推出不久，实现方式非常接近，最大的不同在于Spring Cloud暂时还没有对Zuul2.0的整合计划 。

而鉴于目前对 Java8，Spring Cloud等技术栈积淀还不够，top cloud网关确定暂选用Zuul1.0

从目前来看，gateway替代zuul是趋势，后续将持续关注spring cloud社区……

## 4. 参考

1. [Spring Cloud Gateway VS Zuul 比较，怎么选择？](https://juejin.im/post/5dd747a2f265da7dde7687a7)
2. [Zuul & Spring Cloud Gateway选型比对](https://www.houjq.com/2018/08/zuul-gateway/)

