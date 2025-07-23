---
title: Kubernetes Namespace TLDR
---

Kubernetes 实操入门之六，面向 Kubernetes / OpenShift 用户或潜在用户，建议先阅读最基本的 [Kubernetes TLDR](k8s-tldr.md)，随之要了解的就是日常接触最多的几个 Kubernetes Object，我们按照 [Pod](k8s-pod-tldr.md)、[Deployment](k8s-deployment-tldr.md)）、[Service](k8s-service-tldr.md)、[Ingress](k8s-ingress-tldr.md)、[ConfigMap & Secret](k8s-configmap-tldr.md)、**Namespace** 的顺序。

## Namespace

- Kubernetes 通过 Namespace 提供了基本的租户隔离，类似 GitLab Group / Jira Project / Confluence Space。
- 应用团队使用 Kubernetes，首先需要平台管理员创建相应的 Namespace，同时也会将应用团队的技术负责人设定为该 Namespace 的管理员（也就是所谓的二级管理员）。
- 而该 Namespace 下的绝大部分工作如应用部署、Day 2 运维、成员管理等等，都由 Namespace 成员在相应权限内完成，平台方有帮助、支持的责任，但并非代劳。
- 平台默认禁止跨 Namespace 的网络访问，但是两个 Namespace 的网络开放仍是由二级管理员自主配置，不需求助平台管理员；但是注意 Service 的调用需要由同 Namespace 的 `<service-name>` 调整为 `<service-name>.<namespace-name>`。
- 无论叫什么名字，如果一个平台没有类似 Namespace 的设计，不能二级管理、自助服务，那就算不上真正的平台。
- Kubernetes Object 会分为 Namespace 相关或无关的（`kubectl api-resources` 可以看到各 Object 的 `NAMESPACED` 是 `true` 还是 `false`），前者的话只需要保证在 Namespace 内不重名即可。

## Manifest

所有的 Kubernetes Object，都是使用 YAML 格式描述其配置、然后以此创建相应实例，不过由于 Namespace 的配置项非常少，通常是一个命令就创建了。以下是 Namespace 的例子：

```
apiVersion: v1
kind: Namespace
metadata:
  name: test-namespace
```

- `apiVersion`：
  - 该字段不止表示版本也包含 API 分组（可以理解为以下 `kind` 的大类），如果不是 `***/v1` 形式明确指出分组则属于 [core](https://kubernetes.io/docs/reference/kubernetes-api/service-resources/service-v1/)，但并不能表示为 `core/v1`。
  - `v1` 表示正式版。不同种类的 Object 有自己的发展成熟周期，从 `v1beta1` 到同时支持 `v1beta1` 和 `v1` 到最后废弃 `v1beta1`，这就是 Kubernetes TLDR 中提及**部分平台变更场景用户必须跟进调整**的原因之一。
- `kind`：Kubernetes Object 的类型。
- `metadata`：
  - `name`：Namespace 实例的名称。

## Advanced

以下主要属于进阶内容，此处不展开讨论，主要是补充之上为了不把主题说得太复杂而省略的部分，也避免引起误解。

- 在 OpenShift 平台，创建 Namespace 会同时创建 OpennShift Project Object，反之亦然，一些扩展功能比如 `openshift.io/sa.scc.uid-range` 当然是跟 Project 挂钩的。
- 对于我们现在的 IT 规模，`Cluster - Namespace` 的两层结构其实也不是很合适，新云 POC 时厂商的做法通常是一个集群一个租户，但是一者基于我们现在的 IaaS 层并不能很轻松的部署 Kubernetes 新集群，二者考虑到多集群部署仍然有别的问题，理想方案应该是跨集群的"Tenant - Namespace"这种模式，但没有现成的。
