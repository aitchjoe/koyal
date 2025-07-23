---
title: OpenShift 实操入门
---

请先通过 [OpenShift 快速了解](openshift-tldr.md)熟悉基本概念，本文为 OpenShift 用户的入门实操指导（Getting Started）。

## 试验场

基于如下原因我们设置了 OpenShift 的试验场（Playground / Sandbox）：

- 虽然 Kubernetes、OpenShift 有提供 [minikube](https://kubernetes.io/docs/tutorials/hello-minikube/) 或 [OpenShift Local](https://developers.redhat.com/products/openshift-local/)，方便在个人电脑上学习或开发，但限于我们当前的桌面配置，连基础的[容器](container-run-tldr.md#development-environment)都很难在本地运行。
- 有的用户团队已申请了 OpenShift Namespace，但即使测试集群的工作很多也是正式任务，所运行的应用不能随意破坏，因此并不适合学习阶段的新手。

用户通过以下两个途径在 OpenShift 试验场学习试用：

- 测试集群 `paas-wh-01-uat` 的 Namespace [*sandbox*](why-here.md)：所有登录用户**默认**具备该 Namespace 的 `Edit` 权限，也就是说能部署或卸载应用。
- 所有的试验集群（参见 [*PaaS 集群清单*](why-here.md)）：请联系 PaaS 平台[*支持人员*](why-here.md)创建用户专属的 Namespace，用户具备该 Namespace 的管理权限；如果特别要求，甚至可以尝试集群管理操作。
  - OpenShift 支持 [self-provision](https://docs.redhat.com/en/documentation/openshift_container_platform/4.18/html/building_applications/projects#disabling-project-self-provisioning_configuring-project-creation)，也就是说登录用户可自行创建 Namespace / Project，但还要考虑命名规范及限制创建数量，暂未放开。

前一种方式用户勿需任何申请就能直接使用，我们认为这**非常重要**，"申请、需要批准"哪怕是最简单的流程都会**打消**很大一部分人的尝试冲动；而后者的好处是用户不受干扰且可以尝试二级管理操作，但现在尚不能做到用户登录时自动创建。另外注意**一定要清楚试验场的性质**，比如 Namespace [*sandbox*](why-here.md) 是共享的、能被全部用户访问，而所有试验场都会不通知用户就进行清理，因此一定不要作为正式用途；之前由于用户不熟悉试验场的概念，甚至在 GitLab 的 [*sandbox*](why-here.md) 组保存了真实代码。

## 客户端工具

OpenShift 平台提供了 Web Console，对用户更友好，但从学习的角度仍建议从命令行开始，其中主要使用的客户端工具是 [oc](openshift-tldr.md#oc) 及 [helm](helm-tldr.md)，实际还有[更多](https://docs.redhat.com/en/documentation/openshift_container_platform/4.19/html/cli_tools/cli-tools-overview#cli-tools-list)。由于墙的因素国外下载网站不稳定，通过代理的下载速度也堪忧，而能不能方便的下载工具同样会影响用户尝试，因此我们提供了内网下载：

- `oc`：[官方下载](https://docs.redhat.com/en/documentation/openshift_container_platform/4.19/html/cli_tools/openshift-cli-oc#installing-openshift-cli) / [*内部下载*](why-here.md)（仅 4.16.46 Windows 版）
  - 其实 OpenShift Web Console 也提供了 `oc` 的内部下载，登录后点击右上角"帮助"并选择"Command Line Tools"就可以下载各个版本；但有一个问题，因为我们现在部署了多套[*集群*](why-here.md)且版本不同，从特定集群只能下载对应版本，而客户端应该使用高版本，因此新手就别折腾了，有经验的用户可以在确认高版本集群后从 Console 下载，但内部高版本基本也不会有最新补丁。
- `helm`：[官方下载](https://helm.sh/docs/intro/install/#from-the-binary-releases) / [*内部下载*](why-here.md)（仅 v3.18.6 Windows 版）
- 由于内部下载为手工同步自官方来源，版本有限且不保证实时更新，用户如有需要可自行从官方获取。

虽然试验场位于 OpenShift 集群，但用户的尝试是在个人电脑上使用客户端工具交互，因此我们主要提供 Windows 环境的安装说明。由于命令行工具通常就是单个执行文件（虽然压缩包内还有说明文件等），对于这种情况推荐以下方式安装：

1. 创建 `C:\bin` 目录并将其加入系统 `Path` 变量，这是一次性工作。
1. 直接将工具压缩包的执行文件保存到该目录，其他文件略过。
1. 执行 `oc version` 或 `helm version` 确认是否就是新下载的版本。
   - 用户可能会在某些文档看到 `ocw` 这个命令，执行时调成 `oc` 即可，原由见最后安全运维一节的"wrapper"链接。

## 命令行登录

1. 根据以上[试验场](#试验场)一节所采用的方式确认使用哪个 OpenShift 集群。
1. 登录该集群的 Web Console，各集群具体网址参见 [*PaaS 集群清单*](why-here.md)。
   - 一般都会提供 LDAP 登录，请尽量使用这种方式，但由于非技术因素有测试环境的集群只能通过 SSO 登录。
1. 点击右上角用户名并选择"Copy login command"会跳转到另一个窗口。
   1. 如果之前是 LDAP 登录则需要再登录一次，如果是 SSO 则略过这一步。
   1. 窗口会显示"Display Token"按钮，点击展开。
   1. 展开后将"Log in with this token"之下的命令复制到 Windows 命令行窗口执行，大致的命令形式：
      ```
      oc login --token=sha256~****** --server=https://api.paas-wh-01-uat.example.com:6443
      ```
1. 登录成功后的简单验证：
   1. 执行以下命令确认自己可以访问哪些 Project / Namespace：
      ```
      oc projects
      ```
      如果未成功登录则会显示 `Error from server (Forbidden): ......`。
   1. 切换到准备使用的 Namespace 如：
      ```
      oc project sandbox
      ```
   1. 显示当前使用的 Namespace：
      ```
      oc project
      ```
      输出类似如下，同时也包含所登录的集群信息，因此**强烈建议**在每次执行变更类操作时都先用该命令确认一下：
      ```
      Using project "sandbox" on server "https://api.paas-wh-01-uat.example.com:6443".
      ```
   1. Helm 不需要专门登录，只要 `oc` 登录成功即可，执行以下命令确认：
      ```
      helm ls
      ```
      如果登录成功则显示当前 Namespace 下的 Helm Release（可能结果为空），如果未登录则是 `Error: list: failed to list: secrets is forbidden: ......`。
1. 退出登录集群，或者过期（默认 [accessTokenMaxAgeSeconds](https://docs.redhat.com/en/documentation/openshift_container_platform/4.18/html/authentication_and_authorization/configuring-internal-oauth) 是 24 小时）后自动退出：
   ```
   oc logout
   ```

另外 Web Console 步骤也不是必须，可以如下直接登录（将星号替换为 LDAP 用户密码）：

```
oc login api.paas-wh-01-uat.example.com:6443 --username=****** --password=******
```

但这种方式必须集群支持 LDAP 登录，如果只有 SSO 登录就不可行，而且用 Token 可能也更安全？

## Hello oc

**注意**：

- 如果使用的是共享试验场，为避免和其他用户产生重名冲突，请将以下的 `xxx` 统一调整为其他字符串，保证试验场内唯一即可；推荐使用自己的用户 ID，但由于 Kubernetes 的命名规范，需将其中的 `_` 改为 `-` 或移除。
- 每次执行变更类操作之前，都使用 `oc project` 确认当前集群和 Namespace，以避免误操作；我们会在文档中时时提醒，但不再给出具体命令。

### Pod

我们使用以下的 [Pod](k8s-pod-tldr.md) 示例：

```
apiVersion: v1
kind: Pod
metadata:
  name: nginx-xxx
spec:
  containers:
  - name: nginx
    image: public.ecr.aws/nginx/nginx-unprivileged:1.29.1-alpine3.22
```

1. 拷贝以上内容到新建 YAML 文件，比如 `app-pod.yaml`。
1. 确认当前集群和 Namespace，然后创建 Pod：
   ```
   oc apply -f app-pod.yaml
   ```
   如果以上不使用固定的 `name: nginx-xxx` 而是可以生成唯一随机名的 `generateName: nginx-`，那么应该执行 `oc create ...`。
1. 观察 Pod 创建过程（有可能创建过快导致看不到中间过程，可以另开窗口在上一步之前执行）：
   ```
   oc get pods -w
   ```
   结果类似以下（实际很可能有其他 Pod 混在其中，可以过滤但先别复杂化了）：
   ```
   NAME        READY   STATUS              RESTARTS   AGE
   nginx-xxx   0/1     Pending             0          0s
   nginx-xxx   0/1     ContainerCreating   0          2s
   nginx-xxx   1/1     Running             0          15s
   ```
   - 以上各种 `STATUS` 的命名很直观；`READY` 后一位数字代表一个 Pod 总共有多少容器、前一位数字代表已经有多少个容器就绪。
   - 待状态变成 `Running` 后 `Ctrl+C` 退出该命令。注意由于 Namespace 都有**配额限制**，超额后就无法成功创建，遇到这种情况先按最后销毁步骤的提示处理。
1. 查看 Pod 详细信息：
   ```
   oc get pod/nginx-xxx -o yaml
   ```
   对比以上 YAML 的原始内容，已添加了相当多的默认配置项和运行时状态。
1. 查看 Pod 日志：
   ```
   oc logs nginx-xxx
   ```
   实际查看的是 Pod 中的容器日志，因为该 Pod 只包含了一个容器就不用明确指定，但也可以 `oc logs -c nginx nginx-xxx`。
1. 登录到容器内部：
   ```
   oc rsh nginx-xxx
   ```
   在容器内部可以执行任意可用命令，但如果是变更类命令也应该先确认当前集群和 Namespace：
   - ```
     ps -ef|more
     ```
1. 确认当前集群和 Namespace，然后销毁 Pod：
   ```
   oc delete pod/nginx-xxx
   ```
   - 试验完后请**务必**销毁 Pod 以免占用资源。
   - 也可以执行 `oc delete -f app-pod.yaml`。
   - 使用以上"观察 Pod 创建过程"的同样方式观察销毁过程。
   - 如果容器应用复杂，可能会长时间停在 `Terminating` 状态，能够通过 `--force` 等参数强行删除，但这属于进阶内容，未理解前不要随意使用。
   - 如果在之上创建阶段因超额失败，可以用 `oc get pods` 列出所有 Pod，优先删除其中 `AGE` 最大的。
     - 但注意只能在试验场如此操作，其他情况**务必**和所有者联系确认。
     - 直接删除 Pod 不一定能清理干净，参见下一节 Deployment。

### Deployment

先熟悉以上 [Pod](#pod) 一节的内容，其中提及的操作不再详细说明。我们使用以下的 [Deployment](k8s-deployment-tldr.md) 示例：

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-xxx
spec:
  selector:
    matchLabels:
      app: nginx-xxx
  template:
    metadata:
      labels:
        app: nginx-xxx
    spec:
      containers:
      - name: nginx
        image: public.ecr.aws/nginx/nginx-unprivileged:1.29.1-alpine3.22
```

1. 拷贝以上内容到新建 YAML 文件，比如 `app-deployment.yaml`。
1. 确认当前集群和 Namespace，然后创建 Deployment：
   ```
   oc apply -f app-deployment.yaml
   ```
1. 查看 Deployment 列表及明细：
   ```
   oc get deployment
   ```
   ```
   oc get deployment/nginx-xxx -o yaml
   ```
1. 查看该 Deployment 下的 Pod：
   ```
   oc get pods -l app=nginx-xxx
   ```
   - Pod 的名称不再是固定的，而是以 Deployment 名称为前缀添加两段随机字符串，类似 `nginx-xxx-59b7d9fcb5-8cs9g`。
   - 但 Deployment 和 Pod 的关联并不是通过名称，而是以上 YAML 里的 `labels` 和 `matchLabels`，以下会有进一步讨论。
1. 确认当前集群和 Namespace，然后进行以下的删除试验：
   1. 将星号替换为之上获取的具体 Pod 名称：
      ```
      oc delete pod/nginx-xxx-***-***
      ```
   1. 重新查看该 Deployment 下的 Pod：
      ```
      oc get pods -l app=nginx-xxx
      ```
      可以发现指定的 Pod 确实已删除，但几乎同时又重建了一个 Pod（名称中除了最后一段随机字符串都保持不变），这就是 [Deployment](k8s-deployment-tldr.md) 的机制。
   1. 删除 Deployment：
      ```
      oc delete deployment/nginx-xxx
      ```
      继续查看 Deployment 及 Pod，即使没再明确执行删除 Pod 的命令，两者也都清理干净了；但注意还有 StatefulSet 等和 Deployment 类似机制，也就是说遇到 Pod 会自动重建的情况，不仅是 Deployment、也可能是 StatefulSet 等其他 Object 引起的。

通过以上试验就可以理解 Deployment 更适合正式使用，以保证服务的高可用；但这不代表直接创建 Pod 就只有演示用途，我们在 Debug 时通常是直接创建一次性使用的 Pod 而非 Deployment，但配置的镜像一般不会是 Nginx 这种服务、而是 [Busybox](https://gallery.ecr.aws/docker/library/busybox) 等工具。

最后可以用更复杂的试验来验证以上提及的 Deployment 和 Pod 的关联问题：

- 故意将 `labels` 和 `matchLabels` 中的 `app` 改成不同值，会发现无法成功创建 Deployment：`selector does not match template labels`。
- 在成功创建 Deployment 后，按 [Pod](#pod) 一节再直接创建一个 Pod，名称除了最后一段都和 Deployment 生成的 Pod 相同、且不带 `labels`；再进行之上的删除试验，就会发现 Deployment 和这个 Pod 完全没关系。

### Multiple Objects

在上一节 Deployment 的基础上我们在添加 [Service](k8s-service-tldr.md)：

```
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-xxx
spec:
  selector:
    matchLabels:
      app: nginx-xxx
  replicas: 2
  template:
    metadata:
      labels:
        app: nginx-xxx
    spec:
      containers:
      - name: nginx
        image: public.ecr.aws/nginx/nginx-unprivileged:1.29.1-alpine3.22
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-xxx
spec:
  selector:
    app: nginx-xxx
  ports:
    - protocol: TCP
      port: 8080
      targetPort: 8080
```

1. 拷贝以上内容到新建 YAML 文件，比如 `app.yaml`。
   - 注意和 [Deployment](#deployment) 一节相比，多了 `replicas: 2` 这个配置，在之下查看 Pod 时会发现有两个实例。
   - 真实应用还会有 [Ingress](k8s-ingress-tldr.md) 及 [ConfigMap](k8s-configmap-tldr.md) 等，入门文档就不展开了，在[从头创建应用 Chart](helm-chart-walkthrough.md) 里会有讨论。
1. 确认当前集群和 Namespace，然后创建完整的 Kubernetes 应用：
   ```
   oc apply -f app.yaml
   ```
1. 除 Deployment、Pod 外，查看新生成的 Service：
   ```
   oc get svc/nginx-xxx -o yaml
   ```
1. 另开一个窗口将服务转到本地：
   ```
   oc port-forward svc/nginx-xxx 8080:8080
   ```
   - 如果本地的 `8080` 端口已被占用，可以调整为 `8088:8080`或其他，注意第一个端口代表可选择的本地端口，而第二个是 Service 的、配置已固定。
   - 还可以 `oc port-forward deployment/nginx-xxx 8080:8080` 甚至 Pod，而且没有任何区别，以下会有进一步讨论。
1. 回到原窗口可以如下访问 Nginx 服务：
   ```
   curl localhost:8080
   ```
1. 确认当前集群和 Namespace，然后清理全部资源：
   ```
   oc delete -f app.yaml
   ```
   虽然可以分别删除 Deployment 及 Service，但以上方式更正规。

我们将这个试验变得更复杂，用以解释 Service 的作用：

1. 在以上多次 `curl` 后我们查看两个 Pod 的日志，发现始终只有一个 Pod 有访问记录，类似如下：
   ```
   127.0.0.1 - - [22/Aug/2025:04:08:38 +0000] "GET / HTTP/1.1" 200 615 "-" "curl/7.55.1" "-"
   ```
   为何如此？与 [port-forward](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_port-forward/) 的机制有关，它只是"Forward one or more local ports to a pod"，并不是先转发到 Service 再平均分发到多个 Pod 实例；无论 `oc port-forward` 到 Service、Deployment 还是 Pod，都只是选择某一个 Pod 的方式；因此我们需要真正的 `curl` 到所部署的 Service，继续下一步。
1. 由于暂未实现 [Ingress](k8s-ingress-tldr.md) 不能从外部访问，我们只能在集群内部进行这个试验，也就是说在同一 Namespace 部署一个 Debug Pod，然后从中去 `curl`。由于试验的 Nginx 镜像已经带有 `curl` 工具（和 VM 不同，很多镜像都不带常用工具），我们实际上不用真正新建 Debug Pod，登录到 Nginx 任意一个 Pod 就行：
   ```
   oc rsh nginx-xxx-***-***
   ```
   在容器内部执行：
   1. ```
      for i in $(seq 1 20); do curl nginx-xxx:8080 $i; done
      ```
1. 回到本地环境查看两个 Pod 的日志，均有类似以下访问记录且数量基本平分：
   ```
   192.50.6.1 - - [22/Aug/2025:04:16:52 +0000] "GET / HTTP/1.1" 200 615 "-" "curl/8.14.1" "-"
   ```

## Hello helm

**注意**以上 [Hello oc](#hello-oc) 一节起始的提醒事项。

### Helm install

1. 创建 Helm [Chart](helm-tldr.md#chart) 脚手架：
   ```
   helm create nginx-xxx
   ```
   - 所生成的 `./nginx-xxx` 目录包含 Helm Chart 的实际内容，比较复杂可以暂时看作一个黑盒，深入了解参见[从头创建应用 Chart](helm-chart-walkthrough.md)。
   - 本文试验基于 Helm v3.18.6，如果不是特别大的版本变化应该不会影响演示结果。
1. 确认当前集群和 Namespace，然后部署 Helm 应用：
   ```
   helm install nginx-xxx ./nginx-xxx --set image.repository=public.ecr.aws/nginx/nginx-unprivileged --set image.tag=1.29.1-alpine3.22 --set service.port=8080
   ```
1. 查看 Helm 部署：
   ```
   helm ls
   ```
   结果类似如下：
   ```
   NAME            NAMESPACE       REVISION        UPDATED                                 STATUS          CHART           APP VERSION
   nginx-xxx       sandbox         1               2025-08-21 18:10:03.6073005 +0800 CST   deployed        nginx-xxx-0.1.0 1.16.0
   ```
1. 查看 Helm 所创建的 Kubernetes Object：
   ```
   oc get deployment/nginx-xxx svc/nginx-xxx sa/nginx-xxx
   ```
1. 如以上 [Multiple Objects](#multiple-objects) 一节验证服务。
1. 确认当前集群和 Namespace，然后卸载 Helm 应用：
   ```
   helm uninstall nginx-xxx
   ```
   再如上查询一下 Helm 所创建 Release 及 Kubernetes Object 是否存在。

### UPGRADE FAILED

我们来模拟 Helm 部署时常见的"UPGRADE FAILED"和其他异常：

1. 正常安装：
   ```
   helm install nginx-xxx ./nginx-xxx --set image.repository=public.ecr.aws/nginx/nginx-unprivileged --set image.tag=1.29.1-alpine3.22 --set service.port=8080
   ```
   1. 稍等一会后确认容器实际状态：
      ```
      C:\>oc get pod -l app.kubernetes.io/instance=nginx-xxx
      NAME                         READY   STATUS    RESTARTS   AGE
      nginx-xxx-5c68f78f69-kbjtq   1/1     Running   0          13s
      ```
      和之上 [Deployment](#deployment) 的 `-l app=nginx-xxx` 不同，Helm 默认使用语义更规范的 Well-Known Labels [app.kubernetes.io/instance](https://kubernetes.io/docs/reference/labels-annotations-taints/#app-kubernetes-io-instance) 等。
   1. 确认 Helm 部署状态：
      ```
      C:\>helm history nginx-xxx
      REVISION        UPDATED                         STATUS          CHART           APP VERSION     DESCRIPTION
      1               Tue Sep  9 10:12:39 2025        deployed        nginx-xxx-0.1.0 1.16.0          Install complete
      ```
      `helm ls` 查看 Release 列表，`helm hist` 查看指定 Release 的变更历史，目前只有一个 `REVISION`，以下继续。
1. 故意设置错误镜像：
   ```
   helm upgrade nginx-xxx ./nginx-xxx --set image.repository=public.ecr.aws/nginx/no-this-image --set image.tag=1.29.1-alpine3.22 --set service.port=8080
   ```
   1. 稍等一会后确认容器实际状态：
      ```
      C:\>oc get pod -l app.kubernetes.io/instance=nginx-xxx
      NAME                         READY   STATUS              RESTARTS   AGE
      nginx-xxx-5c68f78f69-kbjtq   1/1     Running             0          73s
      nginx-xxx-89b6567c6-5ntpk    0/1     ImagePullBackOff    0          44s
      ```
      因为故意设置错误镜像，导致了 `ImagePullBackOff`；但由于是 [RollingUpdate](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#strategy)，只有当新部署就绪后才会移除旧 Pod（`kbjtq`），因此即使部署出错也不会中断服务。
   1. 确认 Helm 部署状态：
      ```
      C:\>helm history nginx-xxx
      REVISION        UPDATED                         STATUS          CHART           APP VERSION     DESCRIPTION
      1               Tue Sep  9 10:12:39 2025        superseded      nginx-xxx-0.1.0 1.16.0          Install complete
      2               Tue Sep  9 10:13:48 2025        deployed        nginx-xxx-0.1.0 1.16.0          Upgrade complete
      ```
      - Revision `1` 的状态是 `superseded`，即"已被取代"。
      - 虽然容器实际状态并不正常，但最新的 Revision `2` 仍是 `deployed`，也就是说 **Helm 部署成功不代表应用实际部署成功**。
1. 在设置错误镜像的同时，要求 Helm 部署命令必须等到全部 Pod 就绪才完成（`--wait`），也就是解决以上的"不代表"问题：
   ```
   helm upgrade --wait nginx-xxx ./nginx-xxx --set image.repository=public.ecr.aws/nginx/no-this-image2 --set image.tag=1.29.1-alpine3.22 --set service.port=8080
   ```
   1. 由于该命令会长时间等待，在这个过程中另开一个窗口执行：
      1. 稍等一会后确认容器实际状态：
         ```
         C:\>oc get pod -l app.kubernetes.io/instance=nginx-xxx
         NAME                         READY   STATUS             RESTARTS   AGE
         nginx-xxx-5497bb7858-g4fvm   0/1     ImagePullBackOff   0          5s
         nginx-xxx-5c68f78f69-kbjtq   1/1     Running            0          2m22s
         ```
         正常 Pod（`kbjtq`）未变，但出错的 Pod 被重新创建了，这是因为设置了不同的镜像名称，否则会保持原样。
      1. 确认 Helm 部署状态：
         ```
         C:\>helm history nginx-xxx
         REVISION        UPDATED                         STATUS          CHART           APP VERSION     DESCRIPTION
         1               Tue Sep  9 10:12:39 2025        superseded      nginx-xxx-0.1.0 1.16.0          Install complete
         2               Tue Sep  9 10:13:48 2025        deployed        nginx-xxx-0.1.0 1.16.0          Upgrade complete
         3               Tue Sep  9 10:14:57 2025        pending-upgrade nginx-xxx-0.1.0 1.16.0          Preparing upgrade
         ```
         - 由于设置了错误镜像名称，Pod 不可能就绪，因此最新的 Revision `3` 会长时间是 `pending-upgrade` 状态。
         - 对于 `pending-upgrade` 状态的 Release，`helm ls` 查询不到，需要 `helm ls -a`。
      1. 故意并发执行以上的正常安装步骤：
         ```
         C:\>helm upgrade nginx-xxx ./nginx-xxx --set image.repository=public.ecr.aws/nginx/nginx-unprivileged --set image.tag=1.29.1-alpine3.22 --set service.port=8080
         Error: UPGRADE FAILED: another operation (install/upgrade/rollback) is in progress
         ```
         - 这就是我们在真实场景常遇到的一种错误，用户在相近时间内触发了同一 Release 的多次 CD 任务；由于 Helm 本身的部署通常很快，如果不是刻意同时执行，一般都是启用 `--wait` 导致的。
         - 可以尝试 `helm upgrade --force`，但结果相同。
   1. 回到原窗口，约 5 分钟后出现以下错误：
      ```
      Error: UPGRADE FAILED: context deadline exceeded
      ```
      这是典型的超时错误，可以通过 `--timeout` 调整默认的超时设置。
   1. 再次确认 Helm 部署状态：
      ```
      C:\>helm history nginx-xxx
      REVISION        UPDATED                         STATUS          CHART           APP VERSION     DESCRIPTION
      1               Tue Sep  9 10:12:39 2025        superseded      nginx-xxx-0.1.0 1.16.0          Install complete
      2               Tue Sep  9 10:13:48 2025        deployed        nginx-xxx-0.1.0 1.16.0          Upgrade complete
      3               Tue Sep  9 10:14:57 2025        failed          nginx-xxx-0.1.0 1.16.0          Upgrade "nginx-xxx" failed: context deadline exceeded
      ```
      - Revision `3` 从之上的 `pending-upgrade` 状态变成最终的 `failed`。
      - 之上故意并发执行的 `helm upgrade` 并没有产生新的 Revision，因为是在 Helm 部署阶段出的错，而不是 Helm 部署完后的应用启动阶段。
1. 重现 `pending-upgrade` 状态，然后尝试新的应对：
   ```
   helm upgrade --wait nginx-xxx ./nginx-xxx --set image.repository=public.ecr.aws/nginx/no-this-image --set image.tag=1.29.1-alpine3.22 --set service.port=8080
   ```
   1. 另开窗口执行：
      1. 确认 Helm 部署状态：
         ```
         C:\>helm history nginx-xxx
         REVISION        UPDATED                         STATUS          CHART           APP VERSION     DESCRIPTION
         1               Tue Sep  9 10:12:39 2025        superseded      nginx-xxx-0.1.0 1.16.0          Install complete
         2               Tue Sep  9 10:13:48 2025        deployed        nginx-xxx-0.1.0 1.16.0          Upgrade complete
         3               Tue Sep  9 10:14:57 2025        failed          nginx-xxx-0.1.0 1.16.0          Upgrade "nginx-xxx" failed: context deadline exceeded
         4               Tue Sep  9 10:24:03 2025        pending-upgrade nginx-xxx-0.1.0 1.16.0          Preparing upgrade
         ```
      1. 强制取消正在执行的部署并回滚：
         ```
         C:\>helm rollback nginx-xxx 2
         Rollback was a success! Happy Helming!
         ```
         Revision `2` 是当前最新的正常状态，回滚到 `3` 技术可行但没意义。
      1. 确认 Helm 部署状态：
         ```
         C:\>helm history nginx-xxx
         REVISION        UPDATED                         STATUS          CHART           APP VERSION     DESCRIPTION
         1               Tue Sep  9 10:12:39 2025        superseded      nginx-xxx-0.1.0 1.16.0          Install complete
         2               Tue Sep  9 10:13:48 2025        superseded      nginx-xxx-0.1.0 1.16.0          Upgrade complete
         3               Tue Sep  9 10:14:57 2025        failed          nginx-xxx-0.1.0 1.16.0          Upgrade "nginx-xxx" failed: context deadline exceeded
         4               Tue Sep  9 10:24:03 2025        pending-upgrade nginx-xxx-0.1.0 1.16.0          Preparing upgrade
         5               Tue Sep  9 10:25:11 2025        deployed        nginx-xxx-0.1.0 1.16.0          Rollback to 2
         ```
         回滚并不是删除历史到只剩下 Revision `2` 和更早的，实际上还是新增 Revision `5` 但状态同 `2`。
   1. 回到原窗口，之前执行的命令并没有因为以上的回滚而中断，还是在超时后退出：
      ```
      Error: UPGRADE FAILED: context deadline exceeded
      ```
      或者也可以直接中断（`Ctrl`+`C`）退出，结果为 `context canceled`。
   1. 再次确认 Helm 部署状态：
      ```
      C:\>helm history nginx-xxx
      REVISION        UPDATED                         STATUS          CHART           APP VERSION     DESCRIPTION
      1               Tue Sep  9 10:12:39 2025        superseded      nginx-xxx-0.1.0 1.16.0          Install complete
      2               Tue Sep  9 10:13:48 2025        deployed        nginx-xxx-0.1.0 1.16.0          Upgrade complete
      3               Tue Sep  9 10:14:57 2025        failed          nginx-xxx-0.1.0 1.16.0          Upgrade "nginx-xxx" failed: context deadline exceeded
      4               Tue Sep  9 10:24:03 2025        failed          nginx-xxx-0.1.0 1.16.0          Upgrade "nginx-xxx" failed: context deadline exceeded
      5               Tue Sep  9 10:25:11 2025        deployed        nginx-xxx-0.1.0 1.16.0          Rollback to 2
      ```
      虽然最新的是 Revision `5`，但实际变更的是 `4` 的状态。

还可以进行更多试验，比如尝试 `--atomic` 参数，不再展开。

## 安全运维

这个主题不算入门内容，但很关键**需要从入门开始就养成良好习惯**，因此也有必要提及。这里的安全指的是尽可能不出纰漏的运维，比如不要误把针对测试集群的命令执行到生产上去，因为运维人员的个人电脑是可以同时连接生产、测试不同环境的；而这也不是堡垒机之类能彻底解决的，在个人电脑同时打开生产测试两个远程窗口和直连也没差别，而且即使只能连一个环境也存在操作错集群的可能、只能连一个集群也存在操作错 Namespace 的可能……

总之这个问题的**本质**是运维人员不可能只管理一个对象，而现实又很难让人在一段时间内只专心做一件事，甚至就存在同时连多个对象的场景，因此结果就是在上下文切换中很容易误操作到错误的目标对象。在应用部署阶段，可以将命令固化到 CD 环节，从而避免人工操作引发的纰漏，但在排查问题或其他场景，仍会有不少手工执行命令的情况，所以需要通过各种设计来缓解：

- 如之上提及的在每次执行变更类操作时都先用 `oc project` 确认一下当前的集群及 Namespace。
- 使用以下命令参数强制指定 Namespace，大部分命令都支持这个参数：
  ```
  oc -n sandbox delete ......
  ```
  也就是说即使当前 Namespace 是另一个也会按 `-n` 指定的 Namespace 执行，但这限制不了集群；不过由于正式 Namespace 在生产测试的命名不同（尾缀 `-p` Vs. `-t`），如果执行到错误集群，因 Namespace 不存在则不会成功变更，倒是间接解决了这个问题，但个人还是很不喜欢这种命名方式；另外也可以指定集群参数，但太冗长了不够实用。
- [*A wrapper to run oc dangerous command safely*](why-here.md) 或类似方案，实际就是在每次执行变更（危险）操作前自动 `oc project` 并提示用户确认。

但注意无论哪种方案都无法杜绝问题，比如执行了 `oc project` 但看都不看结果就顺手做下一步了；而即使彻底解决了这个问题，那也只是安全运维的一小块。总之**疏忽是人的天性、即使低级错误也很正常**，要求绝对不出纰漏，最终只会是"不干活就不出错"；我们**要做的是**引导用户养成良好习惯、形成肌肉记忆，在每次事故后追溯根因、而不是把一线操作人员当主要责任对象，容易疏漏的点能及时形成文档、固化到自动化脚本或工具……通过种种的技术或非技术设计而非泛泛的认真负责的口号，来尽量降低出问题的概率和实际影响、能快速的观测识别并告警出了问题，以及出问题后的有效恢复机制等等等等。
