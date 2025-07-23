---
title: Kubernetes Ingress TLDR
---

Kubernetes 实操入门之四，面向 Kubernetes / OpenShift 用户或潜在用户，建议先阅读最基本的 [Kubernetes TLDR](k8s-tldr.md)，随之要了解的就是日常接触最多的几个 Kubernetes Object，我们按照 [Pod](k8s-pod-tldr.md)、[Deployment](k8s-deployment-tldr.md)、[Service](k8s-service-tldr.md)、**Ingress**、[ConfigMap & Secret](k8s-configmap-tldr.md)、[Namespace](k8s-namespace-tldr.md) 的顺序。

## Ingress

- Ingress 是将 Kubernetes 服务暴露到集群外的机制，目前主要支持 HTTP 服务，相比之下 Kubernetes Service 主要针对集群内应用互访的场景且支持 TCP。
- 我们的 Kubernetes / OpenShift 部署通常以无状态的业务应用为主，这类应用一般也都是 HTTP，而需要数据持久化、以 TCP 为主的中间件则使用集群外的服务如青云平台，因此 Ingress 的这个限制对日常使用场景影响不大。
- Ingress 实现通常是集群内置了 Nginx 或 HAProxy 作为反向代理，所有 Kubernetes 应用的外部访问都由它们转发。以 OpenShift 的 HAProxy 为例，用户每创建一个 Ingress Object，就是在 HAProxy 中添加一条相应配置，典型如应用域名和实际部署 IP 的绑定。
- 而 Nginx 或 HAProxy 本身也需要保证高可用，因此它们的多实例通常由集群外的 4 层 LB 来分发流量。
- 从集群外访问 Kubernetes 应用的完整调用链路为："**客户端 - 集群外 LB - Ingress - Service - Pod**"。但注意这只是一个抽象描述，实际上从 OpenShift 的 HAProxy 配置可以确认，网络流量是从 HAProxy 直接分发到了 Pod。
- 由于以上的实现机制，外部用户只能通过域名访问 Kubernetes 应用，而平台也提供了类似 `*.apps.paas-wh-01-uat.example.com` 的泛域名，这样就能马上达到最终用户可用的状态；虽然方便但在实际使用中，强烈建议在生产环境或正式的联调测试环境**申请专门的独立域名**，除了更规范外，一个判断依据就是如果我们之后提供多集群如 `paas-bj-01`、`paas-bj-02`，无论应用是多集群部署还是切换集群部署，应用的调用方是否也需要变更调用地址？
- 由于某些原因，在生产环境不要用平台公共的 LB，各应用需**自行**申请 LB 并指向 OpenShift 的 Ingress 节点，同时将应用域名指向该 LB。
- 用户接触 Ingress 后一个**常见的误用场景**就是同集群内的应用间也走 Ingress，这显然延长了调用链路，走 Service 即可即使是跨 Namespace。

## Manifest

所有的 Kubernetes Object，都是使用 YAML 格式描述其配置、然后以此创建相应实例，以下是 Ingress 的例子：

```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: example
spec:
  rules:
    - host: example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: test
                port:
                  number: 80
```

- `apiVersion`：
  - `networking.k8s.io` 表示 API 分组，可以理解为以下 `kind` 的大类，对比 [Service](https://kubernetes.io/docs/reference/kubernetes-api/service-resources/service-v1/) 为 `core` 核心组。
  - `v1` 表示正式版。不同种类的 Object 有自己的发展成熟周期，相比 Pod、Deployment、Service 等，Ingress 较晚直到 Kubernetes v1.19 才成为稳定版本，而我们平台在这之前就投入了使用，导致用户有使用 `networking.k8s.io/v1beta1` 甚至 `extensions/v1beta1` 的情况，虽然每次平台升级都不会马上废弃过时版本，但是多次升级到现在的 v1.23，就只支持正式版了，仍在使用 Beta 版的则会出错，这就是["**部分平台变更场景用户必须跟进调整**"](k8s-tldr.md#change)的原因之一。
- `kind`：Kubernetes Object 的类型。
- `metadata`:
  - `name`: Ingress 实例的名称。
- `spec`：绝大部分 Object 都会有这个配置，但该配置下的实际内容则随 `kind` 而不同。
  - `rules`：可以配置多项 Rule，就是说可以通过不同域名、不同路径访问同一个或不同的 Service。
    - `host`：Ingress 必须通过域名访问集群内服务。
    - `http`：目前 Ingress 只支持 HTTP 服务，该项配置主要就是和特定 Service 关联起来。

## Advanced

以下主要属于进阶内容，此处不展开讨论，主要是补充之上为了不把主题说得太复杂而省略的部分，也避免引起误解。

- 上 Kubernetes 后一个常见的用户需求是流量控制、IP 白名单等 API Gateway 功能，由于 Ingress 的底层实现就是 Nginx / HAProxy 等，所以理论上都做得到，实际上通过 `nginx.ingress.kubernetes.io/***` 标签配置也可以部分实现，但从这个标签名就可以看出是 Ingress 的 Nginx 扩展（否则命名应该是 `ingress.kubernetes.io/***`），这样做显然是有隐患的，比如 Ingress 的底层实现换为 HAProxy，这就类似于开发应该只使用 Interface 提供、而非 Implementation 的额外功能。总之如果官方未将 Ingress 功能从基本的接入扩展到 API Gateway 范畴，我们**强烈不建议**在 Ingress 层面实现这类需求。
- 以上的部分需求，可以使用新兴的 Kubernetes [Gateway](https://kubernetes.io/docs/concepts/services-networking/gateway/) 这个 Object 来实现，和以上 Nginx 扩展方式的不同在于这是 Kubernetes 的标准规范，目前和 API Gateway 团队正在验证中。
- 当然 Service Mesh 或 API Gateway 平台是更完整的解决方案。
- 如果使用 OpenShift，我们还会看到 Route 这个 Object，它是在 Kubernetes Ingress 未成熟前 OpenShift 自行扩展的替代方案，目前这两个 Object 都可以使用、甚至 Route 具备更多功能，但是我们的**原则**还是能使用 Kubernetes 原生 Object 就不要使用 OpenShift 的扩展版本，即使是在 OpenShift 平台。
- 在 OpenShift 创建 Ingress 时仍会生成 Route Object，但和直接创建 Route 性质还是不一样的，因为我们使用的是 Ingress 接口，保证了最大的 **Kubernetes 平台兼容性**。
