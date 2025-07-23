---
title: Kubernetes Probe TLDR
---

## Probe

- Kubernetes 提供了 [Liveness, Readiness, and Startup Probes](https://kubernetes.io/docs/concepts/configuration/liveness-readiness-startup-probes/)，即[存活、就绪和启动探针](https://kubernetes.io/zh-cn/docs/concepts/configuration/liveness-readiness-startup-probes/)，这三类探针都用于容器应用的健康检查，但适用在不同环节或者有不同的反应。
  - 存活探针持续检查**容器应用自身是否出问题**，如死锁、OOM，这种情况通常可以重启应用来解决，因此 Kubernetes 判定存活探针失败时会重启容器。
  - 就绪探针持续检查**容器应用是否能正常对外提供服务**，在应用自身正常但所依赖的数据库之类不可用时，显然也无法提供服务，但重启应用还是解决不了问题，因此当就绪探针失败时，Kubernetes 只是不再将[服务](k8s-service-tldr.md)请求导向该容器、但不会重启它。
  - 启动探针是在 Kubernetes [v1.16](https://github.com/kubernetes/kubernetes/blob/master/CHANGELOG/CHANGELOG-1.16.md) 引入的新机制，先说明一些其他的概念，再解释已有存活、就绪探针之后为何还要引入一种新探针。
- 健康检查有多种方式，在 [Configure Liveness, Readiness and Startup Probes](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/) 就有 HTTP、TCP、gRPC 三种端口检测方式，当然也有 `exec` 这种执行任意可用命令的方式。
  - 以 HTTP 方式为例，配置好容器应用的端口及路径，Kubernetes 就可以类似 `curl` 一样检查返回结果是否 200。
  - 探针由和容器同一节点机的 [kubelet](https://kubernetes.io/docs/reference/command-line-tools-reference/kubelet/) 发起执行："To [perform](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/#define-a-liveness-http-request) a probe, the kubelet sends an HTTP GET request to the server that ......"
- 虽然有多种健康检查方式，但无论哪种都有一些共性配置，参见 [Configure Probes](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/#configure-probes)，在此列出主要的几个：
  - `periodSeconds`：存活、就绪探针是在容器整个生命周期内持续进行，但显然没必要时时刻刻，通过该参数设置检查间隔，默认为 `10` 秒。
  - `initialDelaySeconds`：大多数应用不会数秒内就完全启动，因此可以通过该参数设置在容器启动多久后才开始做健康检查，默认值为 `0`；但这不代表马上就进行，如果以上 `periodSeconds` 大于它，那实际是在 `periodSeconds` 秒后进行。
  - `failureThreshold`：在实际运转中，由于各种偶发因素，端口暂时失去响应很正常，因此一般不会一次失败就判定出问题了，通过该参数设置**连续**几次失败后才算真正有问题，默认值为 `3`。
- 以上配置项组合起来，影响的是 Kubernetes 对探针的**响应结果和效率**。
  - 如果一个应用启动耗时很长，比如要 60 秒，按默认配置 Kubernetes 会在 `max(0, 10) + 10*(3-1)` 也就是 30 秒之后重启该容器，那么应用将永远无法正常启动。
  - 如果我们将 `initialDelaySeconds` 调大到 60 秒，也就是说 Kubernetes 在 80 秒后才会考虑重启，这样应用就可以正常启动了；但这个值对别的应用可能是副作用，一个 5 秒就能启动的应用必须拖到 60 秒后才能对外提供服务。
  - 或者我们不动 `initialDelaySeconds` 而是将 `failureThreshold` 调大到 `8`，这样也是 80 秒，而且还不影响启动快的应用；但它的问题是在应用启动之后，如果遇到 OOM 等需要重启，连续检测三四次就基本可以确认问题了，这个值调得太大，那重启或停止转发服务需要等待的时间就太长了，而上一种方式没有这个问题。
  - 另外 `periodSeconds` 的设置也可能带来负面影响，如果应用是 1 秒或 21 秒启动，按默认值仍需多等 9 秒；如果调低该值，为了保持总的等待时间不变，那么又得调高 `failureThreshold`，这又会造成另一方面的负面影响，如增加 kubelet 的压力。
  - 从以上各种情况就可以看出，这些配置项**不存在**一个普遍性的最佳值或组合，只能具体应用具体对待；但即使具体到某一个应用，可以探索出一个最优解，但随着功能添加启动更慢，或者优化后更快，仍需同步到探针配置上。
- 启动探针解决的就是之上的两难问题。同一个探针，既要考虑启动阶段应该更保守多等一会应用启动，又要考虑运行阶段出了问题能快速响应，这种矛盾的需求就是问题所在；而现在的解决方式很直接，启动探针**只考虑**启动阶段的问题，存活、就绪探针**只针对**运行阶段进行优化。
  - 配置了启动探针后，其他探针就不会在启动阶段执行，直到应用启动成功，而之后该探针不再执行。
  - 但启动探针主要是针对启动过慢的应用，Kubernetes 的前两个探针和默认配置基本也能适配大部分的专业应用，所以真遇到这类问题，也请**反思**一下自己应用的性能问题。
- 在启动后，就只会有存活和就绪两种探针，而这两者的关系也不是完全独立的，一个"就绪"的应用应该是"存活"的、一个"不存活"的应用当然"未就绪"，但反之不会成立。
  - 但这也只是应该这样，在技术上是可以做到"就绪"且"不存活"的；比如一个快速启动的应用如 Nginx，我们调整存活探针的 `initialDelaySeconds` 到 `30`、端口故意设成错误端口，那么应用很快会就绪（Pod 状态为 `ready`），但因"不存活"又会一直重启，每次重启后又很快就绪……由此可见即使是健康检查这么小的领域，也没想象中简单。
  - 虽然两种探针的目的不同，但实现上却可以采用相同的方式，一个简化做法是都配置同样的 HTTP 访问，然后连续失败 3 次则"未就绪"、5 次则"不存活"，当然这种做法应对不了复杂状况。
- 可以认为**探针是 Kubernetes 对健康检查做的标准化，但它也只是标准化了"接口"**；应用需要告知 Kubernetes 使用哪个端口和路径做探针检测，但更重要或者说更高标准是**应用必须返回符合语义、符合接口要求的探针结果**。
  - 如开头所述，在应用正常、依赖数据库不可用的情况下，返回的应该是"存活"及"未就绪"，而不是反过来。
  - 要实现及时且真实的应用状态反馈比想象中困难，比如这个 [Caution](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/)："Liveness probes **must be configured carefully to ensure that they truly indicate** unrecoverable application failure, for example a deadlock."，以及配置错的后果："Incorrect implementation of liveness probes can lead to cascading failures."
  - 之所以我们现在很少遇到问题，大概率不是用得很精准、而是没有遇到真正的"high load"。
  - 当然开发人员也不是从头开始，现代的开发框架会帮忙，以下 Spring Boot 一节继续讨论。

## Manifest

探针配置是 [Pod](k8s-pod-tldr.md#manifest) 的一部分，我们只列出 Probe 相关的，其中各要素已在上一节讨论：

```
apiVersion: v1
kind: Pod
......
spec:
  containers:
  - name: app
    ......
    livenessProbe:
      httpGet:
        path: /healthz
        port: 8080
      initialDelaySeconds: 0
      periodSeconds: 10
      failureThreshold: 3
    readinessProbe:
      httpGet:
        path: /healthz
        port: 8080
```

## Spring Boot

- 如上所述，虽然 Kubernetes 定义了探针的标准或接口、以及针对不同结果的反应，但实实在在的返回结果仍然**依赖于应用自身**；而现代的开发框架显然不会忽视这个领域，如 Spring Boot 的 Actuator 提供了所谓的 [Production-ready Features](https://docs.spring.io/spring-boot/reference/actuator/enabling.html)，其中就包括健康检查。
  - 但是注意框架毕竟只是框架，开箱即用也只代表基本可用。
- 参见 Spring Boot 的 [Kubernetes Probes](https://docs.spring.io/spring-boot/reference/actuator/endpoints.html#actuator.endpoints.kubernetes-probes) 文档，在引入 Actuator 依赖后，存活、就绪探针的 `path` 分别配置 `/actuator/health/liveness`、`/actuator/health/readiness`，就基本可用了。
  - 注意以上文档和我们的试验都是基于 Spring Boot 3.5.4，不同 Spring Boot 版本或用户修改了应用配置，都可能引起探针配置变化。 
  - 之前存活、就绪探针多使用同一个 `/actuator/health`，但建议升级 Spring Boot，使用新的更具体的 `/actuator/health/readiness` 等。
- 但我们**更关注** `/actuator/health/liveness`、`/actuator/health/readiness` 的返回结果到底反应了 Spring Boot 应用的哪些真实状态？
  - Spring 在 [Application Availability](https://docs.spring.io/spring-boot/reference/features/spring-application.html#features.spring-application.application-availability) 文档有 Liveness State、Readiness State 的具体说明，但对大多数人还是过于抽象：
    - "An application is considered **live** as soon as the context has been refreshed"
    - "An application is considered **ready** as soon as application and command-line runners have been called"
  - 可以简化理解为应用基本启动 Vs. 完成所有的准备工作（如果有的话），我们在以下 [Lab](#lab) 一节通过真实试验来体会。
- 将管理端口和实际的业务服务端口分开是一个最佳实践，但有可能出现管理端口有响应、而业务端口出现问题，因此仍建议通过业务端口暴露健康状态，详见 [Kubernetes Probes](https://docs.spring.io/spring-boot/reference/actuator/endpoints.html#actuator.endpoints.kubernetes-probes) 讨论。
- 如 [Java in Container](container-tldr.md#java-in-container) 所述，相比其他技术栈，Java 应用在容器启动是最慢的？因此有调整 Kubernetes 默认配置的必要，以下给出一个示例，但**注意**这不是（也没有）一个普遍性的最佳值：
  - 参见 [Kubernetes Probes](https://docs.spring.io/spring-boot/reference/actuator/endpoints.html#actuator.endpoints.kubernetes-probes)，"Generally speaking, the `startupProbe` is not necessarily needed here"，如果以下调整仍不满足实际应用，再考虑引入启动探针。
  - `initialDelaySeconds`：三种探针都从默认值 `0` 增加到 `30`，相比 Demo，我们的真实 Java 应用耗时 30 秒以上启动很常见。
  - `periodSeconds`：都保持默认值 `10` 不变，特别轻量级的应用可以调整到 `5`，但也不用再低。
  - `failureThreshold`：
    - `livenessProbe`：重启阈值从 `3` 调高到 `5` 或者 `10`，但再高也没多大意义。
    - `readinessProbe`：保持默认值 `3` 不变。
    - `startupProbe`：调高到 `30`，如果引入启动探针说明应用启动很慢，那么就多设大一些。

### Lab

我们可以通过以下试验来真正确认 Spring Boot 的存活、就绪状态实际代表什么：

1. 生成 Spring Boot 脚手架，其中包含 [web,actuator,data-jdbc,mysql](https://start.spring.io/#!type=maven-project&language=java&platformVersion=3.5.4&packaging=jar&jvmVersion=17&groupId=com.example&artifactId=demo&name=demo&description=Demo%20project%20for%20Spring%20Boot&packageName=com.example.demo&dependencies=web,actuator,data-jdbc,mysql) 依赖。
1. 添加 `application.properties` 配置：
   ```
   spring.datasource.url=jdbc:mysql://nohost:3306/mydatabase
   spring.data.jdbc.repositories.enabled=false
   management.endpoint.health.probes.enabled=true
   ```
   - 故意设置错误的 MySQL 地址（`nohost`），以验证当运行时依赖出问题时应用的健康状态。
   - 如果不如上禁用 `jdbc.repositories.enabled`，那么在启动阶段就会因连接不上数据库而终止，也就无法继续之后的试验。
   - 在本地开发环境只有 `/actuator/health` 这个路径，`/actuator/health/liveness` 等需通过以上的 `health.probes.enabled` 启用，但在 [Kubernetes](https://docs.spring.io/spring-boot/how-to/deployment/cloud.html#howto.deployment.cloud.kubernetes) 环境勿需配置、会自动启用。
1. ```
   mvnw spring-boot:run
   ```
1. ```
   # curl localhost:8080/actuator/health
   {"status":"DOWN","groups":["liveness","readiness"]}
   # curl localhost:8080/actuator/health/liveness
   {"status":"UP"}
   # curl localhost:8080/actuator/health/readiness
   {"status":"UP"}
   ```
   当应用正常启动后，存活、就绪状态均正常，哪怕连接不上数据库，但全局健康状态是 `DOWN`（`/actuator/health` 是 global health endpoint）。
1. 再增加 `application.properties` 配置：
   ```
   management.endpoint.health.group.readiness.include=readinessState,db
   ```
   - 就绪或存活是一个 [Health Groups](https://docs.spring.io/spring-boot/reference/actuator/endpoints.html#actuator.endpoints.health.groups)，因此可以指定 Spring Boot 应用通过了哪些检查项才能认定为就绪 / 存活。
   - `readinessState` 为默认的就绪状态，基本等同于应用已启动，参见 [Spring Boot application lifecycle and related Application Events](https://docs.spring.io/spring-boot/reference/features/spring-application.html#features.spring-application.application-events-and-listeners)。
   - 显然我们添加的是数据库相关的检查项，能连接上数据库才认可就绪。
1. ```
   mvnw spring-boot:run
   ```
1. ```
   # curl localhost:8080/actuator/health
   {"status":"DOWN","groups":["liveness","readiness"]}
   # curl localhost:8080/actuator/health/liveness
   {"status":"UP"}
   # curl localhost:8080/actuator/health/readiness
   {"status":"DOWN"}
   ```
   因为连接不上数据库，就绪状态为 `DOWN`，使用 `curl -i ...` 则明确提示为 503。

从以上试验可以看出，Spring Boot 提供了健康检查的框架，但在连接不上数据库的情况下，它**并不能**帮应用做出是否还能提供服务的决策，因为在服务可用和不可用之间还有"服务降级、但仍能提供服务"这个选项，而是否有这个选项完全取决于应用本身的设计，正如 [Checking External State With Kubernetes Probes](https://docs.spring.io/spring-boot/reference/actuator/endpoints.html#actuator.endpoints.kubernetes-probes.external-state) 所述：

> As for the "readiness" probe, the choice of checking external systems **must be made carefully by the application developers**. For this reason, Spring Boot does not include any additional health checks in the readiness probe.

当然外部依赖也不止数据库，还包括缓存等，而且也不止检查外部依赖，还有 `diskspace` 等，参见 [Common Application Properties](https://docs.spring.io/spring-boot/appendix/application-properties/index.html) 中的所有 `management.health.***.enabled`。
