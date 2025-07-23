---
title: Helm TLDR
---

## Helm

[Helm](https://helm.sh/) 是 [Kubernetes](k8s-tldr.md) 包管理工具、部署 Kubernetes 应用的主流方式。

1. 要理解 Helm，首先要知道 Kubernetes 最基本的部署方式，参见 [Deployment](k8s-deployment-tldr.md) 说明，[OpenShift 实操入门](openshift-start.md) 也有实际演示。主要做法就是：
   1. 创建 YAML 文件如 app-deployment.yaml，其中的 Deployment Object 用以创建真正执行程序的 [Pod](k8s-pod-tldr.md) 并保证其高可用。
   1. 执行 `kubectl apply -f app-deployment.yaml` 命令在 Kubernetes 生成该 Object。
1. 而一个典型的 Kubernetes 应用通常还包含以下 Object：
   1. [Service](k8s-service-tldr.md)：Pod 前的虚拟 LB。
   1. [Ingress](k8s-ingress-tldr.md)：暴露给集群外用户访问。
   1. [ConfigMap & Secret](k8s-configmap-tldr.md)：在开发时不能确定、或者部署时需要调整默认值的配置项，通过这类 Object 挂载到具体运行的 Pod。
1. 因此一个真实应用的部署需要执行多个 `kubectl apply -f app-****.yaml`。
   - 或者包含 [Multiple documents](https://yaml.org/spec/1.2/spec.html#id2760395) 的单个 YAML 文件，这样可以一次 `kubectl apply -f app.yaml`。
1. 而更复杂的是一个后端应用通常依赖数据库缓存等中间件，可能在 VM 环境会找专门团队部署，但 Kubernetes 上却很方便用户自行部署、自助服务。
1. 再进一步，由于一个应用也会部署到测试、生产多个环境，各环境的配置总有不同之处：
   - 简单的比如外部访问域名不一样。
   - 复杂的如测试环境使用同样部署在 Kubernetes 内的 MySQL、而生产环境则连到有专门团队支持的外部数据库平台。

当情况如上所述越来越复杂，如果不想写 app-test.yaml、app-prod.yaml 多个文件而且大部分内容还是重复的，那自然会考虑使用模板保存共性内容，并实现变量替换、条件分支等动态功能。这也就是 Helm 的由来：

- Helm 基于 [Go template language](https://helm.sh/docs/chart_template_guide/functions_and_pipelines/) 实现了模板功能，当然 Helm 也不只是一个模板渲染工具。
- Helm 将 Kubernetes 应用的多个 Objects 和依赖作为一个整体管理，并且规范了布局和命名等等，这个整体的 Package 叫做 [Chart](#chart)。

不过更重要的是 Helm **生态圈**已足够成熟：

- 比如我们部署自己应用的同时也需要将 MySQL 部署到 Kubernetes，我们并不需要操心 MySQL 的 Deployment、Service 等等应该怎样写，直接引入现成的 [MySQL Chart](https://artifacthub.io/packages/search?kind=0&ts_query_web=mysql&verified_publisher=true) 即可。
  - 从这一点说，Helm 和 Maven 很象，由于**先发优势**两者的制品库已纳入市场绝大部分主流软件，即使 Helm 的 Go 模板很不友好、Maven 的 pom.xml 配置很繁琐，人们再怎么诟病还是不得不使用 Helm / Maven 格式的制品；即使 Gradle 能取代 Maven 工具、使用的依然是 Maven 制品库。
- 所以如果仅针对 Helm 的模板功能，如果只是一个非常简单的场景，我们在 CI 脚本用 `sed` 替换变量更轻松；另外也有 Kubernetes 集成的很专业的 [Kustomize](https://kubernetes.io/docs/tasks/manage-kubernetes-objects/kustomization/) 方案，但只要有依赖，我们就不得不又回到 Helm，也就是以下的 Chart。

## Chart

- [Helm](#helm) 是包管理工具，而 Helm Package 就叫做 [Chart](https://helm.sh/docs/topics/charts/)。
- Chart 自然包含了上一节讨论的内容，主要是各种 Kubernetes Object 的 YAML 文件，当然其中的内容需要 Helm 动态处理后才能使用。
  - Helm 部署使用 [Helm Install](https://helm.sh/docs/helm/helm_install/) 命令，典型参数如下：
    ```
    helm install -f myvalues.yaml myredis ./redis
    ```
    ```
    helm install --set name=prod myredis ./redis
    ```
    其中变量无论是通过文件（`myvalues.yaml`）传入还是直接传入（`name=prod`），Helm 都会使用值（`prod`）替换 Chart 模板文件的变量（`name`）、或者不同值走不同的分支。
- Chart 规范了包内容的目录结构：
  - `Chart.yaml` 为 Chart 说明或元数据。
  - `templates` 目录包含主要 Kubernetes Object 如 [Deployment](k8s-deployment-tldr.md)、[Service](k8s-service-tldr.md)、[Ingress](k8s-ingress-tldr.md) 的模板文件。
  - `values.yaml` 包含模板文件中变量的默认值。
  - 完整说明参见 [The Chart File Structure](https://helm.sh/docs/topics/charts/#the-chart-file-structure)。
- Chart 当然也有和 [Docker Hub](container-run-tldr.md#public-registry) 类似的 [Artifact Hub](https://artifacthub.io/)，但后者不止 Chart 这一类制品。
- Chart 是一种软件制品（以上 **Artifact** Hub 就是个明示），但和其他绝大部分制品有**很大差别**：
  - Chart 由数量有限的纯文本文件组成，且发布时会压缩成一个 TGZ 文件，因此体积非常小。
  - 因为体积够小，Chart 把全部的直接间接依赖都纳入到了同一个文件；这是绝大部分制品都不会考虑的做法，只能保存一个关系索引指向所依赖的制品。
  - 由于以上特性（包含全部依赖且体积小），以及之前很多 Chart 文件是散布在各处如 GitHub Releases、内部还需要配置 JFrog 镜像，就没有使用 [helm pull](https://helm.sh/docs/helm/helm_pull/) 方式，而是直接手工下载 TGZ 文件后纳入到了项目源码管理；而项目自己的 Chart，大多是一个直接运行的应用而不是其他应用 Chart 的依赖，发布的必要性不大，所以不用打包压缩直接跟源码走即可。
- 完整实操请参见[从头创建应用 Chart](helm-chart-walkthrough.md)，由于 Helm Chart 本质就是一种模板，因此开发人员使用 Chart 的难度还是在 [Kubernetes](k8s-tldr.md) 的概念上，这是学习的**重点**。
- 但实际上在我们研究业界专业的 Chart 如 [OpenSearch](https://artifacthub.io/packages/helm/opensearch-project-helm-charts/opensearch?modal=template&template=statefulset.yaml) 时，即使只考虑模板处理，也会觉得太复杂了。
  - 这是因为一个专业产品需要尽可能的考虑众多客户的不同环境，设置各种选项、兼容不同 Kubernetes 发行版及版本、甚至还做一些抽象封装等等。
  - 而 Go Template 比较丑陋的语法也加剧了这个问题，当满屏都是大括号、为了保证 YAML 格式又不能做有层次的缩进，这对读者实在是太不友好了。
    - 虽然理论上可以将 Chart 模板当成一个黑盒不去研究内部，但从实践经验来看远达不到这个理想状态。
    - 社区也有这方面的讨论，如 [I don't want to use go templates](https://github.com/helm/helm/issues/8290)、[Proposal: Jsonnet template integration](https://github.com/helm/helm/issues/2577)。
  - 但对于一个**内部应用**，部署环境和方式都很明确，因此没必要设置太多动态选项，不通过变量传入、直接硬编码、需要时再变更源码，这并不是一个大忌讳；实际上[从头创建应用 Chart](helm-chart-walkthrough.md) 的重点工作就是"精简"，更简洁、对用户友好要优于不必要的灵活。

## Command

本节主要讨论和应用部署相关的命令，这也是最常用的，完整命令列表参见 [Helm Commands](https://helm.sh/docs/helm/helm/)。我们从 [Helm Install](https://helm.sh/docs/helm/helm_install/) 的第一个示例开始：

```
helm install -f myvalues.yaml myredis ./redis
```

- `./redis`：这是未打包的 Chart 目录，也可以使用打包好的文件如 `./nginx-1.2.3.tgz`，以上链接有详细说明，一共"six different ways"。
- `myredis`：[Release](https://helm.sh/docs/glossary/#release)，中文翻译成[发布版本](https://helm.sh/zh/docs/glossary/#发布版本)。
  - 注意将"**发布版本**"和"**应用版本**"区分开，举例说明可能更直观。在开发阶段，对于同一应用我们通常会部署多套，比如开发用、内部测试用、联调用，这个对应到 Release 上，可能就是 `myapp-dev`、`myapp-test`、`myapp-uat`，这就是发布版本；而每一个发布版本比如 `myapp-uat`，应用本身也在迭代升级，从 `v0.1.0` 到 `v0.2.0` 等等，这就是应用版本。
  - 由于两个都是版本，之后还会提到发布版本的版本（revision），中文说起来确实容易混淆，日常交流中我们就使用 Release 这个词。
  - 但以上例子也只是说明一个典型的 Release 使用场景，根据实际情况也可以是 `myapp-xxx`、`myapp-yyy`……
  - Helm 部署的 Kubernetes Object 的名称、[Labels and Selectors](https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/) 等都会包含 Release 名称，因此 `myapp-uat` 的 [Service](k8s-service-tldr.md) 只会对应到 `myapp-uat` 的 Pod 上去，这自然就产生了不同 Release 的**隔离**。
  - 如上，我们完全可以在 Kubernetes 的一个 Namespace 内做到**同一应用多套部署**，没有必要还按 VM 的模式创建多个 Namespace `myproj-d`、`myproj-t`。
  - 当然要注意在同一 Namespace 是不是已经有该 Release 存在了，如果重名则会出错。
- `-f myvalues.yaml`：如之上讨论的 Helm 机制，我们需要将部署时才能决定的变量值传给 Chart 模板，`myvalues.yaml` 就是包含变量和实际值的 YAML 文件。
  - 通常我们也会针对不同的 Release 创建不同的 Values 文件，如 `v-test.yaml`、`v-uat.yaml`、`v-prod.yaml`。
  - 由于这些文件通常也会随源码保存，因此**注意**敏感信息不要使用这种方式，而是类似文档里的第二种方式：
    ```
    helm install --set name=prod myredis ./redis
    ```
    在实际使用比如 CI 环境中类似 `--set password=$ENV_PASSWORD`，并通过其他安全机制获取敏感信息。
- `--dry-run`：这个选项帮助**排查问题**很有用，很多命令都有；启用它就不会实际执行命令，只是根据 Chart 模板和传入变量值生成待提交的 YAML 内容。
- 由于 Helm 本质上就是部署 [Deployment](k8s-deployment-tldr.md) 等 Kubernetes Object，因此也如该链接所说，`helm install` 成功也"**不代表**应用一定正确运行"。

其他和部署相关的命令：

- [helm list](https://helm.sh/docs/helm/helm_list/)：列出当前 Namespace 的全部 Release。
  - 因此可以在 Install 之前使用该命令查看是否有重名。
  - 也可以使用简短命令 `helm ls`，文档里未提及，但执行 `helm list --help` 有 Aliases 说明。
- [helm upgrade](https://helm.sh/docs/helm/helm_upgrade/)：升级 Release，前提是该 Release 已存在。
  - 虽然这个命令很多时候是为了升级应用版本，但两者不是完全等同的，也有可能是只调整了一个变量值、或者修改了 Chart。
  - 可以使用 `helm upgrade --install` 命令做到 Release 不存在时 `helm install`、存在时 `helm upgrade`，在 CD 场景通常使用这种方式。
- [helm rollback](https://helm.sh/docs/helm/helm_rollback/)：回滚到 Release 的前一状态，这个"状态"英文用的"revision"、中文用的"版本"。
  - 典型场景当然是 Helm Upgrade 后发现应用有问题，先回滚到之前的正常状态再来排查。
- [helm uninstall](https://helm.sh/zh/docs/helm/helm_uninstall/)：完整卸载，包括 Helm 的 Release 信息及创建的几乎所有 Kubernetes Object。
  - Helm Upgrade 可能会因为 Bug 等因素无法升级，这个时候彻底卸载后重新 Install 一般就会成功，但这主要针对无状态应用，特别是在生产环境**务必谨慎**使用该方式。

## Advanced

- 现在（2025-06）讨论的 Helm，基本上就是指 Helm 3，和 Helm 2 在机制上有很大差别（如 Tiller），[Changes since Helm 2](https://helm.sh/docs/faq/changes_since_helm2/) 有详细说明，但我们也不用深入了解，知道这个情况就行。
- 之前 Chart 的发布可以看成就是保存在一个文件服务器上，但是后来也 [Storing Helm Charts in **OCI Registries**](https://helm.sh/blog/storing-charts-in-oci/)，而这代表主流的[容器制品库](https://helm.sh/docs/topics/registries/#use-hosted-registries)如 Docker Hub 都可以保存 Chart 了。
  - 虽然 OCI Registries 希望成为一个"store more than container images"的 Common Storage，但个人认为至少 Chart 真没必要，因为它的特性（包含全部依赖且体积小），文件服务器足够了，即直观、手工下载又方便，
- 执行 `helm install` 这个命令成功**不代表**应用实际部署成功，需添加以下参数，[OpenShift 实操入门](openshift-start.md#upgrade-failed)的 UPGRADE FAILED 一节有详细演示：
  ```
      --wait     if set, will wait until all Pods, PVCs, Services, and minimum number of Pods of a Deployment, StatefulSet, or ReplicaSet are in a ready state before marking the release as successful. It will wait for as long as --timeout
  ```
  启用该功能应该是一个最佳实践，特别是在 CD 领域，如 Flux [HelmRelease](https://fluxcd.io/flux/components/helm/helmreleases/#install-configuration) 的 `disableWait` 默认为 `false`；但 Flux、Argo CD 等并不是简单的执行 Helm 命令，有它们自己持续的 [Reconciliation](https://fluxcd.io/flux/concepts/#reconciliation) 机制，从这个角度说，一个命令是否带 `--wait` 也不是重点。
- 在开头讨论批量 `kubectl apply` 时就有一个问题，包含多个 Object 的同一应用的部署是应该能做到事务性的，而 [Helm Install](https://helm.sh/docs/helm/helm_install/#options) 确实支持：
  ```
      --atomic   if set, the installation process deletes the installation on failure. The --wait flag will be set automatically if --atomic is used
  ```
  这显然也能说明 Helm 远不止是一个模板工具。
