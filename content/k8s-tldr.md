---
title: Kubernetes TLDR
---

基本的 Kubernetes 概念解读，面向全体 IT 人员，具体的技术实操见最后 [Run](#run) 一节。

## General

- 了解 Kubernetes 之前通常需要知道一些容器的基本概念，参见 [Container TLDR](container-tldr.md)，但是记住 Kubernetes 的核心并不是容器。
- 虽然以容器方式运行应用已足够简单和标准，但是按照生产可用（**Production Ready**）的要求，比如至少需要两个实例保证基本的高可用、以及顺之而来的前置 LB，再考虑更复杂的 Day 2 Operations，如实例扩容、扩容后的 LB 同步、发布新版的滚动升级等等，这显然不是容器本身层面考虑的事，需要的是容器编排。
- 编排这个词比较抽象，具体必须做什么或者绝不做什么并无定论也无需纠结，知道它的目标是 Production Ready 就基本理解这个词了。很显然 VM 应用部署同样也隐含编排的概念、需要达到编排的目标，所以两者最大的区别是**容器编排有唯一的主流选项和事实标准 Kubernetes**。
- Kubernetes 的特性，如不可变基础设施、申明式部署和全生命周期管理、API 驱动等等，都不是自己发明的全新事物，各家厂商都或多或少实施着形式不同但本质没差别的方案，所以选择 Kubernetes 的重点最终还是以结果论，因为我们需要的是一个开放标准和成熟生态。
- Kubernetes 对自己的定位[不是](https://v1-32.docs.kubernetes.io/docs/concepts/overview/#what-kubernetes-is-not)传统的 PaaS 平台，实际上 PaaS 这个词也没有明确定义；对比 PaaS 和 Kubernetes 等编排系统，前者在对使用场景有更多约束的同时也提供了更多开箱即用的功能，而后者需要用户更多接触底层概念的同时适用场景更多、扩展性更强。
- 也有 Kubernetes 即 CaaS（Container as a Service）的说法，这其实就是个"听君一席话，如听一席话"的效果，而且 Kubernetes 可不是只能编排容器，反正我们还是拿 PaaS 作为我们平台的市场推广词。
- K8s 是日常提及 Kubernetes 的简称，8 为 `K` 到 `s` 之间的字母数，另一个常见的是 O11y（Observability）。
- 和 Linux 一样，我们往往使用更友好、功能更齐全的 Linux distributions 如 CentOS、Ubuntu 等，对应的 Kubernetes Distribution 则是 [OpenShift](openshift-tldr.md) 或 Rancher，而本地部署的 OpenShift 则是 OCP（OpenShift Container Platform）。
- 一个稍具规模的企业都不会只有一个 Kubernetes / OpenShift 集群，因此再发展就会涉及到对集群的编排。

## Opinionated

- **引入 Kubernetes 对传统企业 IT 最大的意义在于它会逼着你走业界的最佳实践，由于事实标准和霸主地位的背书，在挑战五千年的 IT 传统时消除了太多的阻力。**
- 新增一个容器实例，如果还要象物理机 VM 那样坚持打申请走流程等审批，面对 Kubernetes 自动半自动的扩容机制就会显得十分荒谬；而 VM 实际也可以走上这条路的，但因为还可以往回走，就往回走了。但这不代表就不要管控了，领导应该关注的是总量和趋势，而不是每个实例的细节，这里想强调的重点是**部分传统管控已经不合时宜了**，包括以下的内容。
- 以往的 VM 和 LB 是两个环节，分别申请资源后还要记得申请网络权限，但是在 Kubernetes 已经简化成了一个命令，不过因为种种原因我们的 OCP 也部分走回了老路。
- Kubernetes 上的容器 IP 随机分配且未暴露到集群外，按以往的规矩和做法其实已经控制不住了。
- 在 Kubernetes 部署应用后，全办公网的最终用户马上就可以访问；VM 平台方可能比较委屈，这个是技术问题吗？当然不是，但 Kubernetes 解决了。
- 甚至一个细节，最终用户对 Kubernetes 应用的访问必须通过域名和标准端口。

## Change

- 在传统做法中，平台通常定位在核心功能，如果这个领域的核心功能已基本定型那肯定是以稳定为主，即使升级也很少动接口影响用户；但是应用不一样，是要不停发展新功能的，大家可以对比一下平台 Vs. 应用在产品迭代上的频度。
- 不幸的是 Kubernetes 可以认为是半平台半应用，因为容器编排并没有限定功能范围，所以升级频度即使不像 GitLab 应用（每月 22 号中版本升级），但肯定比传统的 DB、VM 要高不少，类似产品还有对象存储。
- 由于 Kubernetes 不局限于核心功能，也决定了它不可能像 DB、VM 那样快的成熟稳定下来（注意是相对，实际已是 Graduated / Conservatives 了，而且 DB、VM 也发展得更早、历史更久）。目前基本是小半年一个中版本，当然它也有这方面的自觉比如 [API Versioning](https://v1-20.docs.kubernetes.io/docs/reference/using-api/#api-versioning) 的设计。
- 基于以上不同，决定了我们很难长时间停留在 Kubernetes 的一个老版本上，不像 MySQL 5 Vs. 8，大部分程序其实用的就是标准 SQL，这样 5 一直用下去当然没问题，除了官方 23 年停止支持。
- 我们也调研过其他真实用户的升级情况，确实有一直用一个老版本而不升级的。但实际情况是人家等不及官方的升级自行扩展了很多功能，到后来是不敢升级升不了级而非不想升级；而我们没有这个能力自行扩展，那么**谨慎紧跟**官方升级只能是唯一选项，除非就想要祖传架构一百年不动摇。
- 对于我们实际运维的 [OpenShift / OCP](openshift-tldr.md) 产品：
  - OCP 3 到 3.11 就不再继续了，对应的 Kubernetes 是 1.11，在当时 Kubernetes 的最新版已是 1.20，而且其他软件比如 GitLab 对集成 Kubernetes 的版本要求最低也是 [1.15](https://docs.gitlab.com/13.10/ee/user/project/clusters/index.html#supported-cluster-versions)，因此我们无法一直停留在 OCP 3。
  - OCP 4 直到 4.10 才支持配置 Prometheus Remote Write，这对我们统一采集监控数据非常重要，因此我们也无法一直停留在 4.8。
- 由于 Kubernetes 的 API Versioning 设计，包括我们内部所做的一些平台设计，导致了**部分平台变更场景用户必须跟进调整**。这个平台方会做充足的准备，权衡收益（对平台方和对用户的）和用户工作量，Kubernetes 也会在升级前自动提示预警。总之，我们希望用户能够理解这种做法，而承认我们设计不足缺少预见、权衡之下承认需要和用户一起调整这并不可怕，**不干扰用户是考量的重点但并不是铁律，实际上一直不动、"没有干扰到用户"才真正可怕！**
 
## Run

Kubernetes 实操入门从了解以下日常最多接触的几个 Kubernetes Object 开始：

- [Pod](k8s-pod-tldr.md)
- [Deployment](k8s-deployment-tldr.md)
- [Service](k8s-service-tldr.md)
- [Ingress](k8s-ingress-tldr.md)
- [ConfigMap & Secret](k8s-configmap-tldr.md)
- [Namespace](k8s-namespace-tldr.md)
