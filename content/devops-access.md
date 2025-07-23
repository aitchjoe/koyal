---
title: DevOps 访问权限解析
---

## 问题

作为云原生应用的 GitLab [CI/CD](cicd-tldr.md) 完整示例，[*gitlab-helm-demo*](why-here.md) 演示了：

1. 由于该代码库包含 `.gitlab-ci.yml` 文件，按约定符合规则的代码提交将触发 GitLab CI/CD 流水线。
1. 根据 `.gitlab-ci.yml` 的配置，流水线中的 CI Job：
   1. **克隆** GitLab 代码库中的源码到 CI Job 所运行的环境（VM 或容器）。
   1. 编译源码并构建容器镜像。
   1. 将容器镜像**发布**到 GitLab Registry。
1. 同样根据 `.gitlab-ci.yml` 的配置，流水线后续的 CD Job：
   1. 使用 [Helm](helm-tldr.md) 方式将应用部署到 [OpenShift](openshift-tldr.md) 也就是 [Kubernetes](k8s-tldr.md)。
      - Helm 本质上也是将 [Deployment](k8s-deployment-tldr.md) 等 Kubernetes Object **提交**（`kubectl apply`）到 OpenShift，因此相比传统部署方式直接将程序包安装到目标环境并启动，OpenShift 多了以下环节，该链接有详细说明。
   1. OpenShift 启动 Deployment 配置的容器应用，并检查所运行节点机是否存在指定的容器镜像。
   1. 如果容器镜像不存在，则 OpenShift 根据 Deployment 所配置的权限从 GitLab Registry 将该镜像**拉取**到本地。
   1. 容器应用最终成功运行。

在以上描述中，"克隆"、"发布"、"提交"、"拉取"操作明显就涉及了 DevOps 各组件之间的**读写访问**，按顺序是读、写、写、读，其中写权限控制是标配，而企业环境中基本也会设置读权限；因此要想让 DevOps 整个流程顺利运转，**访问权限的配置就是 DevOps 准备工作的重要一环**，其中包括：

- 由于 GitLab 全家桶带来的便利，以下 GitLab 内部各组件的访问不需额外配置权限。注意并不是说这部分没有权限控制，**仅仅是不需手工配置**、由 GitLab 自动完成而已。
  - GitLab CI Job 克隆 GitLab 代码库源码。
  - GitLab CI Job 将容器镜像发布到 GitLab Registry。
    - 注意虽然不需要在准备工作中配置，实际执行的 CI 脚本（`.gitlab-ci.yml`）还是需要引用 GitLab 默认配置的 Token，也就是说并非完全透明。
- 需要配置 GitLab CD Job 提交 Object 到 OpenShift 的权限：
  1. 在 OpenShift 相应 Namespace 下创建专门给 GitLab CD 使用的 Service Account（SA），并获得该 Account 的 Token。
  1. 在 GitLab 相应项目的 CI/CD Settings 中创建上一步 Token 的变量。
  1. 在 GitLab CD 脚本（`.gitlab-ci.yml`）中引用该变量进行部署。
- 需要配置 OpenShift 拉取 GitLab Registry 中容器镜像的权限：
  1. 在 GitLab 相应项目的 Repository Settings 中创建 `Allows read-only access to registry images` 的 Deploy Token，获得相应的 `username` / `token`。
  1. 在 OpenShift 相应 Namespace：
     1. 创建对应之上 Token 的 Image Pull Secret。
     1. 在 Deployment 直接引用该 Secret、或者将 Secret 关联到 Deployment 的 SA。**注意**这个 SA 和以上为 CD 创建的 SA 是不同用途，因此不应该使用同一个，虽然技术上允许。

从以上描述可以看出对新手已经**非常复杂**了，因此虽然示例的 README 提供了操作说明，但不出问题还好，一旦出现波折对于如何排查就可能很茫然了，实际也有多人问起。再考虑到 GitLab 全家桶非常强调[**自主管理、自助服务**](devops-talk.md)，无论是配置准备、还是简单问题排查都是二级管理员的工作，因此有必要将这方面内容解析得更清楚。

另外**注意**由于 DevOps 团队的调整，现在已使用 Jenkins 取代 GitLab CI/CD、JFrog 取代 GitLab Registry 了，但不影响对本质的说明。

## 简化问题

以上问题显得这么复杂是因为：

- 整个 DevOps 流程涉及到了多个组件，而不是简单的 A 访问 B。
- 虽然涉及多个组件，但是具体到产品只有 GitLab 和 OpenShift。因此一会儿要配 GitLab 访问 OpenShift、一会儿又反过来配 OpenShift 访问 GitLab，对于没理解清楚的人只会更加困扰。
  - 当然如上将 GitLab 全家桶拆为 GitLab 代码库、Jenkins、JFrog，从用户的视角更容易理解，因此以下讨论中也要留意区分提及的 GitLab 是存储源码的 GitLab 代码库、还是执行流水线的 GitLab CI/CD 组件、或者是存储容器镜像的 GitLab Registry。
  - 但在理解之后，使用全家桶的便利性要大很多，从上一节准备工作的步骤描述就可以看出差别，如果变成 GitLab 和 Jenkins、JFrog 的访问，那自然要有类似 GitLab 和 OpenShift 交互的额外准备工作。
- Authn 的核心就是验证用户 ID 加密码，GitLab、OpenShift 包括 Jenkins、JFrog 本质上都不外如是，但没想清楚以下问题之前同样会觉得困扰：
  - 组件间的访问不应使用普通用户（真人），而应该是机器人用户（bot）。
  - 但是不同产品的机器人用户不一定叫这个名称，甚至不会提及机器人用户这个概念：
    - 比如 OpenShift 的 Service Acconut（SA）。
    - 比如 GitLab 在创建 Deploy token 时会生成 `Deploy token username` 和 `token`，显然"Deploy token user"就是 bot，当然也排除在 GitLab 正常的用户管理之外。
  - 不同产品对用户 ID 的命名也不一致，很多时候 `username` 就是不变的 ID、而不是可变的 Name。
  - 而密码除了比较熟悉的 password、token 名称外，GitLab 的 [Deploy token](https://docs.gitlab.com/13.9/ee/user/project/deploy_tokens/)、[Deploy key](https://docs.gitlab.com/13.9/ee/user/project/deploy_keys/index.html)、[Personal access token](https://docs.gitlab.com/13.9/ee/user/profile/personal_access_tokens.html)、[Project access token](https://docs.gitlab.com/13.9/ee/user/project/settings/project_access_tokens.html) 的不同命名虽然是为了区分使用场景，但对没想清楚的人只会更加干扰。
  - 按专业做法我们会根据不同用途创建多个 bot 用以隔离权限，类似以上的多种 Token，还比如以上 OpenShift 用于 CD 和应用运行的 SA；因此拿到多个 Token 后，如果还是不理解、将某个 bot 误用在另一个场景，自然会产生 Authn 或 Authz 错误。
- 在服务提供方（被访问方）创建 bot 用户后，由于 CI/CD 脚本是保存在源码的，因此访问方肯定不应该在脚本中直接使用密码，而间接的做法显然也会给一些用户造成困扰：
  1. 访问方通常先将密码等敏感信息保存在专门设计的安全地方，这也是**准备工作**的一环，比如 GitLab CI/CD Settings 的 Variables，以及 OpenShift 的 Image Pull Secret。
  1. 在实际访问时引用这些密码的方式也不同，比如直接在 GitLab CI/CD 脚本中引用密码变量，或者仅仅将 Image Pull Secret 关联到 Deployment 即可。

所以我们首先简化问题，只看 A（发起访问方）访问 B（被访问方）在权限方面**本质上需要做什么**：

1. 在 B 端：
   1. 创建供 A 访问的 bot 用户 ID 及密码，**不管名称叫做什么**，甚至没有用户这个实体，甚至没有用户 ID（有些实现是 Token 直接对应到下一步的权限上）。
   1. 给 bot 赋予相应的权限。这一步可能包含在上一步，比如在创建的同时有权限选项、或者使用新建用户的默认权限即可。虽然这一步可能是隐含的，但实质是权限控制中**最重要**的一环。
1. 在 A 端：
   1. 在管理配置中保存 B 提供的 bot 用户 ID 及密码，**不管是保存在哪儿**。
   1. 在实际访问中使用已保存的 bot 用户 ID 及密码，**不管是显式还是隐含**。

基于以上理解再来拆分上节的问题：

1. 问题描述中强调的每个动词，就是一种类型的 A 访问 B。
1. 对于每一类的访问定位 A 和 B 的真实身份。
   - 不要被产品名称干扰，而是按照它的实际职能或角色；自然就可以清晰的区分存储源码的 GitLab 代码库、执行流水线的 GitLab CI/CD 组件或 Jenkins 产品、存储容器镜像的 GitLab Registry 或 JFrog 等等。
1. 再将实际产品文档中的具体步骤映射到以上所抽象的本质内容。

至此就相对容易理解整个复杂场景了，出现问题后即使不能马上解决、至少也能定位到问题出在哪个部分。

## 实例解析

按之上简化问题中所归纳的核心点，我们对比映射 [*gitlab-helm-demo*](why-here.md) 的部署环节：


| 本质步骤                     | GitLab CD (A) 提交 Deployment 到 OpenShift (B) | OpenShift (A) 从 GitLab Registry (B) 拉取镜像 |
| ------                       | ------                                         | ------                                        |
| 1. B 端创建 bot 并获取 token | `oc create sa gitlab-bot`<sup>1</sup>          | 在 Repository Settings 新建 Deploy token 并选中 `read_registry`<sup>5</sup>
| 2. B 端给 bot 授权           | `oc policy add-role-to-user edit -z gitlab-bot`<sup>2</sup> | 已在上一步完成（`read_registry`）<sup>6</sup>
| 3. A 端配置保存 bot 信息     | 在 CI/CD Settings 中添加 `OPENSHIFT_BOT_TOKEN` 变量及相应值<sup>3</sup>| 创建 Image Pull Secret<sup>7</sup>
| 4. A 端使用所保存的 bot 信息 | 在 `.gitlab-ci.yml` 的 CD 脚本中引用以上变量<sup>4</sup> | 指定 Deployment 中的配置项 `imagePullSecrets` 为以上 Secret<sup>8</sup>

注释：

1. SA `gitlab-bot` 名称可以任意调整，但要尽量直观，表明给谁用、用做什么，可能 `gitlab-cd-bot` 更好，另外注意在以下步骤保持一致。
   - 以上表格中省略了获取 SA Token 的步骤，每个 SA 都有一个相同前缀的 Secret，通过 `oc describe secret/gitlab-bot-token-****` 即可获取 Token。
2. 新建 SA 的默认权限不够 CD 部署使用，所以需要额外授权。
3. 变量 `OPENSHIFT_BOT_TOKEN` 名称可以任意调整，但要尽量直观，且以下步骤保持一致。
   - 所使用的 bot 信息当然包括 ID 和密码两者，但 ID 不是敏感信息我们直接在下一步 Hard Coding 了。
4. GitLab CI/CD Job 在运行时会将 Settings 中的变量都设为环境变量，而 helm 或 oc 命令都有 `--token` 参数选项，因此可以通过引用环境变量来设置密码；而 ID 不是敏感信息，就直接保存在源码的 `.kubeconfig` 文件中。
5. 因为拉取是读操作，有些项目的 GitLab Registry 会设为公开访问，那样就无需配置、包括以下步骤。
6. 这个 Deploy token 对应权限当然是只能读同一项目的 Registry、而不是全部的 GitLab Registry。
7. 在 OpenShift 3 叫 Image Secret，显然到 OpenShift 4 名称更直观了。
8. 具体配置参见官方文档 [Pull an Image from a Private Registry](https://v1-32.docs.kubernetes.io/docs/tasks/configure-pod-container/pull-image-private-registry/#create-a-pod-that-uses-your-secret)，另外还可以 [Add image pull secret to service account](https://v1-32.docs.kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/#add-image-pull-secret-to-service-account)。
   - 前者是直接引用 imagePullSecrets，后者是通过 SA 间接引用。
   - 未找到官方建议但似乎更倾向于使用 [SA](https://kubernetes.io/blog/2025/05/07/kubernetes-v1-33-wi-for-image-pulls/) 方式。

## 基础

虽然了解上述本质后可以加快理解，但是无论如何终归需要了解 GitLab、Kubernetes / OpenShift 的术语和基本概念，比如：

- 在 OpenShift 从 GitLab Registry 拉取镜像的准备工作中，为什么配置的是 Deploy tokens 而不是 Deploy keys？这两个名称并没有标准定义或者说业界统一的含义，因此如何选择只能看具体产品（GitLab）它自己的设计思路。
  - 官方文档 [GitLab token overview](https://docs.gitlab.com/security/tokens/) 介绍了两者区别，前置"pull packages and container registry images of a project without a user and a password"，而后者"cannot be used with the GitLab API or the registry"。
- Deployment 或 Pod 配置了 imagePullSecrets 就可以拉取受保护的容器镜像，如果说这个还容易理解，那么为什么给 SA 添加了 imagePullSecrets 也可以呢？这自然要理解 Kubernetes 的运作机制。
  - 即使能理解是间接引用，那起码还要了解 Deployment 未配置 SA 时的默认值是什么（`default`）。
  - 补课：[Managing Service Accounts](https://v1-32.docs.kubernetes.io/docs/reference/access-authn-authz/service-accounts-admin/) 以及 [Configure Service Accounts for Pods](https://v1-32.docs.kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/)。
- 当然也要具备一定的 Debug 基础。在 [*gitlab-helm-demo*](why-here.md) 的操作文档里实际已给出了部分 Debug 方法，比如：
   ```
   helm ls --kubeconfig kubeconfig --kube-context=uat --kube-token=****
   ```
   同样是从 DevOps 的复杂交互环节抽身出来，只验证 A 到 B 有没有问题；也不需要重建真实的 Pod 去验证，直接手工命令行看使用的 Token 是否成功……这不外乎就是[**拆分问题、交叉验证**](how-to-ask-for-help.md#常规排查)，但能马上反应过来、甚至在不知道以上命令参数的情况下就去摸索尝试，这种用户真的不多。
