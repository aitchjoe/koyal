---
title: OpenShift TLDR
---

## General

- [CNCF](https://www.cncf.io/)（云原生计算基金会）对 OpenShift 的定位是 [Kubernetes](k8s-tldr.md) 的[发行版](https://landscape.cncf.io/guide#platform--certified-kubernetes-distribution)，类似 Red Hat Enterprise Linux（RHEL）之于 Linux。
- 可以将 OpenShift 理解为一个品牌或项目伞，包含众多本地部署产品和云上的托管服务，其中主要一个就是我们 [*PaaS*](why-here.md) 平台使用的 [OpenShift Container Platform](https://www.redhat.com/en/technologies/cloud-computing/openshift/self-managed)（OCP）。
- 不采用原生 Kubernetes 而是红帽 OpenShift 的原因和 Linux Vs. RHEL 的道理一样，作为重要平台，稳定性和厂商支持是关键因素，红帽对 OpenShift 的定位也是 [Enterprise Kubernetes](https://www.redhat.com/en/blog/enterprise-kubernetes-with-openshift-part-one)。
- 虽有 OpenShift 是 Kubernetes 发行版的说法，但实际上 OpenShift 做了很多"[What Kubernetes is not](https://v1-32.docs.kubernetes.io/docs/concepts/overview/#what-kubernetes-is-not)"的工作，比如 [CI/CD](https://docs.redhat.com/en/documentation/openshift_container_platform/4.19/html-single/cicd_overview/index)、[可观测性](https://docs.redhat.com/en/documentation/openshift_container_platform/4.19/html-single/observability_overview/index)等等。
- OCP 集成或者说默认提供可观测性等组件，自然方便了用户，而且也磨合得更好；但问题是这些组件通常会基于开源方案，却在迭代节奏上远赶不上独立的上游开源项目，我们实际遇到过这种状况，在需要一个功能特性且上游已提供的情况下，仍需等待 OCP 中版本升级。
  - 红帽也注意到了这个问题并进行调整，比如从 4.8 开始 OpenShift Logging "[with a distinct release cycle from the core OpenShift Container Platform](https://docs.redhat.com/en/documentation/openshift_container_platform/4.8/html-single/logging/index#release-notes)"，但这也只是加快了节奏。
  - 而且 OpenShift 通常会对开源软件进行一定封装，也就是说即使底层是最新的开源版本，也不代表用户就肯定能用上所有特性。
  - 这个问题并不是红帽或 OpenShift 特有的，本质上还是一种工程权衡。
- 无论发行版还是 PaaS，业界对这些词都没有明确的定义，没有规定必须做什么、一定不能做什么……所以我们勿需纠结，基于 OCP 这个产品定制我们的 PaaS 平台就好；而用户也勿需纠结，平台提供了什么功能直接用起来体验，将 PaaS 理解成我们平台的宣传词就好。
- 但总归 OpenShift 是 [With its foundation in Kubernetes](https://docs.redhat.com/en/documentation/openshift_container_platform/4.19/html/architecture/architecture)，因此学习 OpenShift 仍要从 Kubernetes 概念和常见的 [Objects](k8s-tldr.md#run) 开始。
  - 注意从官方资料学习时，[OpenShift 文档](https://docs.redhat.com/en/documentation/openshift_container_platform/)是假定读者有 Kubernetes 基础的、至少不会详细解释，因此前置知识仍需从 [Kubernetes 文档](https://kubernetes.io/docs/home/)学起。

## Products

- 我们实际使用的本地部署产品是 OpenShift Container Platform，通常简称为 **OCP**；而 OCP 产品我们跨越了两个大版本，早期是 OCP 3 但现在已全部转到 OCP 4。
- 从 2018 年开始搭建 PaaS 平台，到 2025 年已发展为多个 [*OCP 集群*](why-here.md)，现在更适合的应该是 OpenShift Platform Plus 这个产品，因为支持[多集群](https://www.redhat.com/en/technologies/cloud-computing/openshift/platform-plus)管理及安全等，这是"[most complete solution](https://www.redhat.com/en/technologies/cloud-computing/openshift/self-managed)"。
  - 不过由于信创因素，现在已开始引入国产系统作为替代，但 OpenShift 应该还会存在相当长的一段时期？
- 另一个本地部署产品是 [OpenShift Kubernetes Engine](https://www.redhat.com/en/technologies/cloud-computing/openshift/kubernetes-engine)（OKE、原名 OpenShift Container Engine），在 [Red Hat OpenShift portfolio: A choice of container solutions](https://www.redhat.com/en/resources/openshift-container-platform-datasheet#section-3) 有这三个产品的详细比较。
  - OKE 比 OCP 主要少"Serverless, Service Mesh, and Pipelines"，由于各种因素，我们 PaaS 平台实际使用的也就是 OKE 功能。
  - 由于 OKE 只是 OCP 的"subset"，没有专门的 OKE 文档，参见 [About OpenShift Kubernetes Engine](https://docs.redhat.com/en/documentation/openshift_container_platform/4.19/html/overview/oke-about)。
  - 如果我们的 PaaS 平台做到极大规模、集群数量众多，那要么升级到 Plus 版作为更完整的解决方案、要么使用 OKE 降低成本，但现在规模不算大，再考虑信创，继续沿用 OCP 即可。
- 还有一个本地部署产品 [OpenShift Virtualization Engine](https://www.redhat.com/en/technologies/cloud-computing/openshift/virtualization-engine)，这个对标的是 VMWare，当然目前的成熟度和稳定性远不能跟后者比，但从长远来讲，更希望用 Kubernetes 等开放体系或标准来管理 VM。
- 其他产品如 OpenShift Dedicated、OpenShift Service on AWS 等等，知道是[公有云服务](https://www.redhat.com/en/technologies/cloud-computing/openshift/openshift-cloud-services)就行。
- OKD 开源项目不算红帽 OpenShift 的产品条线，也不是 OpenShift 的上游，而是"a [**sibling**](https://okd.io/docs/project/#what-is-okd) Kubernetes distribution to Red Hat OpenShift"。
  - OpenShift 还有一个亮点是 [Red Hat Enterprise Linux CoreOS (RHCOS)](https://docs.redhat.com/en/documentation/openshift_container_platform/4.19/html/architecture/architecture-rhcos)（请通过链接了解如"Controlled Immutability"等重要特性），虽然 RHCOS 也是开源的，但 OKD 转向了更开放的 [Fedora CoreOS](https://okd.io/docs/project/faq#what-are-the-relations-with-ocp-project-is-okd4-an-upstream-of-ocp)。
  - 总之即使不考虑厂商支持，可能也很难轻松的将 OpenShift 替换为 OKD。

## Compatibility

本节主要关注 OpenShift 和 Kubernetes 两者的兼容，不是讨论版本升级带来的兼容问题；而且主要从用户应用运行的角度，不涉及平台方在运维上要做的改变。

### Kubernetes to OpenShift

讨论在原生 Kubernetes 平台正常运行的应用迁移到 OpenShift 平台可能遇到的问题。

- 这里的原生指基于 [Kubernetes](https://kubernetes.io/) 开源项目搭建的平台，而不是 OpenShift、[Rancher](https://www.rancher.com/) 等可能做了一定扩展的 Kubernetes 发行版；如果是后者，那么首先要如下节一样考虑所做扩展的兼容性。
- OpenShift 是 [Certified Kubernetes - Distribution](https://landscape.cncf.io/?group=certified-partners-and-providers)，但从 [Certified Kubernetes Software Conformance](https://www.cncf.io/training/certification/software-conformance/) 来看，认证范围主要是"supports the required **APIs**"，而这**不代表** Kubernetes 平台上的应用在不做任何调整的前提下就肯定能转到 OpenShift。
- 虽然业界早有以非 root 用户运行应用的最佳实践，但 OpenShift 默认的安全控制更严，要求普通业务应用的容器能**以十位数的随机 UID 运行、无权限问题**，这是早期从 Docker、Kubernetes 转 OpenShift 时最常遇到的问题。虽然这是运行时发生的问题，OpenShift 技术上也可以放松控制，但正规方式还是在容器镜像制作阶段去解决，参见 [Unprivileged Image](container-run-tldr.md#unprivileged-image)。
- Kubernetes 对 [Ingress](k8s-ingress-tldr.md) 的底层实现没有任何倾向，由用户["select at least one ingress controller"](https://kubernetes.io/docs/concepts/services-networking/ingress-controllers/)，通常本地部署时会选择 [Ingress NGINX Controller](https://github.com/kubernetes/ingress-nginx)，而 OpenShift 是基于 HAProxy。
  - 即使有以上差别，如果用户在 Kubernetes 上使用的是 Ingress 标准配置，那么迁移到 OpenShift 也没问题；可以将其理解为只调用接口，无论接口下的实现是什么，都没影响。
  - 但如果用户在 Kubernetes 的 Ingress 配置中使用了 Nginx 相关的 [Annotations](https://github.com/kubernetes/ingress-nginx/blob/ingress-nginx-3.15.2/docs/user-guide/nginx-configuration/annotations.md)，用以更细粒度的 Nginx 控制，那么切换到 OpenShift 的 HAProxy，这些配置自然不会生效；这明显是**违背**了只使用接口的原则，在 Kubernetes Ingress TLDR 的 [Advanced](k8s-ingress-tldr.md#advanced) 一节有进一步讨论。

### OpenShift to Kubernetes

讨论在 OpenShift 平台正常运行的应用迁移到其他 Kubernetes 平台可能遇到的问题。

- OpenShift 基于 Kubernets 但做了相当多的扩展，如果用户有使用，那别的平台自然无法识别，也就有了所谓的"兼容性"问题。
  - 显然从红帽或 OpenShift 的角度，不会"为了方便迁移到别的平台"而官方建议不要用扩展功能。
- 从用户使用的角度，这种兼容问题有两个因素：
  - 一种是 OpenShift 扩展的 Object，无论是因为添加功能特性还是当时 Kubernetes 原生的 Object 不成熟，以下 [Objects](#objects) 一节会讨论取舍的细节。
  - 一种是 Kubernetes 完全不考虑的领域，比如 OpenShift 在 CI/CD 领域推的 [Source-to-Image (S2I)](https://docs.redhat.com/en/documentation/openshift_container_platform/4.19/html/cicd_overview/ci-cd-overview#openshift-builds) 等方案。
    - 这类方案只能具体情况具体分析。以 S2I 为例，我们强烈不建议采用，至少我们不提供这方面的支持；原因从结果论，没成气候，业界有更专业的产品更主流的方案。
    - 由于 OpenShift 集成方案的实现最终也会落到扩展 Object，因此并不能像一个独立产品比如 Jenkins，只要是使用了标准的 Kubernetes 部署方式在 OpenShift 安装，那自然也能基本原封不动的在别的平台安装。
- 总之，即使用的是 OpenShift 产品，但是**从搭建 PaaS 平台的初始，我们就坚持"尽量使用 Kubernetes 原生功能及方案"的原则**，而已使用的 OpenShift 扩展功能在原生方案成熟后也主动推进用户转换。
  - 这其中的考量就是尽可能的拥抱**开源开放**，不要绑定在红帽 OpenShift 或任何一个特定的厂商，厂商对我们的主要作用是提供高可用保障，但并不是我们的方向。

## Objects

- 请先从 Kubernetes TLDR 的 [Run](k8s-tldr.md#run) 一节了解常见的 Kubernetes Object，这是理解以下 OpenShift Object 的基础。
- DeploymentConfig：在 Kubernetes [Deployment](k8s-deployment-tldr.md) 未成熟前 OpenShift 自行扩展的替代方案，但 OCP 4 最早版本就明确["recommended to use Deployments"](https://docs.redhat.com/en/documentation/openshift_container_platform/4.1/html/applications/deployments#deployments-comparing-deploymentconfigs_what-deployments-are)。
- Route：在 Kubernetes [Ingress](k8s-ingress-tldr.md) 未成熟前 OpenShift 自行扩展的替代方案，到 OCP 4.6 提示["Creating a route through an Ingress object"](https://docs.redhat.com/en/documentation/openshift_container_platform/4.6/html/networking/configuring-routes#nw-ingress-creating-a-route-via-an-ingress_route-configuration)。
  - 无论创建 DeploymentConfig 还是 Deployment 的连锁反应都是直接创建 Pod，对比 Route 直接创建的是 HAProxy 的配置记录，而 Ingress 直接创建的是 Route、由 Route 而 HAProxy；但这样做也只是 OpenShift 这样设计而已，理论上肯定能做到由 Ingress 直接生成 HAProxy 配置。
  - Route 可以通过 Annotation `haproxy.router.openshift.io/timeout` 来 [Configuring route timeouts](https://docs.redhat.com/en/documentation/openshift_container_platform/4.19/html/networking/configuring-routes)，还有更多 Annotation；但这不是现在还使用 Route 的理由，Ingress 的 Annotation 一样可以**同步**到 Route。
    - 这种做法**违背**了只使用接口的原则，参见以上 Kubernetes to OpenShift 一节。
- Project：和 Kubernetes [Namespace](k8s-namespace-tldr.md) 类似，都是["the unit of isolation and collaboration"](https://docs.redhat.com/en/documentation/openshift_container_platform/4.19/html/project_apis/project-apis)，但注意它不是替代品、而是 Namespace 的补充扩展，以上链接有详细说明。
  - Project 和 Namespace 是一一对应的，创建一个会自动生成另一个。
  - Project 多出了 Quota、Resource limits 等属性，这对企业应用场景很重要；因此虽然倾向于尽可能使用 Kubernetes 原生 Object，但在 OpenShift 平台还是应该直接使用 Project。
- 注意扩展本身不代表就是二等公民，甚至可以说 [CRD](https://kubernetes.io/docs/tasks/extend-kubernetes/custom-resources/custom-resource-definitions/) 扩展机制是 Kubernetes 成功的一个重要因素，因此以上的取舍是在功能重合的前提下作出的。
- 以上只是最常见的 OpenShift Object，且和 Kubernetes 原生 Object 有重合需要对比说明，实际上远不止这几个。
