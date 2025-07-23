---
title: Kubernetes Service TLDR
---

Kubernetes 实操入门之三，面向 Kubernetes / OpenShift 用户或潜在用户，建议先阅读最基本的 [Kubernetes TLDR](k8s-tldr.md)，随之要了解的就是日常接触最多的几个 Kubernetes Object，我们按照 [Pod](k8s-pod-tldr.md)、[Deployment](k8s-deployment-tldr.md)、**Service**、[Ingress](k8s-ingress-tldr.md)、[ConfigMap & Secret](k8s-configmap-tldr.md)、[Namespace](k8s-namespace-tldr.md) 的顺序。

## Service

- 在 Kubernetes 语境提到的 Service 通常是指特定的 Kubernetes Service Object，因此请注意上下文，和泛泛的 Service / 服务这个词区分开。
- 可以将 Service 看作同应用多个 Pod 前的 LB，但注意它不一定是 Nginx / F5 这样的实体，而是使用了 iptables 等机制来实现。
- Service 是 Kubernetes 的核心 Object，在 API 分组中 [Service](https://kubernetes.io/docs/reference/kubernetes-api/service-resources/service-v1/) 和 [Pod](https://kubernetes.io/docs/reference/kubernetes-api/workload-resources/pod-v1/) 都是 `core`，反而 [Deployment](https://kubernetes.io/docs/reference/kubernetes-api/workload-resources/deployment-v1/) 不在核心而是 `apps`。
- 由于 Kubernetes 的目标是生产可用（没有 LB 的单点当然算不上），自然要将部署 Pod 和通过 Service 提供服务一并考虑，实现顺滑的联动，比如新增一个 Pod 则 Service 会自动同步该实例信息。
- 相比 Service 在 Kubernetes 中的地位，在 IaaS 平台可能只有 VM 算一等公民，LB 要么基于 VM 让用户自己部署、要么使用外部独立的 LB 服务，每次新增一个 VM 实例用户还要记得变更 LB，当然好一点的也会有云管之类提供一些半自动化的支持。
- 显然服务调用方应该使用稳定不变的 Service Name 而不是 Service IP 来访问，因此就存在域名解析的需求；而无论 Pod 还是 Service 都是 Kubernetes 集群内分配的 IP，不会暴露到外部，不能也不应该依赖集群外的企业公共 DNS，因此是 Kubernetes 内置 DNS 来解决。
  - 但是注意这里谈及的其实是 Service 的子类型 ClusterIP，这也是普通业务应用最常用的类型，其他还包括 NodePort（可以直接 IP 访问）等。
- 总之 **Service 作为 Kubernetes 平台的一等公民同时参与了 Kubernetes 应用生产级别的部署，不存在传统做法中 VM 和 LB 相对割裂的问题**。

## Manifest

所有的 Kubernetes Object，都是使用 YAML 格式描述其配置、然后以此创建相应实例，以下是 Service 的例子：

```
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  selector:
    app.kubernetes.io/name: MyApp
  ports
    - protocol: TCP
      port: 80
      targetPort: 9376
```

- `apiVersion`：
  - 该字段不止表示版本也包含 API 分组（可以理解为以下 `kind` 的大类），如果不是 `***/v1` 形式明确指出分组则属于 [core](https://kubernetes.io/docs/reference/kubernetes-api/service-resources/service-v1/)，但并不能表示为 `core/v1`。
  - `v1` 表示正式版。不同种类的 Object 有自己的发展成熟周期，从 `v1beta1` 到同时支持 `v1beta1` 和 `v1` 到最后废弃 `v1beta1`，这就是 Kubernetes TLDR 中提及**部分平台变更场景用户必须跟进调整**的原因之一。
- `kind`：Kubernetes Object 的类型。
- `metadata`:
  - `name`: Service 实例的名称。
- `spec`：绝大部分 Object 都会有这个配置，但该配置下的实际内容则随 `kind` 而不同。
  - 以上配置示例隐含了 Service 的默认子类型 `type: ClusterIP`，而 `spec` 下的实际内容也会随 `type` 而不同；但通常也不用明确指定 `type`，Kubernetes 会根据实际的配置项来自动推断子类型，有冲突也会报错。
  - `selector`：用来将 Service 和需要它分发流量的 Pods 关联起来的机制，但不是在 Pod 设一个 `service` 字段之类的做法，而是更灵活的方式，我们在 Kubernetes Label TLDR（TODO）中说明。
  - `ports`：将对外暴露的端口（`port`）和实际提供服务的 Pod 端口（`targetPort`）关联起来，可以关联多对。

## Advanced

以下主要属于进阶内容，此处不展开讨论，主要是补充之上为了不把主题说得太复杂而省略的部分，也避免引起误解。

- 和 Deployment 类似，Service 也不是直接对应到 Pod 的，而是通过 EndpointSlice 或 Endpoint。
- 由于 Service（ClusterIP）的实现机制，无论不会暴露到集群外的 IP、还是集群内置的 DNS，都决定了这种访问不能提供给集群外的请求方，所以 Service 机制主要解决的是集群内应用的互访，外部访问使用的是 Ingress，参见 [Kubernetes Ingress TLDR](k8s-ingress-tldr.md)。
- 可能有用户会困惑之上做的只是服务名解析、但为何使用的是域名解析？实际上这个服务名是有完整域名的，只不过在同一个集群、同一个 Namespace 下，省略了后面的部分，如果要跨 Namespace 访问则名称需要调整为 `<service-name>.<namespace-name>`。
- 有的用户也知道并不是只能通过以上 LB 机制来解决这个问题，比如在 Spring Cloud 是通过 Eureka 动态获取应用的实例 IP；虽然不是域名解析，但本质上仍是把双方事前约定好、稳定的名称在运行时动态转换为实际的 IP，实际上这种机制也叫客户端负载均衡。
