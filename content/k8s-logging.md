---
title: K8s Logging
---

在日志领域，从 VM 转到 K8s 需考虑的问题会复杂很多。

## 共享

**在 K8s 上，无论日志存储、还是采集端，都是多应用共享的，因此一是调优很难面面俱到、二是出问题的爆炸半径大很多。**

在 VM 模式下，通常一个 VM 只会部署一个应用实例，也就是说该 VM 上的采集端是应用专属的。但是到 K8s，每个节点机是随机混部多个用户的应用，容器应用的日志输出（stdout/stderr）实际落盘为节点机本地存储的日志文件；而 K8s 采集程序通常采用 DaemonSet 模式部署，即每个节点机一个采集端、采集该节点机上所有用户的日志文件。**由此带来的复杂状况本质上不是因为 VM 或 K8s、而是专属采集和混部应用的共享采集之间的差别。**

当然不是说 K8s 就不能实现专属采集，比如新版 OpenShift Logging Operator 已可定制用户专属的采集端、或使用 Sidecar 模式，但一者不是主流，二者只要落盘仍绕不开共享节点机存储。所以之下的讨论还是聚焦在现阶段主流产品所能提供的方案上，否则即使理论上做得到，以我们的资源也没有深度定制改造的可能。

### 日志滚动

日志文件的滚动（Rotation）配置代表了由于任何原因（采集端出问题 / 采集端上游出问题）无法及时采集的情况下，能够在源端所保存的日志数据大小。如果在采集恢复前落盘的日志文件已滚动删除，那这些数据就丢失了；这个风险既跟故障恢复时间有关、也和应用日志的吞吐率强相关，但无论哪种因素这个配置都是越大越保险。

而由于 K8s 应用共享节点机日志存储，且每个节点机的本地数据盘通常也不会太大，因此对每个容器实例的日志体积限制会比 VM 紧很多，以 K8s [Log rotation](https://kubernetes.io/docs/concepts/cluster-administration/logging/#log-rotation) 的默认[配置](https://kubernetes.io/docs/reference/config-api/kubelet-config.v1beta1/)为例：

- `containerLogMaxSize` is a quantity defining the maximum size of the container log file before it is rotated. Default: "10Mi"
- `containerLogMaxFiles` specifies the maximum number of container log files that can be present for a container. Default: 5

也就是说只有 50M，再结合 K8s 对节点机 Pod 数量的限定，就保证了节点机存储不会由于这个因素占满。当然该配置可以调整，但目前还不能分应用区别配置，如果只有某应用因生成日志太快需增加 100M，那么即使其他应用没这种需求，节点机存储需考虑的扩展也得是 100M 乘以 Pod 数；而反过来，调整某应用的一个或一组 VM 却不用考虑这么多。注意这里的重点不是有没有更多存储给 K8s，而是**日志采集需求（更保险）和资源需求（更节省）、性能需求的冲突在共享环境下会放大很多**，也就是上面说的很难面面俱到，以下就是一个红帽自己最真实的案例：

- [LOG-4536: add request and buffer settings to address memory consumption](https://github.com/openshift/cluster-logging-operator/pull/2220) 在解决了某个问题后又触发了另外的问题，不得不 [LOG-5123: **revert** LOG-4536 due to log loss](https://github.com/openshift/cluster-logging-operator/pull/2366)，不管哪种做法都没有也不可能彻底消除问题；当然这不是说就没有进一步调优的可能，但总归是**复杂很多**。

### 重试

在采集程序本身没问题、但推送数据给上游服务时出错的情况下，需要考虑的就是重试策略。以 Grafana 的采集工具 [Promtail](https://grafana.com/docs/loki/latest/send-data/promtail/troubleshooting/#loki-is-unavailable) 为例：

> In case of **any error** while sending a log entries batch, Promtail adopts a "**retry then discard**" strategy:
>
> - Promtail retries to send log entry to the ingester up to `max_retries` times
> - If all retries fail, Promtail discards the batch of log entries (which will be **lost**) and proceeds with the next one

它的 [backoff](https://grafana.com/docs/loki/latest/send-data/promtail/configuration/#clients) 默认配置是尝试 10 次后即丢弃数据，另外 [OTel](https://github.com/open-telemetry/opentelemetry-collector/blob/v0.95.0/exporter/exporterhelper/README.md#configuration)、[Fluent](https://docs.fluentbit.io/manual/administration/scheduling-and-retries#configuring-retries) 也采用了类似策略，但 [Vector](https://vector.dev/docs/reference/configuration/sinks/loki/#request.retry_attempts) 则是近乎无限次重试。当然这些配置都是能调整的，但凭这可以了解到各产品的**默认倾向**。

从这里也可以看出，**在日志等可观测性场景，数据的 "discard / lost" 并不是一个完全不可接受的选项**，当然这不是平台方放松运维的借口，只是希望大家理解这个前提；反之如果绝不丢失数据是前提，那么设计出来的日志产品就是另一个 Transactional Database 了。再比如之上，能不能提供一个既不重试也不丢弃的选项？仍然是基于这个前提，这些产品可能更倾向于工程实现上的简化，而不是考虑那出错数据又如何保存？

回到本节主题，从保守的角度当然是一直重试不丢数据为佳，但也只能保证在这个环节不丢数据，如果长时间的重试阻塞自然会在上游日志滚动的环节丢数据，不过总归还是把丢数据的时机延后了，目前 VM 就是采用这种策略。但是到了 K8s 共享环境，如果是上游服务完全不可用，当然只能一直重试，但如果服务可用、只是有部分请求或数据出错，那么一直重试就可能会阻塞另一部分能正常推送的数据（无论出自同一应用或其他应用），产生更严重的后果，这是我们真实遇到的问题（但不是发生在采集端端而是聚合端，但性质一样），在以下真实案例中链接说明。

虽然我们查到了根因消除了以上具体问题，但重点是如果因为其他因素引起了同样的状况，那我们是一直重试全面阻塞还是抛弃部分数据保证其他数据正常采集？前者引起滚动丢数据的概率不低、也影响本该正常使用的其他用户，而后者如果故障能马上恢复又白白丢了数据，总之共享环境下的权衡要**更左右为难**。

### 更多

- 在我们的暴力压测中，发现对采集程序的性能和稳定性影响很大：
  - [*Throttle*](why-here.md) 讨论了如何在采集端控制异常用户行为？显然这又是共享环境中的额外工作（VM 环境也有限流、但通常发生在传输后端），而且也是左右为难。
  - 这实际上也是共享环境下的**典型共性需求：如何既共享又相对隔离、尽量不被异常用户干扰**。类似 K8s Namespace 的 Resource Quota（但只限制了 CPU 和内存），以上的滚动配置其实也可以理解为一种 Log File Quota 只不过是全局的、不能按 Namespace 定制，而具体到这个问题我们需要的是一个 Log Throughout Quota 即使只是全局的。
  - **在共享环境也必须忍受被干扰的可能**，所有的好处都有代价，如果需求是彻底不被干扰，那就另建集群了。
  - 在产品提供相关控制功能前，也不是说所有问题都必须等着技术解决，现实是一个正常甚至出错的应用都不会产生这种极端情况、除非故意，而在内部可控的租户环境中辅以相关告警等，基本也可以及时消除。
- 每个 VM 的采集端应该没有 HA，而虽然 K8s 多节点部署了多个采集端，但每个采集端只负责自己节点机的日志采集，因此实质上也不是 HA。但重点是 VM 的采集端出问题了只会影响这个 VM 上的应用，而 K8s 采集端出问题则会影响该节点机上的所有应用，也就是说后者是更需要 HA 的。
- 如果将一个 VM 一个采集端看作一种横向扩展，在 K8s 同一节点机是不能部署多个采集端的（DaemonSet 模式），因此性能调优也会更操心。
- 灰度验证：即使在试验环境验证了程序或配置，对于日志采集场景，很可能错误是由异常数据或时间积累而触发，因此需要在真实场景做灰度验证。现在 K8s 的最小灰度粒度也是一个节点机，实现也比 VM 麻烦很多。

## 动态

### 元数据

**在 K8s 给日志添加的元数据是动态获取的，因此有可能出现缺失。**

在 K8s 给每一笔日志数据添加的元数据通常包含该应用的 Pod / Container 名称、Namespace、运行的 Node 和 Cluster、以及 Kubernetes [Well-Known Labels](https://kubernetes.io/docs/reference/labels-annotations-taints/) 等等，这样才能方便用户区分数据、查询分析。

由于采集端也是读取落盘的日志文件，从日志文件路径中就可以获取 Namespace、Pod 等基本信息，但是进一步的数据特别是 Labels 等只能通过 K8s API Server 动态获取；而这里还有另一个层面的动态问题，相比 VM 创建后基本就是静态的，K8s 上容器的创建和删除会频繁得多。考虑到时间差，比如一个容器启动、输出日志、然后删除，但采集程序还没来得及从 K8s 获取元数据的可能性是存在的，特别是执行完简单 Job 即退出的容器？我们在试验环境遇到过类似情况（参见以下真实案例）；当然可以说程序不够完美，但基于设计原理总归是只能降低概率而不是杜绝，相关产品如 [Vector](https://vector.dev/docs/reference/configuration/sources/kubernetes_logs/#delay_deletion_ms) 当然也会考虑这方面的应对。

而在 VM，这些元数据通常从采集端配置文件静态获取，基本上也就不会出问题。但这其实也跟我们的 VM 现状相关，如果 VM 是在真正的云上动态创建，那么很可能有些元数据也需要从别处动态获取。

#### 租户

其中最重要的一个元数据是租户信息，目前的设计是从 Namespace 按固定规则得到，主要是 Namespace 名称前缀加上 Hard coding 少量不规则的情况。实际上我们更希望的是能够从 Namespace 的 Label 信息中动态获取，这样如果用户想调整 Namespace 的租户，我们只需要修改相应的 Label 项，否则只能是我们添加更多的 Hard coding、或者用户新建 Namespace。

但是由于担心元数据丢失的可能，特别是租户信息，如果未获取到则该日志会被丢弃、后果很严重；而即使不丢弃，也只能保存到一个 `unknown` 租户，从用户的视角看也是丢失了。因此仍然采用了保守的静态方式，虽然实际上最常用的那些元数据貌似从未丢失过，遇到的真实丢失的情况也是间接的间接的元数据。总之基于 Label 我们能有更多灵活设计的可能，后续还是希望能采用。

## 真实案例

- 由于低版本 [*OpenShift Logging Operator*](why-here.md) 的 Bug，未及时释放已打开采集且被滚动删除的日志文件句柄，导致实际文件存储仍被占用、节点机存储快速升高，详见 [Vector log collector causes node high disk usage in RHOCP 4](https://access.redhat.com/solutions/7007644)。
  - 通过升级彻底解决。
- 因为部分异常数据的[*无限重试*](why-here.md)引起全面阻塞。
  - 这就是共享环境最**典型**的例子，单独看不算太严重甚至不算错误，但是影响了本应正常的其他应用、也就是说整个平台的稳定性。对于这类性质的问题，在共享环境理论上很难有一个两全其美、一劳永逸的万能法子，不过在实际上消除诱因后应该概率并不大？
- OTelCol 未及时获取 [*k8s.deployment.name*](why-here.md) 元数据。
  - 这不是一个最常用的元数据，但如果真用起来了、用户基于该字段做日常查询或 Dashboard，那么即使数据还在租户，从用户的视角看也是丢失了。
  - 其他产品如 [Promtail](https://grafana.com/docs/loki/latest/send-data/promtail/configuration/#kubernetes_sd_config) 甚至不提供该元数据？
  - 由于 OTelCol 的 Pipeline 有一个很容易误用且产生严重后果的情况（即使是用户自己用错了），再考虑其他因素，已放弃 OTelCol 转向最新版 OpenShift 默认的 Vector。

上面的问题是云原生可观测性平台在试验或试运行磨合阶段发现的，也很快定位到了根因，有的问题已彻底解决，有的问题如以上共享、动态是设计决定的只能做取舍；而通过这些问题，才真正深入理解了日志采集的机制原理和在 K8s 上的复杂性。

## 参考

- [Promtail and Log Rotation](https://grafana.com/docs/loki/latest/send-data/promtail/logrotation/)
- [A tailed file is truncated while Promtail is not running](https://grafana.com/docs/loki/latest/send-data/promtail/troubleshooting/#a-tailed-file-is-truncated-while-promtail-is-not-running)
