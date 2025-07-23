---
title: Kubernetes Deployment TLDR
---

Kubernetes 实操入门之二，面向 Kubernetes / OpenShift 用户或潜在用户，建议先阅读最基本的 [Kubernetes TLDR](k8s-tldr.md)，随之要了解的就是日常接触最多的几个 Kubernetes Object，我们按照 [Pod](k8s-pod-tldr.md)、**Deployment**、[Service](k8s-service-tldr.md)、[Ingress](k8s-ingress-tldr.md)、[ConfigMap & Secret](k8s-configmap-tldr.md)、[Namespace](k8s-namespace-tldr.md) 的顺序。

## Deployment

- 在 Kubernetes 语境提到的 Deployment 通常是指特定的 Kubernetes Deployment Object，因此请注意上下文，和泛泛的 Deployment / 部署这个词区分开。
- Kubernetes 的一个主要特色就是通过 Deployment 等实现了应用的声明式部署和全生命周期管理。
- Deployment 并不是我们之前习惯的实实在在的部署动作（安装软件、启动服务），可以理解为是一个声明或契约，创建一个 Kubernetes Deployment Object，就类似于现实中签订了一个合同，实际的动作尚未展开；当然在实际生活中，执行合同通常会隔天甚至更久，而 Kubernetes 的后续动作是秒级展开的，但无论多快，本质上同样是分**两阶段**发生的。
- Deployment 声明或合同的要素，包括指定应用的实际运行程序（即容器镜像名称）和应用运行的副本数（即实例数、Pod 数）等。
- 在 Kubernetes 创建 Deployment Object，就类似于提交合同，而 Kubernetes 自然也负责它的合规审查，如果制式不对（配置错误），则不能成功创建 / 签订；注意这个阶段出错一般是部署操作出错、不代表应用运行本身有问题，而反之也成立，能成功创建也不代表应用一定正确运行，这也是让新用户容易产生**困扰**的一个点。
- 但是注意 Kubernetes 的合同审查主要还是格式方面的，比如指定镜像用的是 `image` 这个关键字、拼错了自然会当场发现，但是如果 `image` 所指向的内容并不存在或者需要访问权限，那么是在之后的执行过程中报错。
- 当合同确立后，由 Kubernetes 负责后续执行，即按照指定的容器镜像、运行所指定副本数的容器实例，当然实际上肯定不止这两项，可以看作合同主条款和附件，总之到了这一步才是传统方式中的部署动作。
- 而和现实中仍然一致的地方，Kubernetes 接受的并不是一次性交付合同，还需要负责"保修"，比如说在应用运行期间，如果因各种意外导致某个实例失败，Kubernetes 会负责启动一个新实例、与合同的约定数保持一致，而这种行为也会**持续**至整个合同期（软件生命周期）。
- 而之后如果用户想调整镜像版本或者副本数，就需要修改 Deployment Object，类似于合同变更，Kubernetes 在确认无误后即按新约定执行；或者用户也可以取消合同、删除 Deployment Object，自然 Kubernetes 就会移除该应用的全部容器。
- **总之和现实中的合同一样，Kubernetes 用户只需要按照约定提交自己的需求，所有应用部署的技术细节完全就交给了合同承包商（Kubernetes）去处理**。但这种方式也不是 Kubernetes 独有的，在最原始的部署和 Kubernetes 之间，还有很多选项，所以选择 Kubernetes 的重点仍然是开放标准和成熟生态。
- 当然，这不是只有好处没有代价的，你需要学习 Kubernetes（如何签合同），不过平台方包括 PaaS 和 DevOps 团队也有责任将这个制式合同调整的更简单直观（但不要指望完全不操心），另外比现实还好的一点是你不用担心受骗。

## Manifest

所有的 Kubernetes Object，都是使用 YAML 格式描述其配置、然后以此创建相应实例，以下是 Deployment 的例子：

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 3
  selector:
    ......
  template:
    metadata:
      ......
    spec:
      containers:
      - name: nginx
        image: nginx:1.14.2
```

- `apiVersion`：
  - `apps` 表示 API 分组，可以理解为以下 `kind` 的大类。[apps](https://kubernetes.io/docs/reference/kubernetes-api/workload-resources/deployment-v1/) 这个词语义太宽泛，从它所包含的具体 Object 了解即可，另外对比 Pod，API 分组为默认的 [core](https://kubernetes.io/docs/reference/kubernetes-api/workload-resources/pod-v1/)。
  - `v1` 表示正式版。不同种类的 Object 有自己的发展成熟周期，从 `v1beta1` 到同时支持 `v1beta1` 和 `v1` 到最后废弃 `v1beta1`，这就是 Kubernetes TLDR 中提及**部分平台变更场景用户必须跟进调整**的原因之一。
- `kind`：Kubernetes Object 的类型。
- `metadata`:
  - `name`: Deployment 实例的名称。
- `spec`：绝大部分 Object 都会有这个配置，但该配置下的实际内容则随 `kind` 而不同。
  - `replicas`: 指定应用运行的实例数，对于 Kubernetes 应用也就是 Pod 数。
  - `selector` 和之下 `metadata` 省略的部分是用来将 Deployment 和它所创建的 Pods 关联起来的机制，但不是在 Pod 简单设一个 `parent` 字段之类的做法，而是更灵活的方式，我们在 Kubernetes Label TLDR（TODO）中说明。
  - `template`：所部署应用的实际程序配置，对于 Kubernetes 应用也就是 Pod 配置，该项下的内容说明参见 [Pod TLDR](k8s-pod-tldr.md)。

## Advanced

以下主要属于进阶内容，此处不展开讨论，主要是补充之上为了不把主题说得太复杂而省略的部分，也避免引起误解。

- 类似 Deployment 的"合同"类型并不只有这一个，其他常见的包括 StatefulSet、DaemonSet 和 Job 等等。
- Deployment 也不是直接创建 Pod，而是通过了 ReplicaSet。
- 当然也可以绕过 Deployment 直接创建 Pod，但如果手工删除这个 Pod，就没有一个"合同"让 Kubernetes 继续维持它运行，因此这种做法主要用于一次性或临时性的工作如 Debug；反之，对于 Deployment 间接创建的 Pod，如果用户直接删 Pod 则会自动重建，给人的感觉似乎删不掉，这也是可能让新用户困扰的点。
- 在传统部署中，多为推模式，也就是将程序执行文件推送到待部署的 VM 上然后启动，但 Deployment 在执行阶段是启动 Pod 后由所在节点机的容器运行环境去拉取镜像。
- 如果使用 OpenShift，我们还会看到 DeploymentConfig 这个 Object，它是在 Kubernetes Deployment 未成熟前 OpenShift 自行扩展的替代方案，但是现在 [OpenShift](https://docs.openshift.com/container-platform/4.7/applications/deployments/what-deployments-are.html#deployments-comparing-deploymentconfigs_what-deployments-are) 已明确提示"it is recommended to use Deployment objects unless ......"。不过从语义上讲，DeploymentConfig 这个词更符合 Kubernetes 对 Deployment 的定位。
