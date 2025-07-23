---
title: Observability Talk
---

[*可观测性漫谈*](why-here.md)的文字版本，建议通过该 PPT 更直观的了解。

## 什么是可观测性

### Observability Vs. Monitoring

- "Observability and monitoring are sometimes used interchangeably. As tooling, commercial offerings and practices evolved in complexity, monitoring was [**re-branded**](https://en.wikipedia.org/wiki/Observability_(software)) as observability in order to differentiate **new tools** from the old."
- 注意以上是广义的 Monitoring，而在 CNCF 的早期文档中有将"logging, monitoring, tracing"并列，包括我们内部也有用监控指代"metric"而非泛泛的监控，请通过上下文区分。

### 可观测性是第二件事

- 一个目标，第一是做成，那么第二就是提供一个**能保证持续做好的机制**。
- 开发一个应用：第一件事是写代码，第二件事就是自动化测试。
- 运维一个系统：第一件事是部署上线，第二件事就是可观测性！
- 自动化测试代表着能更迅速、更方便的**观测**到代码变更是否符合预期、不留隐患。

### PaaS 平台的第二件事

- OpenShift 本身就内置了 Prometheus 及 Alertmanager 等可观测性产品。
- 自主搭建云原生可观测性平台：Loki、Thanos / Mimir、Grafana、Promtail / OTelCol / Vector、Grafonnet / Mixin、Terraform、OpenResty、**可观测性数据治理**。

## OpenTelemetry

### 什么是 OpenTelemetry

- "[OpenTelemetry](https://opentelemetry.io/docs/), also known as **OTel**, is a vendor-neutral open source **Observability framework** for instrumenting, generating, collecting, and exporting telemetry data such as traces, metrics, and logs."
  - Framework / 框架：代表着规范化、标准化。
- "As an **industry-standard**, OpenTelemetry is supported by more than 40 observability vendors, integrated by many libraries, services, and apps, and adopted by numerous end users."
  - 在各自领域的地位类似 Kubernetes，但成熟度还相差甚远。
  - OTel 是云原生计算基金会 (CNCF) 下成长速度[第二](https://www.cncf.io/blog/2023/01/11/a-look-at-the-2022-velocity-of-cncf-linux-foundation-and-top-30-open-source-projects/)快的项目。
  - 目前是可观测性领域标准化的唯一选项，在随后讨论中**主要使用 OTel 的术语表述**。

### 可观测性数据

- [三大支柱](https://www.oreilly.com/library/view/distributed-systems-observability/9781492033431/ch04.html)：Log / 日志、Metric / 指标 / 度量、Trace / Distributed trace / 分布式追踪
- 第四个：[Profiling](https://opentelemetry.io/blog/2024/profiling/) / 剖析。
- [Event](https://opentelemetry.io/docs/concepts/signals/logs/) / 事件：有些产品会单列一类，而 OTel 认为"events are a specific type of log"。
- 可观测性数据统称为 Telemetry data，在 OTel 也叫 [Signal](https://opentelemetry.io/docs/concepts/glossary/#signal)。

### 元数据

- [元数据](https://opentelemetry.io/docs/concepts/glossary/#metadata)以键值对的形式添加到可观察性数据，在 OTel 叫 [**Attribute**](https://opentelemetry.io/docs/concepts/glossary/#attribute)。
  - 在日志产品中多叫做 Field、指标中叫 Label，也有叫 Tag 的；总之叫什么都很直观，不影响沟通。
- OTel 一个**重要设计**是将元数据明确划分为了两类：
  - Attribute：可观测性数据本身的元数据，以 HTTP 请求数这个指标为例，典型的包括 HTTP Verb、Path 等。
  - [**Resource**](https://opentelemetry.io/docs/concepts/glossary/#resource)：产生该可观测性数据的应用实例的属性数据，仍以上面的指标为例，主要包括应用的服务名、实例名等。
    - Prometheus 有 [Server-side metric metadata](https://prometheus.io/docs/introduction/roadmap/#server-side-metric-metadata-support) 的说法，其他产品或多或少也会区分，但少有明确定义，因此在实施中对数据治理的影响较大。
  - 但是 OTel 在命名上比较含混，泛泛的 Attribute、特定的 Attribute、Resource、Resource attribute 都在用，请通过上下文区分。
- 元数据是**语义规范**的核心部分，在[数据治理](#数据治理)中讨论。

### 采集端

可观测性平台的部署包含采集端和后端两个主要角色，但在大规模高可用部署时还会有聚合端、消息队列、缓存等。

[**Collector**](https://opentelemetry.io/docs/concepts/glossary/#collector) / 采集端：采集、加工及导出可观测性数据到后端。

- 也有叫做 Agent / Client 的，典型产品如 Fluentd、Logstash、Filebeat、Promtail、OTelCol、Vector。
- Collector 可能只负责某一类、或者采集所有种类的可观察性数据，后者方便在所有数据中重用公共逻辑。
- Collect 可以是 Push 或 Pull 模式。
- 不一定是独立的 Collector 实体，比如 Prometheus 既是后端同时也负责从应用 Collect 数据。
- 加工的一个重点工作是**数据治理**。
- 应用程序可以基于 OTel SDK 将数据直接输出至后端，但在真实场景需考虑重试、批量、敏感数据过滤等，还有应用自身能不能方便获取 Resource 元数据，因此最好还是交由采集端处理。

### 后端

[**Backend**](https://opentelemetry.io/docs/concepts/glossary/#observability-backend) / 后端：接收来自采集端的可观察性数据，负责 Ingest / 写入数据和后续的查询，以及历史数据清理等功能。

- 典型产品包括日志类的 Elasticsearch / Splunk / Loki、指标类的 Prometheus / Thanos / Mimir、分布式追踪 Jaeger / SkyWalking 等。
- 现在的后端产品或厂商也有逐步扩展支持多类可观察性数据的趋势。
- OTel 不关心可观测性后端的实现，但是要求支持 OTel 的采集端和后端通过 OTLP 协议规范完成数据传输，因此可以说 OTel 也规范了后端的部分接口标准。
  - 但由于当前（2024）的后端产品不支持 Resource 元数据？导致 OTel 的设计理念大打折扣。
- 后端的一种主流实现是只负责写入 / 存储的逻辑，而数据实际是存储在更后的对象存储或 ClickHouse 等 NoSQL 数据库。
  - 这里的重点是存储由第三方运维和支持，因此对于我们公司内部的选项只能是对象存储。
  - 这种做法并没有消除数据持久存储的复杂性，但可观测性平台本身的运维要简化许多。

### 生态

- "[**Vendors**](https://opentelemetry.io/ecosystem/vendors/) who natively support OpenTelemetry"
  - Apache SkyWalking / Grafana Labs / Datadog / Elastic / Splunk
- "[**Adopters**](https://opentelemetry.io/ecosystem/adopters/) who use OpenTelemetry"
  - 我们的定位就是 **end-user**，因此：
    - 平台方要做的是选择并用足用好支持 OTel 的 Vendors 产品。
    - 用户方要做的是直接或间接集成 [OTel SDK](https://opentelemetry.io/docs/languages/)。
    - 关于[协议规范](https://opentelemetry.io/docs/specs/)，我们主要关注 Semantic Conventions，而 OTel Specification、OTLP、OpAMP 等更复杂更底层的部分由 Vendors 操心。

## 日志

### 概述

- [日志](https://opentelemetry.io/docs/concepts/signals/logs/)是带时间戳的文本记录，可以是结构化或非结构化的。
  - 如果缺失了时间戳，OTel 也有 ObservedTimestamp 的设计。
  - 事件 / 稽核记录也是日志的一种，通常都是结构化的。
- 日志是最古老也最广泛的一类可观测性数据，因此也最难现代化（结构化 / 语义标准化），OTel 认为是"**biggest legacy**"。
  - OTel 对日志标准化的处理是区分 [Legacy and Modern Log Sources](https://opentelemetry.io/docs/specs/otel/logs/#legacy-and-modern-log-sources)，简单说就是做了妥协，不求也做不到完整严谨的标准化。
- 可观测性领域发展的一个**重点**是数据将进一步关联，比如日志成为分布式追踪里的一个事件。
- 在[十二因子](cn-12factor.md#日志)或云原生的理念中，日志被看作应用的事件流（"Treat logs as event streams"）。
  - 注意这里的重点不是日志的形式是文件还是流，而是应用不再操心日志的处理，比如输出至何处、Rotation 之类。
- 我们使用的日志采集端包括 Filebeat、Promtail、Vector，后端包括 Elasticsearch、Loki。
- OTel 用 [Log Record](https://opentelemetry.io/docs/concepts/glossary/#log-record) 表述某一条日志记录，但实际没这么严谨，通过上下文区分区分泛泛的日志、一批日志记录、某条日志记录即可。

### 结构化 1

- 结构化的日志自然是极大方便了自动化处理，比如采集端的加工和后端的索引查询。
- 当然非结构化的日志文本只要有规律，转为结构化数据也不复杂，但是：
  - 如果只是相对静态的 VM 环境，比如知道部署的是 Nginx，那么采集端做针对性的处理自然简单，这也是之前的常见做法。
  - 但如果是在 Kubernetes 这种共享采集端的场景，采集端要判断每一个容器日志的格式并针对性处理则要复杂许多，当然就更希望**从源头解决**。
- 虽然结构化日志对机器更友好，但通常非结构化日志对人类更友好。
  - 由于日志的一个主要用途是 Debug，因此方便 IT 人员阅读并不是一个可有可无的选项，也就是说**结构化日志并非全是好处**。
  - 无论什么格式的日志在集中采集后都可以做到更友好且多种形式的展现，但仍有很多场景用户是直接查看现场原始日志的。
- 仅仅结构化是不够的，必须配合**语义标准化**，这个我们在数据治理中讨论。

### 结构化 2

- 常见的结构化日志格式是 JSON，但可以多关注如下的新秀了：
  - "[**Logfmt**](https://brandur.org/logfmt) therefore achieves pretty **good readability for both human and computer**, even while not being optimal for either."
    ```
    at=info method=GET path=/ host=mutelight.org fwd="124.133.52.161" dyno=web.2 connect=4ms service=8ms status=200 bytes=1653
    ```    
- 注意格式本身也只代表形式上的结构化：
  - 对比以下的[*示例*](why-here.md)：
    - 形式结构化：
      ```
      {......,"mdc":{},"message":"Question 8 has no answer"}
      ```      
    - 语义结构化：
      ```
      {......,"mdc":{},"questionId":"8","whatHappened":"no-answer"}
      ```
  - 这其实也是 OTel 妥协的重点。
  - 但有没有必要追求完美结构化？

### 日志采集

在日志采集端，从 VM 转到 Kubernetes / OpenShift 需考虑的问题会复杂很多，[K8s Logging](k8s-logging.md) 有详细讨论：

- **在 K8s 上，无论日志存储、还是采集端，都是多应用共享的，因此一是调优很难面面俱到、二是出问题的爆炸半径大很多。**
  - 日志文件的滚动配置代表了由于任何原因（采集端出问题 / 采集端上游出问题）无法及时采集的情况下，能够在源端所保存的日志数据大小；在共享环境下，日志采集需求（滚动配置越大越保险）和资源需求、性能需求的冲突会放大很多。
  - 在我们的暴力压测中，发现对采集程序的性能和稳定性影响很大，这实际上也是共享环境下的典型共性需求：**如何既共享又相对隔离、尽量不被异常用户干扰**。
  - **在共享环境也必须忍受被干扰的可能**，所有的好处都有代价，如果需求是彻底不被干扰，那就另建集群了。
- 红帽自己的真实案例：[LOG-4536: add request and buffer settings to address memory consumption](https://github.com/openshift/cluster-logging-operator/pull/2220) 在解决了某个问题后又触发了另外的问题，不得不 [LOG-5123: **revert** LOG-4536 due to log loss](https://github.com/openshift/cluster-logging-operator/pull/2366)。
  - 不管哪种做法都没有也不可能彻底消除问题，当然这不是说就没有进一步调优的可能，但总归是**复杂很多**。
- 更多讨论参见以上链接。

### 无节制的日志输出

- 典型现象：
  - 日常开启 Debug / Trace 日志。
  - 将 Debug / Trace 性质的日志设为 Info 级别。
  - 简单一条断言就行的情况却输出 Exception stack trace。
  - 对反复出现但不影响运行的错误日志置之不理。
  - "==================="
  - 竟然想用海量日志取代分布式追踪。
  - 一股脑记录完整的输入输出数据而非概要信息。 
- 这些日志问题在业务应用开发中应该是属于相对最容易解决的点。
  - 那**一直不解决说明我们观测到了什么**？参见黑名单[*三期*](why-here.md)。

### 日志采集策略

从平台方的角度，在无法两全的前提下我们的选择是：

1. 保证合理输出的应用日志完整采集及可观测性平台稳定。
1. 不保证无节制输出的应用日志完整采集。
   - 不仅是不保证，还需要考虑**惩罚**机制。

而不是：

1. 保证所有应用日志完整采集。
1. 影响可观测性平台稳定并最终影响全体日志采集。

### 用户需知

- 虽不多见，但我们也数次遇到了稽核的需求。
  - 而遇到压力最巨大的需求是上报监管。
- 众所周知，日志或者说我们日常提及的日志，从设计原理上来讲是不保证绝不丢失不重复的。
  - 也就是说在日志等可观测性场景，数据的"discard / lost"不是一个完全不可接受的选项。
  - 注意这不是我们放松运维的借口，只是希望大家**理解这个前提**；反之如果绝不丢失数据是前提，那么设计出来的日志产品就是另一个 Transactional Database 了。
- 但稽核这个词也会让人困扰，因为业务、管理上的稽核和纯技术的"audit.log"在严肃性上还是有较大差别的，因此我们不纠结这个词、澄清如下：
  - 可观测性平台可以用于**能够容忍少量数据缺失或重复**的任何场景、任何用途。

### 日志产品

- 采集端：
  - Filebeat：ELK 技术栈用的较多。
  - Promtail：和 Loki 配合最佳？**更新**：将被 Alloy [取代](https://grafana.com/docs/loki/latest/send-data/promtail/)。
  - OTelCol：尚未成熟但最有希望？
  - Vector：新秀？
- 后端：
  - Elasticsearch
  - Loki：Grafana 技术栈，由于持久存储全转 OSS、运维相对最轻松。
- 从 OpenShift Logging 5.8 开始，"We [encourage](https://docs.redhat.com/en/documentation/openshift_container_platform/4.15/html/logging/release-notes#logging-release-notes-5-8-0-deprecation-notice) all users to adopt the Vector and Loki log stack"。

## 指标

### 概述

- [指标](https://opentelemetry.io/docs/concepts/signals/metrics/)是一个系统或应用在运行时的测量值，例如系统错误率、CPU 利用率、服务请求次数等。
- 指标要素包括名称、类型、元数据及时间戳、数值，以下为 Prometheus 指标的示例：
  ```
  # HELP http_requests_total The total number of HTTP requests.
  # TYPE http_requests_total counter
  http_requests_total{method="post",code="200"} 1027 1395066363000
  http_requests_total{method="post",code="400"}    3 1395066363000
  ```
- 相比文本类型的日志，**数值**类型的指标自然最方便比较，比如告警的触发条件、不同时间点的趋势判断等等。
- 在云原生环境可以认为 [Prometheus](https://prometheus.io/docs/introduction/overview/) 是准事实标准了，但欠缺 Resource 属性等仍有很大不足。
  - Prometheus 已承诺 "release a Prometheus 3.0 in 2024 with OTel support as one of its more important features"。
    - **更新**：Prometheus 已发布 [3.0](https://prometheus.io/blog/2024/11/14/prometheus-3-0/#otlp-support)，但目前（2025-06）对 OTel 的支持有限、且主要生态仍是版本 2。
  - 在指标领域还有一个 [OpenMetrics](https://opentelemetry.io/docs/specs/otel/compatibility/prometheus_and_openmetrics/) 标准，总的来讲偏底层、OTel 也会考虑兼容。
- 无论如何，指标的**标准化**工作要远比日志轻松。

### Prometheus

- 虽然 OTel 的设计更理想，但目前能落地的最主流指标类产品就是 Prometheus，因此本节随后的讨论主要也是基于 Prometheus。
  - 不只是 OpenShift Monitoring 默认集成了 Prometheus，更重要的是"Kubernetes components emit metrics in [**Prometheus format**](https://kubernetes.io/docs/concepts/cluster-administration/system-metrics/)"。
  - 但 Prometheus 并不局限于 Kubernetes 环境，现代云及传统的 VM 环境支持的也足够好。
  - 有更通用的[时序数据库产品](https://prometheus.io/docs/introduction/comparison/)，但 Prometheus 是集 TSDB、后端、采集端、告警于一身的完整监控系统。
  - Grafana 的日志类产品 Loki 等受其影响很大，很多设计思路均沿袭自 Prometheus，自然也包括不足。
- Prometheus 只专注核心功能：
  - Authn、Authz 交由反向代理解决。
  - 默认仅保存最近两周的数据，长期历史数据存储需要额外的后端产品如 Thanos、Mimir 等。
  - 主要通过独立双实例重复采集数据来保证高可用，因此需要去重，当然这个操作对用户是透明的。

### 指标生成

- 生成指标和生成日志在本质上并无二致，都需要开发人员显式创建日志或指标：
  - 当然创建指标也是基于 SDK 或库，不会从头开始。
    ```
        meterRegistry.counter("o11ydemo_question_asked", "question", tagValue).increment();
    ```
  - 但生成后应用不需要像日志那样操心指标的存储管理，而是交由采集端负责。
  - 在 Kubernetes 环境，平台组件会生成和应用相关的一些运行时通用指标如 CPU、内存等，其他平台应该也类似。
    - 因此从用户的视角，即使不改造自己的应用也能获取一定的指标，但如果需求更特定、更具体，那么改造不可避免。
- 针对主流中间件、硬件、存储、云平台等等，Prometheus 社区提供了大量开箱即用的 [Exporter](https://prometheus.io/docs/instrumenting/exporters/)，也就是说无需改造即可输出 Prometheus 指标。
  - 当然 Exporter 也不是从头做起，专业软件都会考虑到监控需求，因此 Exporter 主要是将现成的监控数据转为 Prometheus 格式，比如 HAProxy Exporter "[scrapes HAProxy stats](https://github.com/prometheus/haproxy_exporter)"、[MySQL Server Exporter](https://github.com/prometheus/mysqld_exporter) 主要利用了 [MySQL Performance Schema](https://dev.mysql.com/doc/refman/8.0/en/performance-schema.html)。
  - 但也有 Natively 支持 Prometheus 的趋势，比如 HAProxy 现已内置 Prometheus 模块、不再需要以上 Exporter。

### 指标采集

- Prometheus 的指标采集主要使用拉模式，也就是应用需提供 HTTP 服务输出指标、由 Prometheus 定期 GET，默认的采集间隔是 1 分钟。
  - 但 OTel 可能更倾向于推模式？"[OTLP Metrics Exporter](https://opentelemetry.io/docs/specs/otel/metrics/sdk_exporters/otlp/) is a Push Metric Exporter"。不过这不是我们现在需要操心的事，保持关注即可。
- 在现代云环境，去哪些 IP、哪个端口拉取数据显然是很难静态配置的，因此 Prometheus 集成了众多主流环境的 [Service Discovery](https://prometheus.io/docs/prometheus/latest/configuration/configuration/)。
- 采集端的一个重点工作是加工 / 标准化可观察性数据，这个动作在 Prometheus 叫 [Relabel](https://prometheus.io/docs/prometheus/latest/configuration/configuration/#relabel_config)。
  - 对应的在 OTel 叫 [Transforming telemetry](https://opentelemetry.io/docs/collector/transforming-telemetry/)，但 OTel 并不会规范这一部分的设计。
  - Relabel 的语法比较笨拙繁琐，我们现在的一个思路是从"采集端 → 后端"调整为"采集端 → 聚合端 → 后端"，在聚合端处理复杂的加工逻辑，并且可以在多类可观测性数据中重用这些逻辑。
    - 但加工不可能完全抛开采集端，比如获取 Resource 属性的工作就不适合聚合端，因为很可能不在同一环境。
    - 目前（2024）聚合端采用 Vector，其中的 [Vector Remap Language](https://vector.dev/docs/reference/vrl/) (VRL) "features a **simple syntax** and a rich set of built-in functions tailored specifically to observability use cases"，而且方便[*自动化测试*](why-here.md)。**不只开发，平台方运维特别是配置变更时一样需要自动化测试**！

### 指标查询 1

相比日志查询，指标查询的门槛要**高很多**：

- 在日志查询时，过滤一些属性（主要是 Resource，比如哪个 Namespace、哪个 Pod）后基本就是文本搜索了，再复杂也就是学点正则表达式，但指标查询远不止此。
- 技术背景或要求：
  - 想研究某个问题，至少需要先了解相关领域有哪些主要指标，简单说就是指标名称列表。
  - 也要分得清类似指标的具体含义，比如 `container_cpu_usage_seconds_total` Vs. `container_cpu_user_seconds_total`。
  - 相同指标下元数据不同取值的具体含义，比如 `container_memory_working_set_bytes` 的 `image` 属性为空 Vs. 不为空。
- 数学统计学背景或要求：
  - 指标是分[类型](https://prometheus.io/docs/concepts/metric_types/)的，其中复杂类型的统计学含义。
  - 还有数据类型 Instant vector、Range vector、Scalar。
  - 复杂的统计函数。
- 类似 SQL 的[查询语法](https://prometheus.io/docs/prometheus/latest/querying/basics/#expression-language-data-types)。
- 还要知道有些指标是有加工过的粗粒度指标，查询后者能快上很多。
- 显然这个复杂度并不是 Prometheus 引入的，而是这领域的本来。

### 指标查询 2

在自身技术或统计不靠谱前提下的最佳实践：

1. Prometheus Monitoring [Mixins](https://monitoring.mixins.dev/):
   - "A mixin is a set of Grafana dashboards and Prometheus rules and alerts, packaged together in a reuseable and extensible bundle."
   - OpenShift 现有的监控主要就是基于 [Kubernetes Mixin](https://github.com/kubernetes-monitoring/kubernetes-mixin)。
   - [Grafana](https://github.com/grafana/loki/tree/v3.0.0/production) 系产品在源码中分享的 Mixin 是真实用于 Grafana Cloud 的。
1. 研究之上的 Mixins，无论 Dashboard、Rule 还是 Alert，最终都会包含具体的查询语句，自然所取用的指标名称、查询条件、计算函数就都是现成的了。
1. 由于这些 Mixins 并不是 Demo、涵盖了各领域的主要真实使用场景，因此对于我们主要就是调整一些外部不可能涉及到的地方，比如属性命名的内部规范（参见之后的数据治理）。

总之就是尽量不要从头做起，而是依葫芦画瓢，关键是**现成的、真实的、复杂的、专业的、开放的参照物确实存在**。

### 指标产品

- 采集端及后端：
  - Prometheus：OpenShift 集成，采集和短期存储。
  - [Thanos](https://docs.redhat.com/en/documentation/openshift_container_platform/4.18/html/monitoring/about-openshift-container-platform-monitoring#about-ocp-monitoring) sidecar：OpenShift 集成，查询去重。
- 长期存储后端：
  - Thanos：早期采用，因不支持多租户转为以下方案。
  - Mimir：Grafana 技术栈，支持多租户；由于持久存储全转 OSS、运维相对最轻松。

## 分布式追踪

### 概述 1

- [分布式追踪](https://opentelemetry.io/docs/concepts/observability-primer/#distributed-traces)用于记录一个用户请求在分布式架构下的完整调用路径。
- 分布式追踪的每一步就是一个 Span，要素包括：
  - Span ID：自身唯一标识。
  - Trace ID：同一请求下的所有 Span 都是同一个 Trace ID。
  - Parent ID：触发本步骤的 Span ID，如果为空自然就是根节点；这是形成完整调用路径的关键。
  - 起止时间：这是分析性能问题究竟出在何处的**关键**，我们最常遇见的一个**误用**就是想用日志实现这个功能。
- 分布式追踪是最需要标准化的可观测性数据，现实中全部应用使用同一语言 / 框架  / 库的概率为零，如果不标准化根本就没办法"分布式"追踪。
  - OTel 早期就是由两个分布式追踪的标准化项目合并而成，最先做也最成熟的就是分布式追踪。
  - 当然首先需要标准化的是跨系统的上下文传递，[Context Propagation](https://opentelemetry.io/docs/concepts/context-propagation/) 是分布式追踪的**核心概念**，目前 OTel 默认采用的是 W3C TraceContext specification。

### 概述 2

- 但分布式追踪并不是只能处理分布式的场景比如网络调用，所采用的技术往往也能追踪到代码级别的[粒度](https://opentelemetry.io/docs/concepts/instrumentation-scope/)。
  - 因此分布式追踪的产品也经常被当作性能分析的工具。
  - 但注意在比较产品时并不是追踪的粒度越细就越好，这是有性能代价的。
- 当日志能够和某个 Span 关联时自然就成了分布式追踪的一部分：
  ```
  2022-12-26T17:20:11.416+08:00 DEBUG [,ed7d6ba1a09fcdfa3167493a499b4778,21c00d70bc3c40e1] 75824 --- [nio-8080-exec-1] ......
  ```
- 不说全部、至少得有一部分应用上了遵循标准的分布式追踪产品，才谈得上是真正的分布式追踪。
  - 才可能构建动态、实时更新的系统依赖图。
  - 当然也不应该区分云原生分布式追踪和非云原生分布式追踪。
  - 如果没有，当作性能分析工具也挺好哒。

## Profiling

### 概述

- [Profiling](https://grafana.com/docs/pyroscope/latest/introduction/profiling/#profiling-fundamentals) 用于分析一个程序的运行时行为，主要是定位程序中的哪一部分消耗了多少 CPU / 内存 / IO 等资源。
  - 取代诸如 [*Java OOM Dump*](why-here.md) 场景的手工分析 / Profiling。
- 2024 年 3 月 OpenTelemetry [announces](https://opentelemetry.io/blog/2024/profiling/) support for profiling，到现在（2025-06）仍是"[under development or at the proposal stage](https://opentelemetry.io/docs/concepts/signals/)"。
- Grafana 有提供可实际落地的后端产品 [Pyroscope](https://grafana.com/oss/pyroscope/)，当然也不可能遵循还未确定的 OTel 规范。
- 这里提一下 APM（Application performance management），本来以为 APM 属于 Profiling 领域，但以 APM 的代表产品 SkyWalking 为例，实际是分布式追踪 + 指标类产品，如前一节分布式追踪所述、被用成了性能分析工具，在此澄清一下，APM 就是个框、装什么都不需要纠结。

## 数据治理

### 语义规范 1

- [Semantic Conventions](https://opentelemetry.io/docs/concepts/glossary/#semantic-conventions)："Defines **standard names and values of Metadata** in order to provide vendor-agnostic telemetry data."
  - "all telemetry data to be [**fully correlated in a precise and robust manner**](https://opentelemetry.io/docs/specs/otel/logs/#limitations-of-non-opentelemetry-solutions)"
  - 永远不可能只有一家厂商的产品满足所有需求，这就是标准化的意义。
  - 2023 年 4 月 Elastic 将 Elastic Common Schema (ECS) [贡献](https://www.elastic.co/cn/blog/ecs-elastic-common-schema-otel-opentelemetry-announcement)给 OpenTelemetry。
- 当然需要标准化的远不止元数据：
  - 最基本的如前面提及的数据分类、部署角色（采集端 / 后端）等术语命名。
  - 偏底层的数据模型、接口协议等，但这一部分我们主要是使用相关产品、不着急深入了解。
- 作为最终用户，从标准化或者说可观察性领域数据治理的角度，我们最关注的就是语义规范，这有几个原因：
  - "There are only two hard things in Computer Science: cache invalidation and [**naming things**](https://martinfowler.com/bliki/TwoHardThings.html)."  -- Phil Karlton
  - 目前 OTel 在语义规范方面的进展非常有限、预期也不乐观。
  - 不可能完全依赖外部标准，必须结合内部实际情况调整。

### 语义规范 2

- 实际大部分命名空间下的元数据语义规范都是给 SDK / 可观测性产品用的，我们顺其自然就行，现在主要关注 [**Resource Semantic Conventions**](https://opentelemetry.io/docs/specs/semconv/resource/) 这个子集中的有限几项：
  - [service.name / service.namespace](https://opentelemetry.io/docs/specs/semconv/resource/#service) 可以理解为应用名称及分组，这个没有严格定义、主要取决于内部设计。
  - [deployment.environment](https://opentelemetry.io/docs/specs/semconv/resource/deployment-environment/) 部署环境这个很直观，但是并没有对取值标准化。
  - Kubernetes：
    - 这是完成度最高的部分，因为本就是事实标准，OTel 的语义规范工作非常轻松，添加 [k8s](https://opentelemetry.io/docs/specs/semconv/resource/k8s/) 命名空间即可。
    - 但尚未考虑 Kubernetes 的 Well-Known Labels 内容（主要是 [app](https://kubernetes.io/docs/reference/labels-annotations-taints/#app-kubernetes-io-component) 前缀的部分）。
    - 对比非 Kubernetes 环境不太理想的状况，大多评论"还不是因为 Kubernetes 够标准"，但这就是应该上 Kubernetes 的**强大理由**啊，通过 Kubernetes 的标准化发布、标准化运维并获取到标准的可观测性数据！
- 我们[补充](https://opentelemetry.io/docs/specs/semconv/general/naming/#recommendations-for-application-developers)的语义规范应放在公司域名的命名空间下。
- OTel 的语义规范主要还是技术向的，但从我们的角度也希望补充一些管理视角的，无论命名还是取值，比如区分技术平台与业务应用、核心与非核心等等。
- 总之语义规范需要足够的积累和持续进化，而**治理类工作是没有一次性的**。

### 数据模型

- 虽然 OTel 通过 OTLP 规范了后端产品的接口协议，但并未干预到后端的具体实现，而当前成熟的后端产品也不可能是从头就基于 OTel 设计的，因此所谓的 "natively support" 实际是打了很大折扣。
- 以日志产品 Loki 为例，"the OpenTelemetry protocol [**differs**](https://grafana.com/docs/loki/latest/send-data/otel/#format-considerations) from the Loki storage model"，而不得不"data in the OpenTelemetry format will be mapped by default to the Loki data model during ingestion"：
  - Loki 名称中不支持除下划线外的其他特殊符号，因此 `k8s.cluster.name` → `k8s_cluster_name`。
  - Loki 有 Index labels 和 Structured Metadata 的设计，但并不等于 Resource 和 Attributes。不同于上一条转换对用户基本没什么心智成本，欠缺 Resource 就很不舒服了。
  - Loki 只有一个 Timestamp，没有 ObservedTimestamp。
- 当然这不是 Loki 一个产品的问题，它本就是参考 Prometheus 的设计。但作为后端产品如此，那么在用户查询展现时显然也不可能还原成 OTel 的理想状态。
- 总之现在的效果就变成了**似乎用上了 OTel、似乎又没有**……

### 租户

- 租户代表可观测性数据的最基本隔离。
- 基于当前的开源实现（主要是 Grafana 系），目前只控制到租户级别，而同租户内未做更细粒度的数据访问控制。
- 因此一个主要的设计工作是选择租户粒度：
  1. 我们对标的是云管中的业务、取值为业务英文缩写。
     - 想法是对于可观测性场景，这种粒度控制下的数据访问既可以接受、也不至于细到太琐碎干扰运维使用，至少这几年来用户未觉不妥。
     - 但目前主要是技术类指标，如果考虑到也能生成业务指标，应该是需要控制更严的，但这就得重新计算一下投入。
  1. 在 PaaS 平台创建 Namespace 时名称首单词为相应业务英文缩写。
  1. 在数据采集时从 Namespace 名称提取首单词作为租户编码并写入后端，有少量标准化前的非规则情况需 Hard coding。
- **TODO**：至少先将技术类平台的租户统一，现在跨平台都匹配不上，**亟需**组织级标准化工作。
- 在 Grafana 中：
  - 数据源和后端租户是一一对应的（技术上其实支持一个数据源访问多个租户）。
  - 同一 Grafana Organization 可以创建多个访问不同租户的数据源。

### 多所有者问题 1

- 关于可观测性数据的归属：
  - 业务应用所生成的可观测性数据自然属于业务团队。
  - 平台所生成的可观测性数据：
    - 首先当然是属于平台团队；
    - 其中和应用团队有关的部分数据，比如某应用 Pod 的 CPU、内存等，即使不说完全归属于应用团队，是不是也应该能自主的查询统计？
- 这就是所谓的多所有者问题，很奇怪未搜到业界相关的讨论和方案，[Multiple Owners Problem](o11y-multi-owners-problem.md) 是自己发明的一个词（该链接有更详细的讨论）。
  - 这里不纠结应用团队到底算不算 Owner，本质是有多个相关方无需授权、**天然**就应该拥有某类数据的查询权限；
  - 而"天然"也代表着开箱即用、而不是通过繁琐的授权配置来勉强达到目的，也就是说即使商用版比如 GEM 提供了 Label-based access control、也没有很好或很简洁的解决问题。
- 如果接受以上前提，那问题就变成了：
  - 我为什么不能很方便的查询到 LB 的可观察性数据？我们是 PaaS 平台方，但同时也是 LB 平台的用户方。
  - 我为什么不能很方便的查询到……

### 多所有者问题 2

- OpenShift 没有明确提及多所有者这个问题场景，但至少解决了指标类型数据的访问（日志类的 Event 也算部分解决）：
  1. 普通用户登录 OpenShift Console 访问自己有权限的 Namespace。
  1. 从 Observe 菜单，**可以且只能**访问和这个 Namespace 相关的平台指标。
- 我们的尝试：
  - 指标：参考 OpenShift 的实现，在查询阶段控制，但粒度是租户而非 Namespace。
  - 日志：在日志采集阶段将多所有者数据同时写入平台租户和应用租户，目前只包括 Event、Audit 日志。
  - 分布式追踪：**未解决**。这是最复杂的场景，允不允许查询上下游应用和本次交易相关的可观察性数据？以上解决的只是双所有者问题，这才是最麻烦的多所有者。
  - 这里的平台数据不止是 PaaS、还包括可观察性平台，就是说用户也可以查询到 Loki 等指标。
  - 目前的技术方案都不太满意，但总归是一个起点。

## 可视化

### 概述

- 可视化是将数据信息以图表等更直观的形式呈现出来，方便用户**快速抓住重点**。
  - 最直观的例子就是汽车仪表盘，司机瞄一眼就知道车速、转速高低的大致范围，而不用分心去纠结具体数值。
  - 当然直观形式也不止视觉，倒车雷达是另一个例子。
- 所以在可视化领域一个最常见的概念就是**仪表盘** / Dashboard。
  - 仪表盘之于查询指标的 PromQL、查询日志的 LogQL，就等同于 BI 中的报表之于 SQL。
- 既然是视觉领域，那么讲究设计感就**不是**一个可有可无的要求。
  - 拙劣的仪表盘让人抓不住重点甚至引起误读。
  - 用户还是需要训练的，从一些复杂图形中找出重点、不误读没有想象中简单。
  - **视觉标准化**是提升整体设计水平的基础，总不能你的红色是危急我的红色是胜利。

### Grafana

- 我们主要聚焦可视化工具最主流的选项 [Grafana](https://grafana.com/oss/grafana/)。
  - Grafana 是从 Elastic 的 Kibana 项目 Fork 出来的，但从后者的日志转向指标可视化，不过后来大家都在做大做全。
  - Kubernetes 等众多生态圈的仪表盘主要都是基于 Grafana 制作。
  - 但 Grafana 不止是可视化工具，能够查询远超可观察性数据类型的众多数据来源是它的一个主要优势，另外还包括告警功能。
- 直接查询数据是有门槛甚至很高的，说明了这类工具的**重点用户是一线 IT 人员**。
  - 当然 Grafana 也提供了 GUI 工具 Builder 帮助用户构造 SQL / PromQL / LogQL 等等。
  - 但能不能直接查询数据不止是易用性的问题，还**代表**安全控制更复杂或者基于这个场景不需考虑。
- 决定可观测性水平的一个**重要表征**就是在 IT 内部讨论时，我们使用的是 Grafana / Kibana 还是 PPT / Excel。
  - 这代表着观测是否做到了**更全面、更自动化以及更真实**。

### 用户行为

- 面向一线 IT 人员的可视化工具和面向业务用户、管理者的 BI 工具在使用行为上其实是有**很大差别**的。
  - 上一节的直接查询数据。
  - 再举个很细节的例子，比如在 Grafana 查询某项目，用户通常是直接用项目代码比如 sakura，因为他的 Git 库也是 sakura、部署的 Kubernetes Namespace 也是 sakura，用户没有任何心智转换成本；但是对于其他人，最好选项列表里是"某某某应用"……
    - 但细节背后的事却不是细节，因为这不只是交互层面的事，还涉及到数据来源的模型设计，简单说就是 Tag PK 维度表（借用数据仓库的概念）。
- **用户是需要训练的**。比如快速查询的技巧（无论是通过仪表盘还是直接查询）：
  1. 在大多数情况下首先约束查询时间范围（Grafana 界面通常在右上角），对于时序数据这是最主要的索引。
  1. 过滤属性字段特别是 Resource，这也是主要索引。
  1. 之后才是其他查询条件。

### Mixin

[Mixins](https://monitoring.mixins.dev/) 是以一种可重用、易扩展的方式制作仪表盘等。

- 相比在 Grafana GUI 界面制作仪表盘，Mixin 是以编程方式进行，显然在入门和学习曲线上是劣势，但在企业环境考虑到持续发展，这是一个专业方向。
- Kubernetes 等众多生态圈已基本从原始手段**转为**使用 Mixin 制作仪表盘。
- 可重用显然也是**视觉标准化**能落地的一个重要手段。

## 告警

### 概述 1

**告警的难点是"不"告警**：

- 以 Prometheus 的告警产品 [Alertmanager](https://prometheus.io/docs/alerting/latest/alertmanager/) 为例，它的主要核心概念都是围绕这个目标设计的：
  - Grouping：尽量将相似告警组合起来只给用户发送一个通知。
  - Inhibition：如果已经收到整个集群出问题的告警，就没必要再通知集群上的某个组件也出问题了。
  - Silences：强制静默，通常针对计划中的变更工作。
- 但告警产品也只是提供了这方面的功能，组合哪些告警、按什么顺序抑制等等并**不是**一劳永逸的工作，需要持续调优。
  - Kubernetes / OpenShift 提供了这方面足够实用的默认配置，但一者是针对平台方、二者仍需内部定制。
  - 作为 PaaS 平台方，我们给用户定制了一批通用告警，但缺失二级管理功能：
    - 同一告警对不同用户的影响可能是不一样的，因此需要允许用户自行定制。
    - 自主静默。

### 概述 2

- **要做到不多、不少、恰到好处的告警，前提是可观测性水平的全面提升**：
  - 在分布式环境，一台服务器宕机并不是了不得的事，但如何确认这件事是有多大影响？却不是只检查能不能 Ping 通这台机器的，我们需要做更多的观测，这台机器提供什么服务、该服务的失败率是否在急遽升高等等等等。
    - 因此同一件事（宕机）在不同场景的告警级别可能是 Critical、Warning 甚至只是 Info。
  - 某个应用出问题是因为数据库平台宕了，我们怎样知道它的真实依赖关系？分布式追踪做起来。
    - 当然不是说数据库宕了就不需要通知应用方，但告警时能说明根因是不是舒服很多。
    - 这也可以认为是告警的**多所有者问题**，但所有者关系的获取却很难是静态的，比如知道应用依赖数据库，但如果是数据库平台的某个区出故障了又如何知道影响了哪些应用？
- 这件事很难，也可以理解宁错勿漏的想法，但起码要往这个方向努力，毕竟我们不希望苦大仇深的做运维。
  - 以个人有限的值班经验，半夜发生的告警没有一个是必须抓人马上解决的。

### 概述 3

**在告警前解决问题**：

- 除了挖断光纤这种意外，通常一个 Critical 前会有十个 Warning。
- 每个告警都应该有反馈，即使不紧急。
  - 一个想法：如何自动生成不理会告警的告状？
- 引起故障的一个最常见情况是变更，而对于这种计划内的工作：
  - 夜间变更。
  - 工作日早间，方方面面的工程师都就位的情况下进行，持续观测一整天。
  - 当然理解第一种做法，但后者出了问题响应更及时、支持更充分、人员更清醒，这种**不同的运维哲学**并不是一个完全不能考虑的选项。

### 通知服务

- 我们需要一个支持二级管理、API 友好、提供多个通知渠道的集中式通知服务平台。
- Deadman Switch：在一段时间未收到 Alertmanager 心跳（Watchdog Alert）后发出告警。
- 目前是基于飞书群的自定义机器人实现。

### Runbook

- Runbook 是响应特定告警的针对性运维手册，本质是自助服务。

## 思拷

### 观测

可观测性数据最初也是最主要的一个用途就是 Debug，但是如 [How to Ask for Help](how-to-ask-for-help.md) 所述：

> 虽然对具体的技术越了解、经验越多就越能更快的解决问题，但很多人并未因此形成良好的意识，他们的**心理反应**大多类似"我知道这个软件的日志在哪儿，所以我去看日志"，而不是"所有软件都应该有日志，所以即使我不知道它在哪儿，我也应该先找出来"，以至于一旦进入一个即使跨度不大的新领域也会手足无措，最终就是"一年的工作经验重复了十年"。

### 观测与解读

- 我观测到我的应用在你的平台运行出错，所以……
- 我观测到我的应用本地未出错、但在你的平台运行出错，所以……

如 [How to Ask for Help](how-to-ask-for-help.md) 所述：

> 在平台支持的场景，交叉验证是定位问题出在平台方还是用户方的最重要手段，但这也是**最容易被误用**的手段。

其实，技术问题还只是最简单的问题……

### 不可观测

- 我们观测到开发了业务功能，但我们能不能观测到……
- 我们观测到设定了研发指标，但我们能不能观测到……
- 我们观测到发布了安全制度，但我们能不能观测到……
- 我们观测到设置了审批流程，但我们能不能观测到……
- 我们观测到发生了运维事故，但我们能不能观测到……

### 不可度量

- 做了一件事：`+1`
- 做好一件事：恳请慎重数字化！
  > When a measure becomes a target, it ceases to be a good measure. - Goodhart's Law

### 不可解读

- 多 = 好？
- 快 = 好？
- 控制 = 安全？
- 简单 = 疏漏？

参考：[代码质量和技术债](tech-debt.md)

### 现实是不可能如此直观透明的被观测到！

- [TOO BUSY TO IMPROVE](https://cn.bing.com/images/search?q=too+busy+to+improve&ensearch=1)
