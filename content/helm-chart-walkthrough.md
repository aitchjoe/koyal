---
title: 从头创建应用 Chart
---

请**首先**通过 [Helm TLDR](helm-tldr.md) 了解 Helm 和 Helm Chart 的基本概念、以及主要的 Helm 部署命令，之后才是本文主题：应用引入 Helm 的具体实操步骤。另外 Helm 的模板语法可以先通过 [Chart Template Guide](https://helm.sh/docs/chart_template_guide/getting_started/#adding-a-simple-template-call) 了解，也可以之后再说，因为以下示例中的语法基本都能望文生义。

本文使用 Helm [v3.18.4](https://github.com/helm/helm/releases/tag/v3.18.4) 进行，**强烈建议**从该链接下载并跟随文档实操；由于 Helm 安装实际是部署到 Kubernetes，如果没这个环境，可以尝试将以下的 `helm install ...` 命令替换为 `helm template ...`，这样也能直观感受 Helm 模板处理这个核心功能，但 `kubectl` 就不能执行了。

## 脚手架

Helm 提供了 [helm create](https://helm.sh/docs/helm/helm_create/) 命令生成一个新的 Chart 且基本可用，我们以此作为创建自己应用 Chart 的起点。

### 生成

1. 在应用源码根目录下生成 Chart 脚手架：
   ```
   helm create app
   ```
   - 为什么不使用具体的应用名称而是泛泛的 `app`，是因为在创建的 Chart 文件中会用该名称作为变量名，如 `app.name`；大家可以对比一下 `helm create aaa-bbb` 的结果，明显使用 `app` 比真实应用名更简洁直观，而且拷贝到其他应用重用时也无需改动。
   - 以上[命令](https://helm.sh/docs/helm/helm_create/)可以通过 `-p` 参数指定其他的 Helm starter scaffold，比如根据我们以下的意图进行定制，从而省略下面的手工调整步骤；但那样读者就不知道来龙去脉了，因此仍基于 Helm 默认的脚手架演示。
1. 将 `app` 目录更名为 `chart`。
   - 上一步解释了为什么用 `app` 这个名称，但从项目整体视角看 `app` 又容易引起混淆，因此改为更直观体现用途的名称。
   - 如果应用 Chart 需要[打包](https://helm.sh/docs/helm/helm_package/)发布，通常 TGZ 压缩文件中的根目录名应该是项目名称而不是泛泛的 `chart`，但如 [Helm TLDR](helm-tldr.md#chart) 所述发布的必要性不大，真要做在 CI 中重命名也很方便。

### 布局

所生成的脚手架目录大致如下：

```
chart/
├── .helmignore
├── Chart.yaml
├── values.yaml
├── charts/
└── templates/
    ├── _helpers.tpl
    ├── deployment.yaml
    ├── hpa.yaml
    ├── ingress.yaml
    ├── NOTES.txt
    ├── service.yaml
    ├── serviceaccount.yaml
    └── tests/
```

其中核心是 `deployment.yaml`、`service.yaml`、`ingress.yaml`，请先从 [Kubernetes TLDR](k8s-tldr.md#run) 了解基本的背景知识。

### 调整

即使是 Helm 官方脚手架，也有一些未遵循最佳实践的做法：

- 典型如默认的 Service port 是 `80`，实际应该以非 root 用户运行在非特权端口（`1024` 及以上）。
- [Image](container-run-tldr.md#image-name) 未使用完整名称，类似从 `nginx` 调整到 `docker.io/nginx`。

而到我们内网环境 OpenShift 平台：

- 由于 Docker 限流因素，我们推荐使用位于 [AWS](container-public-image.md#可用仓库) 仓库的同一镜像，类似从 `docker.io/nginx` 调整到 `public.ecr.aws/docker/library/nginx`。
- 由于 [OpenShift](openshift-tldr.md#kubernetes-to-openShift) 对安全控制更严，只能使用 [Unprivileged Image](container-run-tldr.md#unprivileged-image)，也就是从 `nginx` 调整到 `nginx-unprivileged`。

按上面的思路调整后，我们运行以下命令看实际效果（以下 `koyal-demo` 也可以是其他任意名称，但注意在后续命令保持一致）：

1. ```
   helm install koyal-demo ./chart --set image.repository=public.ecr.aws/nginx/nginx-unprivileged --set image.tag=1.29.0 --set service.port=8080
   ```
   可以再带上 `--dry-run` 参数查看实际生成的 YAML 内容，但这样就不会真正执行。
1. ```
   kubectl port-forward svc/koyal-demo-app 8080:8080
   ```
1. 在另一个窗口可以如下成功访问：
   ```
   curl localhost:8080
   ```

暂不解释以上命令的细节，后续有讨论，我们先从下面的"精简"开始。

## 精简

Helm TLDR 的 [Chart](helm-tldr.md#chart) 一节说明了为何专业产品的 Chart 看着都那么复杂，但对于我们自己应用的 Chart，考虑到使用场景，其实可以舍弃掉那些不必用的灵活性，这样就能做到非常简洁；所以即使是新生成的脚手架，也有很多可以精简的地方。

### 移除

- `.helmignore`：类似 Git Ignore，将 Chart 目录内容打包成 TGZ 文件时忽略该文件指定的内容。由于我们自己的 Chart 主要做应用的部署、而不是其他应用 Chart 的依赖，打包发布的必要性不大，可以移除。
- `templates/`：
  - `NOTES.txt`：成功执行 `helm install` 后的提示消息，但默认的提示一者是 Linux 命令而我们主要是 Windows 环境，二者是 `kubectl` 而不是我们用的 [oc](openshift-tldr.md#oc)，先暂时移除。
  - `hpa.yaml`：[Horizontal Pod Autoscaling](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/) 高阶功能，根据流量压力自动扩展 Pod 实例数，但实现这个的前提是监控等必须做到位；该功能默认未启用，我们先直接删除，扩展实例用手工方式。
  - `tests/`：
    - `test-connection.yaml`：由于 Annotation 设置了 `"helm.sh/hook": test`，并不会在 `helm install` 时执行，而是执行 `helm test koyal-demo` 时才会去测试指定 Release 的连接；而在真实场景，即使要做自动化的 [Chart Tests](https://helm.sh/docs/topics/chart_tests/)，也不会是这个 Demo 性质的"test-connection"，它通过 [Probe](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/) 就能做到；所以我们先不把问题复杂化，暂时移除该文件及 `tests` 目录。

**注意**以上除了 `.helmignore` 不算我们的使用场景，其他项要么代表专业度、要么代表高阶功能，所以我们**不是**真的用不到、只是在这个入门文档里暂时移除，以下同理。

### Chart.yaml

Chart 元数据如应用名称版本等等。主要的调整项：

- `name`：原值为 `app`，这是因为之前 `helm create` 命令的参数就是 `app`，在此需调整为自己应用的真实 ID，比如 `koyal-demo`。Helm Install 生成的 Kubernetes Object 名称、Label 值等，实际是由这个 Chart Name 和 Release 共同决定的：
  - 如果 Release 是相同的 `koyal-demo`，那么结果就是 `koyal-demo`。
  - 但如果 Chart Name 还是之前的 `app`、两值完全不同，那结果就是 `$RELEASE_NAME-$CHART_NAME` 即 `koyal-demo-app`。
  - 如果两者虽然不同但 Release 包含 Chart Name，比如 `koyal-demo-test`，那么直接就是这个值。
  - 可以通过参数 `nameOverride` 覆盖 Chart Name、或者 `fullnameOverride` 直接取代以上规则生成的结果。
  - 从以上的设计可以看出，目的是为了保证不同 Release 产生的名称、Label 等各不相同、但又避免太重复啰嗦，完整的实现逻辑参见"_helpers.tpl"文件。
- `description`：调整为真实项目描述。
- `version`：Chart 版本。如果应用 Chart 不准备发布，其实这个字段作用不大；因为 Chart 跟源码走，使用同一个 Commit 的程序源码和 Chart 文件进行部署。但由于是必填项，且毕竟有语义表示，因此如果变更了 Chart 本身，可以跟随升级该字段。
- `appVersion`: 原值 `1.16.0` 是脚手架默认 Nginx 应用的版本，真实使用中一般是在部署时通过参数指定，因此可以删除。

最终调整为（已略去全部注释）：

```
apiVersion: v2
name: koyal-demo
description: Helm chart demo
type: application
version: 0.1.0
```

### charts

该目录保存所依赖的其他 Chart，在部署应用自身的同时也部署所依赖的数据库、缓存等中间件 Chart，暂时保持为空。

### templates

该目录保存各 Kubernetes Object 的模板文件以及共享的模板工具。

#### _helpers.tpl

Helm Chart 模板并不都是直接引用传入的变量值，也可能如以上 [Chart.yaml](#chartyaml) 一节所述，是结合多个变量按一定规则加工后的结果，而这就是本 Helpers 文件的作用，定义了常用的**中间变量**供多个模板文件引用；请先通过 [Built-in Objects](https://helm.sh/docs/chart_template_guide/builtin_objects/) 熟悉 Helm 提供的内置变量，典型如 `Release.Name`、`Chart.Name` 等。

- `app.name`：默认为 Chart Name，可以被传入的 `nameOverride` 覆盖。
- `app.fullname`：如以上 Chart.yaml 一节的规则，但可以被 `fullnameOverride` 覆盖。
- `app.labels` 及 `app.selectorLabels`：前者包含后者的所有 Labels，而后者包含**隔离** Release 的关键 Label，也就是 Kubernetes 的 [Well-Known Labels](https://kubernetes.io/docs/reference/labels-annotations-taints/)：
  - `app.kubernetes.io/name`：取值为以上的 `app.name`。
  - `app.kubernetes.io/instance`：取值为 Release Name。
- 由于以上取值可能从多个变量合并而成，而 [Kubernetes](https://kubernetes.io/docs/concepts/overview/working-with-objects/names/) 有长度限制，因此统一按最大 63 字符做了截取。
- 注意在 Chart.yaml 一节我们将 Chart Name 从生成的 `app` 改为了真实的 `koyal-demo`，这调整的是变量值；但以上的 `app` 都是变量名，而所有的模板文件都是引用这个名称，因此不能将这些也改了。

更多中间变量及详细实现参见该文件，但除了研究外，可以将其当作一个黑盒工具不做调整。

#### deployment.yaml

要理解对这个文件的处理，当然首先要了解 Kubernetes [Deployment](k8s-deployment-tldr.md) 的背景知识。我们针对该文件的精简按以下几个思路：

- 暂不启用的高阶功能，如之上提及的 HPA。
- 平台默认就足够实用的配置，如 [securityContext](https://kubernetes.io/docs/tasks/configure-pod-container/security-context/)、[nodeSelector](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/#nodeselector) / [affinity](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/#affinity-and-anti-affinity) / [tolerations](https://kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/)、[imagePullPolicy](https://kubernetes.io/docs/concepts/containers/images/#image-pull-policy)。
- 业务应用基本都是无状态应用，因此也不需要 [Volumes](https://kubernetes.io/docs/concepts/storage/volumes/) 相关的配置。
- Namespace 已设置了 [Limit Ranges](https://kubernetes.io/docs/concepts/policy/limit-range/)，暂时移除 [resources](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/) 的定制配置。
- 更多的定制元数据，如 `podAnnotations` / `podLabels`。
- 不会在多处引用、且变更频率很小的情况，这个不是移除，但也**没必要**用干扰人的模板语法动态处理，直接 Hard coding 就行，典型如 `livenessProbe` / `readinessProbe`。

至此，该文件从 78 行调整到 36 行：

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "app.fullname" . }}
  labels:
    {{- include "app.labels" . | nindent 4 }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      {{- include "app.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      labels:
        {{- include "app.labels" . | nindent 8 }}
    spec:
      {{- with .Values.imagePullSecrets }}
      imagePullSecrets:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      serviceAccountName: {{ include "app.serviceAccountName" . }}
      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          ports:
            - name: http
              containerPort: {{ .Values.service.port }}
              protocol: TCP
          livenessProbe:
            httpGet:
              path: /
              port: http
          readinessProbe:
            httpGet:
              path: /
              port: http
```

注意我们并**不是**为裁剪而裁剪，裁剪后的也不是一个最小 Demo，仍然是一个**基本实用**的配置，比如以下提及的 `imagePullSecrets` 我们就没有删除。在此解读一下裁剪后的内容：

- `metadata`：
  - `name`：设为了 `app.fullname`，以上"_helpers.tpl"一节提及了这个值的由来，主要是为了保证不同 Release 之间不会产生命名冲突；但注意这个名称可以被 `fullnameOverride` 覆盖，也无法保证不通过 Helm 部署的 Deployment 就不叫这个名字，因此实际上还是有冲突的可能，真遇到了就调整 Release 名称等来规避。
- `spec`：
  - `selector.matchLabels` 和 `template.metadata.labels`：如"_helpers.tpl"所述，两者的关键 Label 相同，而 `template.metadata` 代表着由 Deployment 创建的 Pod 的元数据，也就是说通过这两个 Labels 设置可以将同一 Release 下的 Deployment 和 Pod 关联起来，详细说明参见 [Labels and Selectors](https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/)。
  - `template`：
    - `imagePullSecrets`：在企业内部，通常也会控制业务应用容器镜像的读（Pull）访问，因此需要通过该 Secret 设置访问 Token，详细说明参见 [DevOps 访问权限解析](devops-access.md)；但注意这个 Secret 一般是手工创建，而不是作为 Chart 的模板之一。
    - `serviceAccountName`: 参见以下 serviceaccount.yaml 一节的说明。
    - `containers`：
      - `image`：之前的 Tag 取值是 `{{ .Values.image.tag | default .Chart.AppVersion }}`，但由于已在 [Chart.yaml](#chartyaml) 中删除了 `appVersion`，因此移除 `default`。
      - `ports`：注意该配置只是一个申明，其中的 `name` 供以下的 Probe 及 Service 引用；而 `containerPort` 并不能控制容器应用的实际启动端口，具体取值需要通过应用文档了解。
      - `livenessProbe` 及 `readinessProbe`：`httpGet.port` 的取值可以是以上 `ports` 的 `name`、这样就能关联到实际的 `containerPort`，或者也可以直接配置数字类型的实际端口，详细说明参见 [HTTP probes](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/#http-probes)。

#### service.yaml

请先了解 Kubernetes [Service](k8s-deployment-tldr.md) 的背景知识。对这个文件的调整主要是将 `service.type` 从可配置项转为 Hard coding，虽然 [Service type](https://kubernetes.io/docs/concepts/services-networking/service/#publishing-services-service-types) 有很多类，但普通应用最常见的只有 `ClusterIP`，而且不同类型的关联属性也可能不同，比如 `NodePort` 类型还有 `nodePort` 字段；因此没有必要做成伪配置项，真需要调整类型直接修改 YAML 文件就行。

#### ingress.yaml

请先了解 Kubernetes [Ingress](k8s-ingress-tldr.md) 的背景知识。调整如下：

- 在微服务架构中，应用有很大可能不需要暴露到集群外，因此需保留是否启用 Ingress 的配置项（`ingress.enabled`）且默认为 `false`。
- `annotations` 配置项的一个主要作用是通过 Annotation `kubernetes.io/ingress.class` 指定 [Ingress class](https://kubernetes.io/docs/concepts/services-networking/ingress/#ingress-class)，但到 Kubernetes 1.18 已 [Deprecated](https://kubernetes.io/docs/concepts/services-networking/ingress/#deprecated-annotation)，替换为以下的 `ingressClassName`。
  - 这也是 Helm 脚手架兼容 Kubernetes 各版本的考量，但对于我们最低都是 Kubernetes 1.25 的情况，就完全没必要了。
  - 实际上脚手架也没考虑更早版本的情况，那样的话 `apiVersion` 还得考虑除 `networking.k8s.io/v1` 外还有 `networking.k8s.io/v1beta1` 或 `extensions/v1beta1` 的选项。
- 同上，`ingressClassName` 用以指定 Ingress class，但大部分情况下一个 Kubernetes 集群只会有一个 [Ingress Controllers](https://kubernetes.io/docs/concepts/services-networking/ingress-controllers/)，使用默认的就好。
- 在我们平台，应用一般不需要自己暴露 HTTPS 服务并操心证书管理，而是交由 LB 端实现，因此不需要 TLS 相关的配置。 
- Ingress 配置支持一个应用服务配多个域名，但通常只用一个，去除模板复杂的循环语法。
- Ingress 也支持同一域名配多个 Path 转向到不同的应用服务，但也不常用，同样移除循环语法；而在只有一个 Path 的情况下，使用根路径即可。
- [Path types](https://kubernetes.io/docs/concepts/services-networking/ingress/#path-types) 在脚手架是 `ImplementationSpecific`，可能更适配原生的 Kubernetes，对于我们的 [OpenShift](openshift-tldr.md) 平台，使用最常见的 `Prefix` 即可。

最终调整为：

```
{{- if .Values.ingress.enabled -}}
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: {{ include "app.fullname" . }}
  labels:
    {{- include "app.labels" . | nindent 4 }}
spec:
  rules:
    - host: {{ .Values.ingress.host | quote }}
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: {{ include "app.fullname" $ }}
                port:
                  number: {{ $.Values.service.port }}
{{- end }}
```

#### serviceaccount.yaml

背景知识参见 [Configure Service Accounts for Pods](https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/)。早期做法大多使用 Namespace 的默认 Service Account（SA）`default`，但现在更倾向使用各自的 SA，Helm 脚手架默认也是创建 Release 自己的 SA；另外和之前的脚手架比，现在的 SA 多了一个 `automountServiceAccountToken` 设置。参见 [Application Security Checklist](https://kubernetes.io/docs/concepts/security/application-security-checklist/#service-account) 的 Service account 部分：

> **Avoid** using the `default` ServiceAccount. Instead, create ServiceAccounts for each workload or microservice.
>
> `automountServiceAccountToken` should be set to `false` **unless** the pod specifically requires access to the Kubernetes API to operate.

作为初级用户现在不用关心 `automountServiceAccountToken` 的技术含义，按以上的 Checklist 调整即可；而且同一个应用也不会一会儿 `true` 一会儿 `false`，直接 Hard coding 就行。因此最终调整如下：

```
{{- if .Values.serviceAccount.create -}}
apiVersion: v1
kind: ServiceAccount
metadata:
  name: {{ include "app.serviceAccountName" . }}
  labels:
    {{- include "app.labels" . | nindent 4 }}
automountServiceAccountToken: false
{{- end }}
```

### values.yaml

Chart 模板变量的默认值，即模板文件中以  `.Values` 开头的变量。根据以上的所有调整，此处也会删除大量配置项、调整默认的 Image 和 Service port 等等：

```
replicaCount: 1
image:
  repository: public.ecr.aws/nginx/nginx-unprivileged
  tag: "1.29.0"
imagePullSecrets: []
nameOverride: ""
fullnameOverride: ""
serviceAccount:
  create: true
  name: ""
service:
  port: 8080
ingress:
  enabled: false
  host: TBD.example.com
```

至此可以再行验证：

1. 由于已经调整了 `values.yaml` 的默认值，不需要还和之前那样通过 `--set` 调整 Image 等等；但同时添加了启用 Ingress 的参数，验证 Ingress 是否有问题：
   ```
   helm install koyal-demo ./chart --set ingress.enabled=true --set ingress.host=koyal-demo.apps.paas-wh-01-uat.example.com
   ```
1. 注意以下 Service 的名称从 `koyal-demo-app` 调整到 `koyal-demo`，原因参见以上 [Chart.yaml](#chartyaml) 一节的讨论：
   ```
   kubectl port-forward svc/koyal-demo 8080:8080
   ```
1. 在另一个窗口可以如下成功访问：
   ```
   curl localhost:8080
   ```
   ```
   curl koyal-demo.apps.paas-wh-01-uat.example.com
   ```

## Production Grade

如上所述，所有的裁剪都不是单纯为了演示用，裁剪结果仍是一个**基本实用且对初级用户友好**的水平；但即使不考虑如 HPA 等高阶功能，一个真实应用 Chart 在初期大概率就会涉及到 [ConfigMap & Secret](k8s-configmap-tldr.md) 等脚手架未考虑的内容，这一章有一些讨论，主要针对我们最常见的 Spring Boot 应用，但显然也不可能在这个入门文档完整展开。

### ConfigMap & Secret

请先了解 Kubernetes [ConfigMap & Secret](k8s-configmap-tldr.md) 的背景知识。

#### env

以 Spring Boot 应用为例，在部署到不同环境时肯定有不同的配置，典型如数据库连接；Spring 也提供了 [Externalized Configuration](https://docs.spring.io/spring-boot/reference/features/external-config.html) / [Profiles](https://docs.spring.io/spring-boot/reference/features/profiles.html) 等机制灵活配置：

- 不同环境的大部分配置都保存在制品中，比如 `application-prod.yaml` 文件为生产环境配置，并通过 `spring.profiles.active=prod` 启用该配置。
- 但敏感信息如数据库密码等**不能**保存在源码或制品中，只能在部署时注入。

因此当使用 Helm 部署时，会利用到 Kubernetes 的 [Define Environment Variables for a Container](https://kubernetes.io/docs/tasks/inject-data-application/define-environment-variable-container/) 机制，因此需要调整 [deployment.yaml](#deploymentyaml)：

```
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          env:
            - name: SPRING_PROFILES_ACTIVE
              value: {{ .Values.spring.profiles.active }}
            - name: SPRING_DATASOURCE_PASSWORD
              value: {{ .Values.spring.datasource.password }}
```

以上的环境变量名称是 Spring 约定的，类似 `spring.profiles.active`，参见 [Common Application Properties](https://docs.spring.io/spring-boot/appendix/application-properties/index.html)；同时又做了一定规则转换，比如变大写、点号改下划线，参见 [Binding From Environment Variables](https://docs.spring.io/spring-boot/reference/features/external-config.html#features.external-config.typesafe-configuration-properties.relaxed-binding.environment-variables)；而对应的 Values 变量命名没有强制要求，但推荐和 Spring 命名保持一致。如上调整后，即可在部署时通过 `--set` 参数分环境设置以上变量值。

#### envFrom

如果有太多配置项需要分环境设置，比如 `SPRING_DATASOURCE_URL` 在制作容器镜像时还不确定、有连接多个中间件的密码需要设置，这时虽然也能如上一节那样操作，但更正规的做法是先创建 ConfigMap & Secret，类似：

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ include "app.fullname" . }}
  labels:
    {{- include "app.labels" . | nindent 4 }}
data:
  SPRING_PROFILES_ACTIVE: {{ .Values.spring.profiles.active }}
  SPRING_DATASOURCE_URL: {{ .Values.spring.datasource.url }}
  ...
```

然后在 Deployment 将 ConfigMap 中的每一笔 Data 都挂载为环境变量：

```
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          envFrom:
            - configMapRef:
                name: {{ include "app.fullname" . }}
```

Secret 的定义和引用类似，主要是保存数据的性质不一样，类似 `SPRING_DATASOURCE_PASSWORD`、`SPRING_REDIS_PASSWORD` 等等。但这种做法还有问题，从 [Common Application Properties](https://docs.spring.io/spring-boot/appendix/application-properties/index.html) 我们就可以知道有海量的 Spring 配置项，如果我们没有在 ConfigMap / Secret 里定义相应的 Values 变量，自然就没法传入调整了；当应用简单时，使用以上方法也足够，但如果越来越复杂，我们也不必每次要用到一个新配置项就去改一次 Chart，实际可以利用复杂的模板函数来实现，具体不展开，可以参考 OpenSearch 的 [ConfigMap](https://artifacthub.io/packages/helm/opensearch-project-helm-charts/opensearch?modal=template&template=configmap.yaml) 做法。

### Probe

相比上一节创建 ConfigMap 模板等还属于 Helm 主题，探针或健康检查并没有扩展 Helm 的用法，而是在对 Kubernetes 和 Spring Boot 双方健康检查机制的深入理解的基础上，所必须做出的优化；但无论算不算 Helm 主题，这些做法才能让应用 Chart 更 Production Grade，因此本文也应该提及，但详细的讨论请参考 [Kubernetes Probe TLDR](k8s-probe-tldr.md)。

### Subcharts

[Subcharts](https://helm.sh/docs/chart_template_guide/subcharts_and_globals/) 即保存在应用 Chart 的 `charts` 目录下、应用所依赖的其他 Chart，典型如中间件 MySQL、Redis 等；虽然这些要求持久存储的中间件主要运行在 VM 环境，但对于测试或试验性质等非生产、不要求 SLA 的用途，是完全可以在 Kubernetes 上启用的，也就是说即使初级阶段，就需要考虑到 Subcharts 的使用。本文不再完全展开，仅列出其中的基本关注点：

- 从官方或可信渠道获取相应 Chart 的 TGZ 文件。
- 按条件启用或禁用所依赖的 Chart。
- 部署时如何给 Subcharts 传值。
