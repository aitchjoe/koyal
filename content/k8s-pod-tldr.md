---
title: Kubernetes Pod TLDR
---

Kubernetes 实操入门之一，面向 Kubernetes / OpenShift 用户或潜在用户，建议先阅读最基本的 [Kubernetes TLDR](k8s-tldr.md)，随之要了解的就是日常接触最多的几个 Kubernetes Object，我们按照 **Pod**、[Deployment](k8s-deployment-tldr.md)、[Service](k8s-service-tldr.md)、[Ingress](k8s-ingress-tldr.md)、[ConfigMap & Secret](k8s-configmap-tldr.md)、[Namespace](k8s-namespace-tldr.md) 的顺序。

## Pod Vs Container

Pod 由一个或多个共享网络、存储的容器所组成，对于应用实例，Kubernetes 直接管理的是 Pod 而非容器，Google 在 [Borg, Omega, and Kubernetes](https://queue.acm.org/detail.cfm?id=2898444) 中讨论了这样设计的原因。

- 一个真实应用除业务主体功能外，通常还会有一些辅助性、支持性的功能如日志管理等，一般做法是将其集成到应用中，不过还有另一种做法，就是主模块和辅助模块既不紧密的耦合在一起、但还是作为一个整体对外提供服务。
- 后一种做法，显然也可以算微服务理念的一种延申，不只业务功能要拆分，非业务的辅助功能也可以拆分，而好处自然也是微服务提及的，不用局限于一种技术栈、有各自的发展节奏、更细粒度的资源配置、控制爆炸半径等等。
- 接受了后一种做法，我们当然就不应该将主辅模块都塞进一个容器、做成一个容器镜像，但由于辅助模块往往响应的是主模块运行时状态，自然需要访问主模块资源，如果是两个独立容器显然会麻烦很多，至此就产生了 Pod 的需求（无论叫不叫这个名字）。
- 而主辅模块从耦合走向解耦，显然也是受到了容器等轻量级技术的影响。在 VM 时代要么将主辅模块部署在一个 VM，以至于无法针对性的分配资源、辅模块出故障也可能影响整个 VM，要么，嗯当时应该完全不会考虑将主辅模块分开部署成多个 VM。
- 当然以我们的情况，比如 Spring Boot 已经集成了庞大的辅助模块库如 Logback，再转向也不现实，因此一个 Java 应用实例通常就只有一个容器，但 Kubernetes 为了保持一致，对这种情况仍是通过 Pod 来间接管理其中的唯一容器。
- 不过注意不要被误导，Pod 的多容器设计针对的是基本纯技术性的辅助功能，但比如一个应用分为前端、后端、数据库，则不应该将这三个容器纳入到同一个 Pod，最简单的判断就是如果仅仅后端应用需要扩容那怎么办？显然这三者是没有主辅之分的，都是服务业务主体功能的不同层面。当然将一个应用作为整体管理也是必须的，但这又是另一个领域的问题了，参见 Helm TLDR（TODO）。
- **总之在日常讨论中非技术人员可以将 Pod 理解为容器而不是又一个全新事物，但是技术人员要清楚相比容器、Pod 代表了一种新的分布式应用部署模式。**

## Manifest

所有的 Kubernetes Object，都是使用 YAML 格式描述其配置、然后以此创建相应实例，以 Pod 为例：

```
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
  - name: nginx
    image: docker.io/nginx:1.14.2
```

- `apiVersion`：
  - 该字段不止表示版本也包含 API 分组（可以理解为以下 `kind` 的大类），如果不是 `***/v1` 形式明确指出分组则属于 [core](https://kubernetes.io/docs/reference/kubernetes-api/workload-resources/pod-v1/)，但并不能表示为 `core/v1`。
  - `v1` 表示正式版。不同种类的 Object 有自己的发展成熟周期，因此用户很可能在某段时间只能使用某类 Object 的 `v1beta1` 版本，但随着发展会同时支持 `v1beta1` 和 `v1`，但因为不能永远停留在过去、最终则会废弃 Beta 版，这就是 Kubernetes TLDR 中提及**部分平台变更场景用户必须跟进调整**的原因之一。
- `kind`：Kubernetes Object 的类型。
- `metadata`:
  - `name`: Pod 实例的名称。
- `spec`：绝大部分 Object 都会有这个配置，但该配置下的实际内容则随 `kind` 而不同。
  - `containers`: Pod 自然是包含一个或多个容器。从名称看使用了复数，从 YAML 类型看使用的是数组类型。
    - `name`: Pod 中该容器的名称。
    - `image`: 容器最基本的属性，决定使用哪一个容器镜像。
- 以上是 Pod 最核心的内容，当然真实情况远不止这几个配置项，仅仅 [Probe](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/) 就需要另开一个 TLDR（TODO）。

## Advanced

以下主要属于进阶内容，此处不展开讨论，主要是补充之上为了不把主题说得太复杂而省略的部分，也避免引起误解。

- 虽然在 Kubernetes 可以依据以上 Manifest 直接创建 Pod，但真实使用中往往通过 Deployment 间接进行，参见 [Kubernetes Deployment TLDR](k8s-deployment-tldr.md)。
- Pod 是 Kubernetes Object，但也可以脱离 Kubernetes 使用 Podman 运行。
- 主辅模块拆分的这种模式，或者说边车模式（Sidecar），我们将在 Service Mesh TLDR（TODO）详细讨论。
