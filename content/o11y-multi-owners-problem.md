---
title: O11y Multi-Owners Problem
---

## Multi-Owners Problem

在 OpenShift 环境，提供了对 core **platform** components 和 **user**-defined projects 的[监控](https://docs.redhat.com/en/documentation/openshift_container_platform/4.18/html-single/monitoring/index#about-ocp-monitoring)，后者的监控 / 指标数据显然属于应用团队等平台用户，但前者却比较复杂：

- 首先平台的监控数据当然全部属于平台团队。
- 但是有部分平台数据是和平台用户相关的，典型如应用团队 Namespace 下某个 Pod 的 CPU、内存等指标，即使不说属于应用团队，至少用户也应该能自主的查询统计。

虽然现代的可观测性后端产品都支持多租户，但默认配置下只是简单的按租户 / Owner 进行隔离，也就是说要么允许用户访问全部的平台数据自然就能看到其他用户的情况、要么禁止用户访问只能请求平台团队支持，显然这都不是理想的方式。当然这些后端产品也提供了细粒度的权限控制，比如 Elasticsearch 的 [Attribute-based access control](https://www.elastic.co/subscriptions) 或 GEM 的 [Label-based access control](https://grafana.com/docs/enterprise-metrics/latest/tenant-management/lbac/)，但是无法做到开箱即用，因为你并不知道这些产品的客户方能不能或者按哪个 Attribute / Label 来区分平台用户。

Multi-Owners Problem 是自己发明的一个词，网上未搜到太多相关的讨论和方案，在此也不用纠结平台用户到底算不算这类数据的 Owner，**本质是对于某类数据、会有多方天然拥有合理的查询权限，如何尽量简洁的实现这个需求**？注意这里的**重点**是简洁而不是能否。而这也远不止是 OpenShift / Kubenetes 这个平台或指标类数据会遇到的问题：

- **所有平台**类的指标数据都面临这个问题。典型如我们需要查询应用交易在 LB 环节的耗时或异常，现在只能是请求等待 LB 平台方去查……
- 除了指标类型的数据，还有**日志**，典型如 Kubernetes 事件或稽核日志，和平台用户相关的数据当然也应该允许他们自主查询，无论是应用团队 Namespace 发生的所有事件，或者记录团队成员操作的稽核日志。
  - 还有**告警**，部分平台告警规则生成的告警实际上只需要通知给用户、不需要平台方操心。
- 还有更**复杂**的场景：
  - 比如在分布式追踪里，是否允许用户在排查错误时查询上游系统的相关日志？
  - 虽然和用户不直接相关，但是比如应用 Pod 所在节点机、VM 所在物理机的运行状态，是否也应该让用户方便的了解？
- 以上只是读（查询）场景，实际上在**写场景**也有类似问题。比如对 user-defined projects 的监控，是由平台方统一采集数据并写入可观测性后端；由于 PaaS 平台方也只是监控平台的一个用户 / 租户，那么监控平台在写入时自然也要控制哪些用户只能写本租户数据、哪些用户可以将数据写入其他租户。

这些问题显然不是单凭产品本身能解决的，因为不可能有一个业界统一的规则来决定平台数据属于哪个或哪些平台用户，自然就无法做到开箱即用；问题解决的好坏完全取决于企业内**可观测性数据治理**的效果，比如不同平台生成的可观测性数据都有同名、语义一致的唯一标签来区分平台用户，那数据越标准自然后续的查询控制就越简单。

另外我们在新云 POC 阶段也咨询过厂商相关问题，但厂商就是一个集群一个租户这种模式，集群上的平台数据自然能都开放给这个集群的唯一用户，似乎没有这方面的强烈需求。

## OpenShift

虽然以上问题无法简单解决，但具体到某一平台，或者进一步简化需求，还是有一些现成且简单的方案，比如 OpenShift 集成的可观察性组件。

### Monitoring

以 OpenShift 内置的监控功能为例演示以上讨论的 Multi-Owners Problem：

1. 以普通用户登录 OpenShift Web Console - Developer - Observe。
1. 进入某个可访问的 Project 比如 `sakura`。
1. 选择 Metrics。
1. Select query - CPU usage。
1. Show PromQL，结果类似如下：
   ```
   sum(node_namespace_pod_container:container_cpu_usage_seconds_total:sum_irate{namespace='sakura'}) by (pod)
   ```
   - 从以上指标名称（`node_namespace......sum_irate`）和相关 Label（这个需要用管理员权限查看）可以确认是平台监控数据。
   - 通过查询条件限定了 `namespace='sakura'`。
1. 尝试手工修改以上查询语句查询一个不属于该用户权限的 Namespace：
   ```
   sum(node_namespace_pod_container:container_cpu_usage_seconds_total:sum_irate{namespace='default'}) by (pod)
   ```
   回车执行时提示 `An error occurred - Bad Request`。
   1. 如果打开 Chrome Developer tools 从 Network tab 可以看到访问的是：
      ```
      https://console.apps.paas-wh-01-uat.example.com/api/prometheus-tenancy/api/v1/query?namespace=sakura&query=sum%28node_namespace_pod_container%3Acontainer_cpu_usage_seconds_total%3Asum_irate%7Bnamespace%3D%27default%27%7D%29+by+%28pod%29
      ```
      而返回的是 400：
      ```
      label matcher value (namespace="sakura") conflicts with injected value (namespace="default")
      ```
   1. 可以发现以上 URL 有两个 `sakura` 参数，一个在 Prometheus 查询语句一个是 `namespace=sakura`，尝试全部修改为 `default` 后直接 curl，这时返回的是 403。
      -  注意 403 是 OpenShift 针对用户访问 Namespace 所作的权限控制，而 400 才是 OpenShift 监控针对用户访问平台数据所作的权限控制。
   1. 尝试移除 PromQL 中的 `namespace='sakura'` 这个条件，查询可以正常执行，但从返回结果可以确认仍只获取了 sakura 这个 Namespace 下的数据。

以上试验就可以看出 OpenShift 同样考虑过本文的问题，解决方案利用了它开源出来的 [prom-label-proxy](https://github.com/openshift/prom-label-proxy#example-use) 项目，我们在下一节讨论其机制后再给出结论。

#### prom-label-proxy

OpenShift 实际是 Fork 的 [prometheus-community/prom-label-proxy](https://github.com/prometheus-community/prom-label-proxy)：

> The prom-label-proxy can **enforce** a given label in a given PromQL query, in Prometheus API responses or in Alertmanager API requests. As an example (but not only), this allows **read multi-tenancy** for projects like Prometheus, Alertmanager or Thanos.

也就是说它会强制将用户的任意查询添加一个指定的限制条件，如上一节的 `namespace`，因此用户也只能获取平台数据中和自己 Namespace 相关的内容了。另外注意：

> This proxy does not perform authentication or authorization, this has to happen before the request reaches this proxy

这也就是上一节 403 和 400 的区别。

虽然 OpenShift 利用 prom-label-proxy 比较轻松的解决了 Multi-Owners Problem 问题，但这个方案有很大**限制**：

- 从 `prom-label-proxy` 名字就可以看出只是针对 Prometheus 指标，不可能涵盖日志类型数据。
- 考虑到可观测性后端会收集所有平台的数据，并不是每个平台的用户 / 租户都叫 `namespace`（如果不在采集或更早的阶段就做标准化）。
- 解决的也只是平台数据首先属于平台、其次属于 Namespace 用户的 Two-Owners Problem，这个方案并不能简单扩展到更多 Owners。
- 数据最终还是要集中采集到长期保存的统一可观察性后端，根据不同的后端产品选型，这个问题仍需重新考虑。

### Logging

我们再来看如何查询日志：

1. 以普通用户登录 OpenShift Web Console - Administrator - Workloads - Pods。
1. 点击进入某应用 Pod 的明细页面。
1. 点击 `Logs` 标签就可以查看到应用自身的容器日志输出。
1. 点击 `Events` 标签就可以查看和这个 Pod 相关的 Kubernetes 事件。
   - 如果没有看到任何事件是因为 OpenShift 默认的 [Amount of time to retain events](https://v1-32.docs.kubernetes.io/docs/reference/command-line-tools-reference/kube-apiserver/)（`event-ttl`）设置为 [3h](https://github.com/openshift/cluster-kube-apiserver-operator/blob/release-4.16/bindata/assets/config/defaultconfig.yaml#L121)。
   - 事件信息实际是从 Kubernetes API Server 查询的（以上 `event-ttl` 就是 API Server 的配置），那么自然是平台方拥有的数据；由于事件不像普通日志那样保存在非结构化的日志文件中，因此是 API Server 在控制普通用户只能查自己 Namespace 的事件。
   - 由于事件只在 API Server 保留 3 个小时，因此 OpenShift 提供了 [Event Router](https://docs.redhat.com/en/documentation/openshift_container_platform/4.6/html/logging/cluster-logging-eventrouter) 将事件转为通常的日志文件形式，然后采集保存到可观测性后端，这样用户就可以通过可观察性平台查看所有的事件，但结果是 API Server 已经解决的 Multi-Owners Problem 又必须在可观测性平台**重新**解决。
1. 当前的 OpenShift 版本（4.16）不支持普通用户查询稽核日志，但因为也是节点机上的日志文件形式，自然也可以采集保存到可观察性后端后统一查询，以及同样也需要在可观察性平台解决 Multi-Owners Problem。

总之和监控一样，无论在 OpenShift 本身解决的如何，这个问题仍需在可观察性后端重新考虑解决。

## 云原生可观察性平台

云原生可观察性平台是 PaaS 团队基于 [Grafana Stack](https://grafana.com/about/grafana-stack/) 自行搭建的服务，叫这个名字仅仅是为了和监控团队负责的平台区分，既不表示只能采集云原生应用的数据、也不表示必须部署在 OpenShift 等云原生环境。但**注意**现在这个平台已基本废弃，全部转到监控团队新的一体化监控平台，此处主要是说明云原生可观察性平台是如何解决 Multi-Owners Problem 的。

### Mimir

由于 OpenShift 监控不支持长期的 Prometheus 数据存储，再考虑到多集群等场景，因此也无法停留在 OpenShift 的现成方案上，总归要输出到可观察性后端。至于后端的产品选型，由于 Thanos 尚不支持 [Multi-Tenancy](https://thanos.io/tip/operating/multi-tenancy.md/)，因此我们基于 Grafana Mimir 的 [Tenant](https://grafana.com/docs/mimir/latest/references/glossary/#tenant) 能力并结合 prom-label-proxy 以解决 Multi-Owners Problem，但如上所述，我们限定在解决最多只有两个 Owner 的情况。

首先是**采集 / 写阶段**，OpenShift 监控数据分为应用指标数据写到 Mimir 的各应用租户、PaaS 平台指标数据写到 Mimir 的 PaaS 租户。注意我们在 PaaS 平台和可观测性平台我们使用的租户概念，我们并没有直接使用 OpenShift 的 Namespace，而是统一使用了公司云管的"业务"；而 Namespace 和"业务"这通过我们约定的规则关联起来，主要是"业务"作为 Namespace 前缀、以及一些 Hard coding 的规则。总之在采集阶段需要处理原始数据到"业务"的映射关系，才能将数据写入正确的租户。
 
在实际写入 Mimir 的过程中，是通过将"业务"信息赋值给 HTTP 头 `X-Scope-OrgID` 来区分租户，无论读写。但是到此只解决了单租户问题，对于本文的主题，还需要考虑 PaaS 平台指标的第二租户也就所属应用的问题；和以上的 prom-label-proxy 处理类似，只不过要 enforce 的 label 不是 `label` 而是我们命名为 `tk_cotenant` 的"业务"信息，这个字段和 `X-Scope-OrgID` 一样，基本按相同的规则从原始数据映射而来，并作为新增 label 保存到 Mimir。在此我们不展开细节，总之**通过 Kubernetes 已足够标准的监控数据和我们设计的约定规则，我们能够较轻松的从原始数据提取到多 Owner 信息**。

当各租户数据正确写入 Mimir 后，**查询 / 读阶段**首先也是通过 `X-Scope-OrgID` 来限制用户只能查询自己应用的租户数据；然后允许所有用户查询 Mimir 的 PaaS 租户，但是在这个查询过程中，通过 prom-label-proxy 限制了仅能查询到平台数据中和自己相关的部分。

至此我们可以说在功能上解决了 Mimir 上的 Multi-Owners Problem，但实际在用户使用上，仍不够满意。因为用户是通过 Grafana 访问 Mimir 数据源来执行查询操作，而以上的实现机制决定了用户查自己租户是直接访问 Mimir、查 PaaS 租户的自己相关部分是通过 prom-label-proxy 间接访问 Mimir，也就是说必须设置成两个数据源，因此对用户的使用可能产生困扰。

### Loki

由于各类可观察性数据的性质差别很大，即使号称统一平台实际的后端产品也很难是同一个，因此我们使用同 Grafana 系的 Loki 作为日志后端，不过在租户机制上和 Mimir 非常相似。由于日志没有类似 prom-label-proxy 这样的工具，且日志信息相对更敏感一些，我们考虑了另外的做法，就是不在读阶段解决，而是**在写阶段将多 Owner 数据复制多份分别写入对应的 Loki 租户**，比如我们在将 Kubernetes 事件、稽核日志写入 Loki 的 PaaS 租户的同时，也写入了相关用户对应的 Loki 租户。这样实际就是将多 Onwer 转为了单 Owner，那么在查询时自然就不需额外工作了，因此也不存在以上 Mimir 查询的麻烦、每个用户只需要一个数据源。

注意以上复制的工作并不是 Loki 完成的，而是采集工具如 Vector 实现的；对于指标数据同样也可以这样做，比如 Prometheus 的 [remote_write](https://prometheus.io/docs/prometheus/latest/configuration/configuration/#configuration-file) 可以同时写入多个目标并作不同的 Relabel；从这一点说，这是一个**万能方案**，不会局限于存不存在类似 prom-label-proxy 的工具。

当然这样做的代价就是数据存储的冗余，按双 Owner 算最多会有一倍。但如果如上简化需求，只考虑以上的事件和稽核数据的复制，还有由此带来的查询架构简化和万能场景，这并不完全是一个被逼无奈才实施的方案。

## 结语

可观测性的 Multi-Owners Problem 是一个常见但实际被有意无意忽视了的问题，无论 OpenShift 的可观测性组件还是我们自己搭建的云原生可观察性平台都只能算勉强解决了，而这个问题显然也不是只依靠产品本身能彻底解决的，可观测性数据治理或标准化是一个非常重要的因素，期望一体化监控平台能重视并更好的解决这个问题。
