---
title: Kubernetes ConfigMap & Secret TLDR
---

Kubernetes 实操入门之五，面向 Kubernetes / OpenShift 用户或潜在用户，建议先阅读最基本的 [Kubernetes TLDR](k8s-tldr.md)，随之要了解的就是日常接触最多的几个 Kubernetes Object，我们按照 [Pod](k8s-pod-tldr.md)、[Deployment](k8s-deployment-tldr.md)、[Service](k8s-service-tldr.md)、[Ingress](k8s-ingress-tldr.md)、**ConfigMap & Secret**、[Namespace](k8s-namespace-tldr.md) 的顺序。

## ConfigMap

- Kubernetes 通过 ConfigMap 实现应用的**集中式配置管理**，一个应用的多个 Pod 实例都是挂载同一个或一组 ConfigMap。
- ConfigMap 实际就是键值对，值可以是一行或多行的文本内容。
- 相比其他 Object，ConfigMap 本身的结构很简单，因此理解它的**重点**是如何将配置挂载、注入到 Pod，或者说 Pod 如何引用 ConfigMap。
- 常见的两种方式是将键值设为容器的环境变量或指定路径下的文件，而如何使用则是应用程序自身的机制了，比如 Spring Boot 的 [Externalized Configuration](https://docs.spring.io/spring-boot/docs/3.1.5/reference/html/features.html#features.external-config)。
- 当应用启动后重新加载变更过的配置并没有想象中简单，但是 Kubernetes 可以用一种简单粗暴的方式解决，就是启动新 Pod 替换之前的，而容器的轻量级和 Kubernetes 的滚动升级保证了这样做的代价很小，甚至可以说一劳永逸的解决了这个问题。

## Manifest

所有的 Kubernetes Object，都是使用 YAML 格式描述其配置、然后以此创建相应实例，由于 ConfigMap 的重点是 Pod 挂载，因此分以下两部分示例。

### ConfigMap

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: env-config
data:
  LOG_LEVEL: DEBUG
  LOG_another_config: something
```

- `apiVersion`：
  - 该字段不止表示版本也包含 API 分组（可以理解为以下 `kind` 的大类），如果不是 `***/v1` 形式明确指出分组则属于 [core](https://kubernetes.io/docs/reference/kubernetes-api/service-resources/service-v1/)，但并不能表示为 `core/v1`。
  - `v1` 表示正式版。不同种类的 Object 有自己的发展成熟周期，从 `v1beta1` 到同时支持 `v1beta1` 和 `v1` 到最后废弃 `v1beta1`，这就是 Kubernetes TLDR 中提及**部分平台变更场景用户必须跟进调整**的原因之一。
- `kind`：Kubernetes Object 的类型。
- `metadata`：
  - `name`：ConfigMap 实例的名称。
- `data`：和绝大部分 Object 都有的 `spec` 字段不同，ConfigMap 的主体是数据字段。
  - 配置数据就是键值对，支持单行或多行文本，遵循 YAML 语法。

### Pod

注意实际使用时通常是在 Deployment 的 `template` 做以下 Pod 配置。

```
apiVersion: v1
kind: Pod
metadata:
  name: test-pod
spec:
  containers:
    - name: test-container
      image: public.ecr.aws/docker/library/busybox:1.36
      command: [ "/bin/sh", "-c", "env|sort" ]
      envFrom:
      - configMapRef:
          name: env-config
      env:
      - name: LOG_LEVEL_2
        valueFrom:
          configMapKeyRef:
            name: env-config
            key: LOG_LEVEL
  restartPolicy: Never
```

- Pod Manifest 解读参见 Kubernetes Pod TLDR，此处只关注 Pod 对 ConfigMap 的引用。
- `envFrom`：将 ConfigMap 中的全部键值对设为环境变量。
  - `configMapRef`：
    - `name`：上一步的 ConfigMap 名称。
- `env`：可以指定将 ConfigMap 中的某个键值对设为环境变量，比上一种方式灵活的是可以调整变量名称。
  - `name`：所设置的环境变量名称。
  - `valueFrom`：
    - `configMapRef`：
      - `name`：上一步的 ConfigMap 名称。
      - `key`：引用 ConfigMap 中的哪个键值对。
- 实际运行的容器会出现 `LOG_LEVEL=DEBUG`、`LOG_LEVEL_2=DEBUG` 和 `LOG_another_config=something` 三个环境变量。
- 将 ConfigMap 的配置数据挂载为容器中的文件在语法上会更复杂，也需要使用 Kubernetes 的存储机制，也就是说会把 ConfigMap 当成一种存储的来源，此处不再展开。

## Secret

- 在 Kubernetes 语境提到的 Secret 通常是指特定的 Kubernetes Secret Object。
- 在目前的实现上，可以将 Secret 等同于 ConfigMap，也就是说 Secret 基本上不比 ConfigMap 更安全。
- Secret 主要是从**语义**上区分应该存放敏感信息而已，可能在将来会针对 Secret 做更周全的保护。
- 相比 ConfigMap，Secret 会对值做一个 Base64 转码但并不是加密，主要是为了保存二进制证书等。
- 当然 Secret Manifest 是 `kind: Secret`。

## Advanced

以下主要属于进阶内容，此处不展开讨论，主要是补充之上为了不把主题说得太复杂而省略的部分，也避免引起误解。

- 在某些情况下 Secret 还是会比 ConfigMap 更安全一点，比如当 Secret 以文件形式挂载到 Pod 时，只会保存在[内存](https://v1-27.docs.kubernetes.io/docs/concepts/storage/volumes/#secret)中。
- 当应用启动后如果变更了配置，大部分情况并不是应用将内存中的某个变量替换为新值这么简单，因此集中式配置管理方案通常也只是负责通知应用有配置变更发生，而不会去考虑真正的处理，比如 [Spring Cloud Config](https://docs.spring.io/spring-cloud-config/docs/3.1.5/reference/html/#_push_notifications_and_spring_cloud_bus)。
- 事实上在 Kubernetes 里的处理也没有以上提及的那么简单，如果 Pod 引用了某个 ConfigMap，之后修改了 ConfigMap 里的配置数据，如果是环境变量的形式则不会同步到 Pod（即使挂载为配置文件也[分情况](https://v1-27.docs.kubernetes.io/docs/concepts/configuration/configmap/#mounted-configmaps-are-updated-automatically)、不一定同步）；而不管有没有同步，Kubernetes 都不会重建 Pod，因为这个时候 Pod 的 Manifest 并没有发生变化，Kubernetes 也没有将"ConfigMap 变了就要重建 Pod"这种逻辑硬编码。
  - 一种常见的应对是不固定 ConfigMap 的名字、而是带上了实际内容的 Hash 值，这样每次修改配置数据自然就会生成新的名称，然后调整 Pod 引用新的 ConfigMap，这样 Pod 的 Manifest 也发生了变化自然就会滚动升级。
  - 很显然如果是人工操作以上的做法会很繁琐，因此如 Kustomize 等工具会针对这种场景做相应的自动化处理，但 Helm 不行？
  - 还有 [Immutable ConfigMaps](https://v1-27.docs.kubernetes.io/docs/concepts/configuration/configmap/#configmap-immutable) 的做法。
