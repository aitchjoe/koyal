---
title: K8s Debug
---

## 背景说明

由于容器、Kubernetes 的特性，传统环境（主要是 VM）一些常用的 Debug 手段会变得麻烦，但如[容器快速了解](container-tldr.md)所述：

> 这不代表容器 Debug 就会变得难上很多，而是说用户要学习适应新的 Debug 方式。

因此对于用户在容器、Kubernetes 环境排查问题时遇到的**新困难**，我们会在以下各节给出针对性的建议。但注意本文**关注的是在新环境中 Debug 方式的调整**，而不是某个技术问题的实际解决过程，也就是说用户要具备 Debug 的经验，比如本文不会解释为什么要用 `telnet`，而是给出容器环境没有这个工具的替代方案……总之如 [*PaaS 平台用户检查清单*](why-here.md)和[求助指南](how-to-ask-for-help.md)所述：

> 查日志、确认变更、重试、交叉验证、升级等常规排查手段在任何时候都不会过时；容器、Kubernetes 不是全新事物，里面的概念、做法很多都是业界并不陌生的最佳实践，**之前积累的经验绝大部分都不会浪费**。

另外涉及到 Kubernetes 的 Debug 场景会以我们实际使用的 [OpenShift](openshift-tldr.md) 平台为例，但本质上不会有差别。

## 常用工具

由于容器镜像缺少一些常用工具或命令，自然会给 Debug 过程带来干扰，在[公共容器镜像](container-public-image.md#tag) Tag 一节有比较几个主流操作系统镜像的默认工具，结果远少于 VM 环境；虽然我们可以在构建应用镜像时就预先将这些工具添加进去，但这又违背了容器的最佳实践，特别是在有以下变通方式的前提下。

相比遇到问题直接在容器内使用熟悉的工具排查问题，我们不否认无论如何变通总归多了一道环节、更不方便一点，但重点是在 Debug 这个非高频场景，实际带来的困难没有新手想象中那么大，业界持续提供这类"缺少工具"的镜像也能证明。当然，所谓最佳实践并不是必须遵守的教条，也有如 JDK 基镜像添加 `curl` 工具（见以上链接），但用户需要感受 / 忍受其他方面的不方便或负面影响，总之这不是一个对错选择而是**工程权衡**。

### 替代品

在 Linux 中有很多命令工具能起到同等作用，而应用方的运维人员虽然有一定经验、熟悉某些基本命令，但毕竟不是 Linux 专家，可能不知道那些"冷门"命令；因此我们首先关注在现成的应用容器内是否存在这类替代品，如有那自然是最方便的一种变通了，学习新命令就好、不需要额外的操作。以下我们列出一些开发人员可能最熟悉的命令，并给出可能的替代方案。

但如之上"公共容器镜像"链接所展现的，大部分替代工具也主要是出现在 `alpine` 镜像，`ubuntu` / `debian` 镜像很少包含；而即使包含，容器也不只是删减工具，所保留的**工具本身同样可能做了精简**，比如 `alpine` 有 `ip` 命令，但不能执行 `ip -s link`、而 VM 上的可以。

#### ping

有如下三个作用：

- 测试网络连接：现在很多服务端都屏蔽了 `ping`，不通不代表网络真的有问题，因此这个作用参见以下的 [telnet](#telnet)。
- 测试网络质量：包括延迟及丢包。尝试以下替代命令：
  ```
  traceroute gitlab.example.com
  ```
- 域名解析：如果只是为了获取 IP 可以在容器外部比如个人电脑上操作，但在容器内操作也是有意义的，因为可以验证容器的域名解析是否有问题。尝试以下替代命令：
  - ```
    nslookup gitlab.example.com
    ```
  - 类似命令还有 `host` / `dig`，但容器镜像很少带。
  - 以下 `telnet` 及部分替代命令也会顺便获取到域名 IP。

#### telnet

用于测试端口连通性。尝试以下替代命令：

- `nc`：
  ```
  # nc -zvw 5 gitlab.example.com 443
  gitlab.example.com (10.130.153.12:443) open
  # nc -zvw 5 gitlab.example.com 444
  nc: gitlab.example.com (10.130.153.12:444): Operation timed out
  ```
- `bash` + `timeout`：
  ```
  # timeout 5 bash -c "</dev/tcp/gitlab.example.com/443";echo $?
  0
  # timeout 5 bash -c "</dev/tcp/gitlab.example.com/444";echo $?
  124
  ```
  如果容器内只有 `bash` 没有 `timeout`，可以移除前面的 `timeout 5`，那样的话如果端口不通则会长时间停在命令执行状态，而不是超时 5 秒后返回 `124`。

#### curl

由于绝大部分企业应用都是 HTTP 服务，因此排除种种干扰因素、使用最基础或最原始的标准客户端 `curl` 来验证 API 调用是否正常，是很多 Debug 场景首先考虑的步骤，另外它也可以顺便测试网络连接。尝试以下替代命令：

- `wget`：
  ```
  wget -q -O - https://gitlab.example.com
  ```
  - 取代 `curl -I`：
    ```
    wget --spider -S https://gitlab.example.com
    ```
  - 也可以用 `--post-data` 或 `--post-file` 参数实现 Post 提交，并通过 `--header` 携带 Token 等等。

只有 `alpine` 镜像有 `wget`，而 `ubuntu` / `debian` 两者都无，但由于 `curl` **很实用**，因此基于 `ubuntu` 的 [eclipse-temurin](https://github.com/adoptium/containers/blob/main/17/jre/ubuntu/noble/Dockerfile#L30) 镜像（也是我们常用的 OpenJDK）添加了这个工具。

#### ps & top

`ps` 用于查看容器进程状态，`top` 能提供更实时的监控，两者互为替代，当然也只是部分替代：

- 取代 `ps`：
  ```
  top -b -n 1 -c
  ```
- 如果只是查看容器主进程命令且以上两者都没有，可以使用比较原始但肯定存在的命令：
  ```
  cat /proc/1/cmdline
  ```

#### netstat

用于查看网络连接和监听端口，但这个工具已基本废弃，由 `ss` 取代、但容器镜像很少包含。

- 取代 `netstat -r`：
  ```
  ip route
  ```
- 取代 `netstat -i`：
  ```
  cat /proc/net/dev
  ```

#### vi

在 Debug 场景，用户通常用 `vi` 修改配置并重启应用查看效果，但容器环境不是这么操作，参见以下的[配置变更](#配置变更)。其他编辑用途可以尝试如下方式：

- 如果想创建一个小型的文本文件，可以在外部编辑后拷贝到容器命令行执行：
  ```
  cat << EOF > /tmp/demo.txt
  line 1
  line two
  EOF
  ```
- 只是修改文件少量内容，比如以上生成的文件：
  ```
  sed -i 's/two/2/g' /tmp/demo.txt
  ```

### Debug Pod

如之上[替代品](#替代品)开头所说，只有 `alpine` 镜像有足够的替代工具，而对于 `ubuntu` / `debian` 镜像，麻烦仍然存在。但在部分常见的 Debug 场景，我们不是必须在应用容器内进行的，典型如验证 API 调用结果，因此我们可以另起一个包含众多工具的 Debug Pod 进行排查工作。显然 [Pod](k8s-pod-tldr.md) 针对的是 Kubernetes 环境，如果是 VM 中的容器，那 VM 应该有足够工具、直接在 VM 验证即可，思路并无二至；而在 Kubernetes 环境，普通用户是无法直接访问容器的宿主机（节点机）的。

但注意在尝试本方式前，用户最好还是在应用容器中实际执行一下，确认命令是否存在；因为应用镜像通常不是直接基于操作系统镜像、而是 JDK 等镜像制作的，而后者就有可能添加工具，以上 [curl](#curl) 一节最后有说明。不过有些替代命令比较繁琐、效果也差强人意，因此在了解本节内容后，用户可以自行决定优选哪种方式。

#### YAML

以下是创建 Debug Pod 的 YAML 模板，但注意我们**更推荐**后面的 [Job](#job) 方式：

```
apiVersion: v1
kind: Pod
metadata:
  generateName: debug-
  labels:
    app: debug-only
spec:
  containers:
  - name: tool
    image: public.ecr.aws/docker/library/busybox:1.37.0
    # image: public.ecr.aws/eks/networking-e2e-test-images/curlimages/curl:latest
    command: ['sleep', '3600']
  nodeName: null
  restartPolicy: Never
```

- `generateName`：使用 `generateName` 而非 `name` 是因为生成 Pod 的实际名称不再是固定的，而是 `debug-` 加上五位随机字符串，这样可以保证名称不冲突，方便起多个 Debug Pod 或者多人 Debug 的场景。
- `labels`：参见以下 [Job](#job) 说明。
- `image`：[BusyBox](https://busybox.net/) 是一个常见工具集，但是不包括 `curl`，我们将更多工具镜像注释（`#`）在其上，方便用户切换。
- `command`：和 Nginx 等服务镜像不同，工具镜像的默认命令不是起动一个持续运行的服务、就是 [Shell](https://github.com/docker-library/busybox/blob/dist-amd64/Dockerfile.template#L5)，因此必须调整默认命令，否则创建后就完成了。
   - 因为 Debug 就是一个临时性工作，为了避免用户事后忘记清理、继续占用资源，设为 `sleep 3600`（一个小时）就自动结束；可以酌情调大，但不要 `sleep infinity`。
- `nodeName`: 如果怀疑应用 Pod 所在的节点机有问题，可以通过该配置将 Debug Pod 明确部署到相同或不同的节点机来排查问题。
  - `null` 表示未限制、将随机部署到任意节点机，也可以删除这一行，如果需要就指定实际的节点机名称。
  - 通过 `oc describe pod/<pod-name>` 或者 `oc get pod/<pod-name> -o yaml` 查询 Pod 所在的节点机。
- `restartPolicy`: 默认配置是 `Always`，表示当命令执行完成后自动重启该 Pod；但对于临时性的 Debug 工作应该相反，否则以上的 `sleep 3600` 就没意义了。

#### Create

创建 Debug Pod 有多种方式，以我们现有的 OpenShift 平台为例，一些基础命令如登录、切换 Namespace 等参见 [OpenShift 实操入门](openshift-start.md)：

- 命令行：
  1. 拷贝以上内容到新建 YAML 文件，比如 `debug-pod.yaml`。
  1. 确认当前集群和 Namespace，然后创建 Pod：
     ```
     oc create -f debug-pod.yaml
     ```
     输出结果会提示具体的 Pod 名称，类似：
     ```
     pod/debug-45xgv created
     ```
  1. 将星号调整为上一步的结果，登录到容器内部即可开始 Debug 工作：
     ```
     oc rsh debug-****
     ```
- Web Console：
  1. 从左侧菜单选择 Administrator -> Workloads -> Pods。
  1. 从主区域左上角选择 Project / Namespace。
  1. 点击右侧 `Create Pod` 按钮。
  1. 将以上 YAML 内容拷贝到输入框（覆盖默认生成的内容）并点击 `Create`。
     - Web Console 的一个好处是提供了默认模板，因此用户熟练后直接修改其中的 `image` / `command` 等，可能比找到本文、拷贝 YAML 还方便。
  1. 创建成功后即可在该 Pod 的 Terminal 界面进行 Debug 工作。
- Template：我们基于 OpenShift Template 提供了可能对新用户更友好的方式，但从以下步骤可以看出对熟练用户不够快捷，而且也不能如上灵活的调整 `image`。
  1. 仍然是在 Web Console 操作，从左侧菜单选择 Developer -> Add。
  1. 从主区域左上角选择 Project / Namespace。
  1. 在主区域中部选择 Developer Catalog 栏的 All services。
  1. 在 All items 下的 Filter 输入框输入 `debug`。
  1. 点击其中的"Debug Tools"。
  1. 在弹出窗口点击"Instantiate Template"并按提示操作。

#### Job

以上直接创建 Pod 的方式有个问题，就是如果用户不主动删除，那么这些 Pod 将一直存在，即使是 `Completed` 状态；虽然已完成的 Pod 不占用计算资源，但并不是说毫无影响，类似 [Clean up finished jobs automatically](https://kubernetes.io/docs/concepts/workloads/controllers/job/#clean-up-finished-jobs-automatically)：

> Finished Jobs are usually no longer needed in the system. Keeping them around in the system will **put pressure on the API server**.

而且这些无用记录对用户使用也是一个干扰。但要做到 Debug Pod 自动且彻底的消失，无法依赖 Pod 本身的配置，而是 Job 的 [TTL mechanism](https://kubernetes.io/docs/concepts/workloads/controllers/job/#ttl-mechanism-for-finished-jobs)。以下是 Debug Job 的模板：

```
apiVersion: batch/v1
kind: Job
metadata:
  generateName: debug-
spec:
  ttlSecondsAfterFinished: 100
  template:
    spec:
      containers:
      - name: tool
        image: public.ecr.aws/docker/library/busybox:1.37.0
        # image: public.ecr.aws/eks/networking-e2e-test-images/curlimages/curl:latest
        command: ['sleep', '3600']
      nodeName: null
      restartPolicy: Never
```

- `ttlSecondsAfterFinished`：和以上 Pod 方式的关键差别，在指定时间后清理已完成的 Job 及相应 Pod，而 Pod 是没有这个配置的。
- 所生成的 Job 名称是 `debug-` 加上五位随机字符串，而 Pod 名称是 Job 名称再加上五位随机字符串。

虽然主要操作和直接创建 Pod 类似，但毕竟还是繁琐了一些，因此如果用户仍倾向于 Pod 方式，我们也提供一些小技巧，就是以上的 Label `app=debug-only`，这样会方便用户或者集群管理员在积累一段时间后、一次性手工批量删除：

1. 确认选择 Pod 的条件（`-l ...`）符合预期（显然不应该给正式用途的 Pod 设置如下 Label）：
   ```
   oc get pods -l app=debug-only
   ```
1. 实际删除：
   ```
   oc delete pods -l app=debug-only
   ```

## 配置变更

另一个常见的 Debug 场景是当用户遇到问题时，会尝试修改应用配置、然后重启观察效果，我们在以下试验中展现容器相比 VM 受到的限制。

### VM 流程

首先在容器中模拟常见的 VM 流程，以 [OpenShift 实操入门](openshift-start.md#pod)中的 Nginx Pod 为例：

```
apiVersion: v1
kind: Pod
metadata:
  generateName: nginx-debug-
spec:
  containers:
  - name: nginx
    image: public.ecr.aws/nginx/nginx-unprivileged:1.29.1-alpine3.22
```

创建并进入该 Pod，以下均在容器内操作：

1. 查看当前用户及用户组：
   ```
   $ echo user=$(whoami) group=$(groups $(whoami))
   user=1002770000 group=root
   ```
   在 OpenShift 平台，容器是以随机的十位数 UID 运行，但属于 `root` 组。
1. 查看当前用户是否有权限修改 Nginx 配置：
   ```
   $ ls -l /etc/nginx/conf.d/default.conf
   -rw-rw-r--    1 1002770000 root          1074 Aug 27 09:30 /etc/nginx/conf.d/default.conf
   ```
   其中第二个 `rw` 和后面的 `root` 表示属于 `root` 组的用户都可以写入。
1. 变更前验证服务（默认配置为 `8080` 端口）：
   ```
   $ curl --no-progress-meter localhost:8080|grep -E 'h1|Failed'
   <h1>Welcome to nginx!</h1>
   ```
1. 修改配置（将监听端口调整为 `8081`）：
   ```
   $ sed -i 's/8080/8081/g' /etc/nginx/conf.d/default.conf
   ```
   除了 `sed` 工具外 Nginx 镜像也带了 `vi`，可以更方便 Debug 时修改，但大多数容器都不会有后者。
1. 重载新配置：
   ```
   $ nginx -s reload
   2025/08/27 09:30:37 [notice] 105#105: signal process started
   ```
   - 应该就是 `kill -HUP 1`，参见 [Controlling nginx](https://nginx.org/en/docs/control.html)，而在容器里 Nginx 主进程的 PID 就是 `1`，可以用 `ps` 查看。
1. 确认变更生效（实际端口从 `8080` 变为 `8081`）：
   ```
   $ curl --no-progress-meter localhost:8080|grep -E 'h1|Failed'
   curl: (7) Failed to connect to localhost port 8080 after 0 ms: Could not connect to server
   $ curl --no-progress-meter localhost:8081|grep -E 'h1|Failed'
   <h1>Welcome to nginx!</h1>
   ```
1. 尝试修改更多（这次我们修改首页标题）：
   ```
   $ ls -l /usr/share/nginx/html/index.html
   -rw-r--r--    1 root     root           615 Aug 13 15:10 /usr/share/nginx/html/index.html
   $ sed -i 's/Welcome/Bienvenue/g' /usr/share/nginx/html/index.html
   sed: can't create temp file '/usr/share/nginx/html/index.htmlXXXXXX': Permission denied
   ```
   可以看出 `index.html` 文件只允许 `root` 用户写入，其他 `root` 组的用户只能读取，因此也就无法变更。

试验到此可以看出和 VM 并无不同，我们继续下一节。

### 容器流程

上一节遇到的问题在 VM 中可以进一步处理，但在容器上却很难实现：

- 当出现文件访问权限的限制时，VM 可以切换为 `root` 用户继续尝试；但是以上容器没有 `sudo` 命令，`su` / `login` 则提示"must be suid to work properly"，也就是说无法切换用户，大部分容器都如此。
  - 当然也可以在容器镜像的制作阶段将 `index.html` 等调整为 `root` 组用户可写，但如果不能提前预期，碰到问题时就变成了一个无法在现场解决的麻烦；而将所有的文件目录调成可写，显然又不符合安全实践。
- 大部分应用并不能像 Nginx 那样通过 Kill 信号来重载配置，因为这件事没想象中简单，不是只读取一下新的配置内容，在应用中生效很可能有一系列的连锁反应；因此通常是通过重启应用来加载新配置，这种做法虽然粗暴但足够简单，并不算低级，但在 VM 很简单的操作换到容器却非如此。
  - 由于应用通常就是容器的主进程（PID = 1），不能在容器内部停止，即使是强行 `kill -9 1`（[Unable to kill process with PID 1 in docker container](https://unix.stackexchange.com/questions/457649/unable-to-kill-process-with-pid-1-in-docker-container/457650#457650) 有详细讨论），而 `kill -15 1` 虽然生效但会重启容器，再进入时之前做过的变更如 `default.conf` 已还原了，而如果不能原地停止应用，那自然也谈不上我们希望通过重启达到的目的。
  - 当然也可以将容器的主进程设为守护进程、再由它启动应用，这样就可以原地重启应用；但容器很少这样处理，因为重启容器本身就很轻松，没必要将容器内弄得更复杂，仍然如之前提示的，这不是一个对错选择而是工程权衡。

因此在配置变更及生效的场景，Kubernetes 通常是如下应对：

1. 将应用配置（如 `default.conf`）创建为 [ConfigMap](k8s-configmap-tldr.md)。
1. 将 ConfigMap 挂载为 Pod 的环境变量或文件，如果是文件且已存在则会被覆盖，这种挂载不受容器内原始文件权限的限制。
   - 在容器环境（比如 VM 中的 Docker）机制类似，但不是挂载 ConfigMap 而是 VM 的本地文件目录。
1. 在排查过程中，如果需要调整配置项则修改 ConfigMap 并重启 Pod，重启可能是手工或由 ConfigMap 变更自动触发，显然这些操作都是在外部而非容器内进行的。
   - 但如上一节描述的，少量情况下也可以直接在容器内操作，但注意**只能**用在 Debug 场景。
1. 如果需要变更别的内容（如 `index.html`），则创建新 ConfigMap 或在原 CongfigMap 添加新配置项，然后重复上面的步骤。

我们会在下一节实际演示，总的来讲确实比 VM 麻烦不少，但这算不可变基础设施的一个标准操作模式。本质上这个麻烦不是从 VM 转容器 / Kubernetes、而是从传统运维转不可变基础设施所带来的，VM 同样也可以实现这种模式并由此带来相应的麻烦（看具体设计，可能会小些）；总之本节的**重点**不是介绍一个差不多轻松的替代方案、实际并没有，而是说明这些麻烦的由来，以及这明显又是一个工程权衡，如果要获得不可变基础设施带来的好处（参见 [AWS Well-Architected](https://docs.aws.amazon.com/zh_cn/wellarchitected/latest/framework/rel_tracking_change_management_immutable_infrastructure.html) 等），那么确实要忍受 Debug 时的麻烦……

### ConfigMap

通过 ConfigMap 变更配置的示例：

```
---
kind: ConfigMap
apiVersion: v1
metadata:
  name: nginx-debug-xxx
data:
  index.html: |+
    <h1>Bienvenue to nginx!</h1>
---
apiVersion: v1
kind: Pod
metadata:
  name: nginx-debug-xxx
spec:
  containers:
  - name: nginx
    image: public.ecr.aws/nginx/nginx-unprivileged:1.29.1-alpine3.22
    volumeMounts:
    - name: config
      mountPath: /usr/share/nginx/html/index.html
      subPath: index.html
  volumes:
  - name: config
    configMap:
      name: nginx-debug-xxx
      # defaultMode: 0664
```

- ConfigMap：
  - `index.html`：这个键名可以随意指定，但我们的目标就是配置这个文件；`|+` 是 YAML 语法、用于配置多行内容并保留最后的回车符，可以换成 `|` 在以下试验中对比效果。
- Pod：
  - `volumes`：
    - `name`：可以随意指定，当然有明确语义最好。
    - `configMap.name`：必须对应以上的 ConfigMap 名称。
    - `configMap.defaultMode`: 默认为 `0644`，可以在以下试验中移除注释对比效果，显然用户需要一些 Linux 的前置知识、了解这些数字的含义。
  - `volumeMounts`：
    - `name`：必须对应到 Volume 名称。
    - `mountPath`：是我们想要在容器中创建或覆盖的完整文件名称，这个完全取决于具体的容器镜像。
    - `subPath`：ConfigMap 中某一项 `data` 键，其值即以上 `mountPath` 对应文件的实际内容。

使用 `oc apply ...` 创建以上 ConfigMap 及 Pod，然后进入容器操作：

1. 确认挂载文件的内容就是 ConfigMap 中配置的：
   ```
   $ cat /usr/share/nginx/html/index.html
   <h1>Bienvenue to nginx!</h1>
   ```
1. 确认文件权限：
   ```
   $ ls -l /usr/share/nginx/html/index.html
   -rw-r--r--    1 root     1002770000        29 Aug 28 09:27 /usr/share/nginx/html/index.html
   $ touch /usr/share/nginx/html/index.html
   touch: /usr/share/nginx/html/index.html: Read-only file system
   ```
   不再是原始文件默认的 `root` 组，但对于当前用户仍是只读状态。
1. 验证实际服务：
   ```
   $ curl localhost:8080
   <h1>Bienvenue to nginx!</h1>
   ```

更多对比试验，注意修改 Volume 相关配置后继续 Apply 可能会出错、需要删除重建：

- 故意设置为 `mountPath: /usr/share/nginx/html/index2.html` 不覆盖原文件，这样可以对比两者的内容及文件权限。
- 启用 `defaultMode: 0664`，使用 `ls -l` 确认变成了登录用户组可写，但 `touch` 仍然出错，这是因为 [A ConfigMap is always mounted as readOnly](https://kubernetes.io/docs/concepts/storage/volumes/#configmap)，显然不可变基础设施并不鼓励从内部修改配置，而其他种类的存储虽然可行但并非配置用途。

## 常见 Debug 场景

### Ingress

#### 常见 Ingress 问题

用户部署应用并通过 [Ingress](k8s-ingress-tldr.md) 将服务暴露到集群外后，新手经常会遇到的一个问题是访问域名时提示：

> The application is currently not serving requests at this endpoint. It may not have been started or is still starting.

而现场实际也给出了一些"Possible reasons you are seeing this page"，确认应用是否启动、访问域名是否就是 Ingress 配置的域名等等，这些常规手段无论 VM 还是容器毫无二致，因此以下我们主要讨论 Kubernetes 场景的特定问题，首先考虑两个因素：

- 查看应用日志确认是否成功启动是基本操作，但 Kubernetes 有[探针](k8s-probe-tldr.md)机制、只有满足探针条件后才会对外提供服务，因此如应用已正常启动，但长时间后仍不能对外提供服务，那有很大概率是由于探针的错误配置引起。
  - 执行 `oc get pod/<pod-name>` 可以查看 Pod 的 `READY` 情况，如类似 `1/1` 则表示就绪，前者表示就绪的容器数、后者表示 Pod 里的容器总数，两者相同才代表 Pod 完全就绪。
  - 如果不熟悉探针配置，可以先移除 Pod 里的这部分配置确认是否由其引起。
- OpenShift 集群有提供泛域名（如 [*.apps.paas-wh-01-uat.example.com*](why-here.md)）直接使用，不需要用户另行申请；而现在有多个集群，虽然概率不大，但我们实际遇到过应用部署在某个集群、但配置了另一集群的泛域名，因此也先确认一下这个因素。

以上是基于 Kubernetes 知识所做的判断，对 Kubernetes 了解越深，自然越能更快定位问题，而下面我们按照排除法的思路给出更具体的示例。

#### 排除法

我们按照**拆分问题、缩小排查范围**的常规思路来演示问题定位，当然这仍不可避免的涉及到对 Kubernetes 的了解，[Ingress](k8s-ingress-tldr.md) 有说明：

> 从集群外访问 Kubernetes 应用的完整调用链路为："**客户端 - 集群外 LB - Ingress - Service - Pod**"。但注意这只是一个抽象描述，实际上从 OpenShift 的 HAProxy 配置可以确认，网络流量是从 HAProxy 直接分发到了 Pod。

显然我们应该逐个环节进行验证，看看问题究竟出在何处。仍以之上的 Nginx 为例，但注意为了防止和其他试验的用户产生命名冲突，可以将以下的 `nginx-debug-xxx` 都替换为其他唯一字符串，包括配置和所执行的命令：

```
---
apiVersion: v1
kind: Pod
metadata:
  name: nginx-debug-xxx-1
  labels:
    app: nginx-debug-xxx
spec:
  containers:
  - name: nginx
    image: public.ecr.aws/nginx/nginx-unprivileged:1.29.1-alpine3.22
    readinessProbe:
      httpGet:
        path: /
        port: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-debug-xxx
spec:
  selector:
    app: nginx-debug-xxx
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8080
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nginx-debug-xxx
spec:
  rules:
  - host: nginx-debug-xxx.apps.paas-wh-01-uat.example.com
    http:
      paths:
        - path: /
          pathType: Prefix
          backend:
            service:
              name: nginx-debug-xxx
              port:
                number: 80
```

- Pod：真实部署时会使用 [Deployment](k8s-deployment-tldr.md) 而不是直接创建 Pod。
  - `name`：为什么不设为 Service、Ingress 同名而是添加尾缀 `-1`？是因为在以下试验时会混淆 `curl` 的目标；而且真实部署使用同名 Deployment，那 Pod 名称本来也会加上五位随机字符串。
    - 即使 Pod 和 Service 名称相同其实也是可以区分的，这种情况下 `nginx-debug-xxx` 是 Pod，而 `nginx-debug-xxx.<namespace>.svc` 就是 Service，可以通过 `ping` 这两个名称来验证、并和 `oc get pod/nginx-debug-xxx-1 svc/nginx-debug-xxx -o wide` 的实际 IP 比对，但我们在以下试验中就不将演示弄复杂了。
  - `readinessProbe`：演示就绪探针对服务可用的影响，会在下一节调整对比。
- Service：
  - `selector`：Service 是通过 Lable 而不是名称和 Pod 关联的。
  - `targetPort`：为实际 Pod 中应用的端口号，由于不应该以 `root` 运行，只能是 `1024` 及以上的端口。
  - `port`：Kubernetes Service 的端口号，通常设成 `80` 或者和 Pod 端口相同。
- Ingress：
  - `host`：其中 `paas-wh-01-uat` 是试验所在的集群，需调整为实际集群。
    - 如 Pod、Service、Ingress 等 Object 都是局限在 Namespace 内的，也就是说不同 Namespace 的同名并不会引起冲突；但基于集群泛域名的 `host` 必须在同一集群保持唯一，因此开头才提示最好统一替换再做试验，当然如果对这些配置都很了解，只替换这个 `host` 也行。
  - `service.name`：对比 Service 和 Pod 是通过 Label 关联，Ingress 和 Service 则是直接通过名称关联的。
  - `port`：当然指向的是 Service 而不是 Pod 的端口。

部署以上 Pod、Service 及 Ingress 后，我们就可以逐层排查了；由于 Nginx 镜像本身有 `curl` 工具，我们直接使用或者如上创建 Debug Pod：

1. 先从最靠近应用的 **Pod** 开始：
   ```
   curl nginx-debug-xxx-1:8080
   ```
   - 由于 Pod 及下面的 Service 等都不会暴露到集群外，因此很多排查工作需要依赖集群内的 Debug Pod。
   - 这一步失败则可能是应用启动问题或者应用的监听端口就不是 `8080`。
   - 注意在应用容器内测试时，不要依赖 `curl localhost:8080` 的结果，因为这不代表能从外部访问。
1. Pod 没问题后验证 **Service**（以下命令可省略 `80` 默认端口）
   ```
   curl nginx-debug-xxx:80
   ```
   如果失败，则如下确认：
   ```
   ocw get ep/nginx-debug-xxx pod/nginx-debug-xxx-1 -o wide
   ```
   实际结果类似：
   ```
   NAME                        ENDPOINTS            AGE
   endpoints/nginx-debug-xxx   192.50.14.147:8080   10m
   NAME                    READY   STATUS    RESTARTS   AGE   IP              NODE                                            NOMINATED NODE   READINESS GATES
   pod/nginx-debug-xxx-1   1/1     Running   0          10m   192.50.14.147   xxwh1a1r10n05.paas-wh-01-uat.example.com   <none>           <none>
   ```
   每个 Service 都会有一个同名的 Endpoint，确认其 `ENDPOINTS` 结果中的 IP（`192.50.14.147`） 是否对应 Pod 的 IP、端口（`8080`）是否上一步成功验证 Pod 的端口；如果没问题，那很可能是探针引起，我们在下一节做更复杂的试验。
1. Service 没问题后验证 **Ingress**，实际验证的是 OpenShift 的路由服务；先从[*集群清单*](why-here.md)查询当前集群的"Router 节点机"，然后调整以下 `Host` 值和 Ingress 配置中的 `host` 相同、IP 为任一 Router 节点机 IP：
   ```
   curl -H "Host:nginx-debug-xxx.apps.paas-wh-01-uat.example.com" 10.154.244.94
   ```
   - 不能直接 `curl nginx-debug-xxx.apps....`，那样就走到 LB 上了，失去了验证 OpenShift 路由的意义。
   - 如果这一步出错，用户能做的是确认 Ingress 配置是否正确，比如 `service` 的名称端口等，如果没问题则联系集群管理员处理。
   - 即使将 Ingress 配置故意设成另一个集群的域名，这一步反而不会出错，因为 Ingress 支持任意域名，实际上集群泛域名仅用于非正式场景。
   - 其实也可以将以上 IP 调整为 `oc get ingress/nginx-debug-xxx` 所获取的 `ADDRESS` 域名，类似 `router-default.apps.paas-wh-01-uat.example.com`，但这实际上还是多了一个路由 LB 的环节，作为进阶内容此处不展开。
1. 如果 Ingress 没问题则最后聚焦到 LB 环节：
   1. 用户可以做的是验证 `host` 配置的域名是否确实指向了应用所部署集群的 LB，如果使用的是集群公共 LB，则通过[*集群清单*](why-here.md)查询对应 IP，如果是用户自行申请，则需要和 LB 提供方确认。
      - 这个操作也同时检查了配错泛域名的可能。
   1. 联系 LB 管理员排查 LB 端的问题。

#### 探针

我们将 `readinessProbe` 的 `port` 故意调成 `8081`，再重复以上试验：

1. 排查各个环节：
   ```
   $ curl -s nginx-debug-xxx-1:8080|grep h1
   <h1>Welcome to nginx!</h1>
   $ curl -m 5 nginx-debug-xxx:80
   curl: (28) Connection timed out after 5002 milliseconds
   $ curl -s -H "Host:nginx-debug-xxx.apps.paas-wh-01-uat.example.com" 10.154.244.94|grep '<h1>'
         <h1>Application is not available</h1>
   ```
1. 确认 Pod 及 Service 状态：
   ```
   C:\>ocw get pod/nginx-debug-xxx-1 ep/nginx-debug-xxx
   NAME                    READY   STATUS    RESTARTS   AGE
   pod/nginx-debug-xxx-1   0/1     Running   0          6m10s
   NAME                        ENDPOINTS   AGE
   endpoints/nginx-debug-xxx               6m10s
   ```
   - 由于故意设错就绪探针，Pod 的 `READY` 始终是 `0/1`，所以 Service 的 `ENDPOINTS` 并没有一个可用 Pod。
   - 而实际上 Pod 已经 `Running`，所以直接访问 Pod 没问题。

这样就很直观的展现了就绪探针对服务可用的影响。另外还可以设置存活探针（`livenessProbe`）对比看效果，不再展开，详见[探针快速了解](k8s-probe-tldr.md)的讨论。

#### 会话保持

这也是我们实际遇到过的问题：

> 在压测时发现应用流量只进入到其中一个 Pod

而且调用链路更长，"业务 A - 业务 B - API Gateway - OpenShift 平台"，各环节内部可能也有多个组件的调用链路；但无论多复杂，本质上还是逐个排除、从源头（Pod）往外推……因此这节只说明一下排查会话保持问题和以上试验的区别：

- 重现会话保持的问题不能简单调用一次，需要循环脚本：
  ```
  while true; do curl -sI nginx-debug-xxx-1:8080|grep HTTP; done
  ```
  当然实际命令没这么简单，还要带上 Token 等，这应该由服务方给出 `curl` 的完整示例。
- 真实案例中是因为客户端复用了 Cookie，这也正说明了排查问题时使用**第三方中立工具**的好处；所有设置都直观的反应在 `curl` 命令参数上，且是标准的 HTTP 调用，因此出问题则是服务端因素、反之则是客户端因素。
- 当问题不是简单的通或不通、而是在一定流量下才反应出来，那么再一行行的检查 Access Log 就太原始了，需要通过可观测性平台直观的查看流量分布等。

实际案例参见 [*Session Stick Debug*](why-here.md)。

### Dump

出现 OOM 时需要了解现场的内存使用状况，容器应用和 VM 应用在这方面的 Profiling 分析并无二致。但在 Kubernetes 环境，普通业务应用通常是无状态的也就是说没有本地持久存储，当出现大问题导致 Pod 被强制销毁重建，那 Dump 文件也就丢失了；至于运行在 VM 中的容器应用可以将 Dump 目录挂载到宿主机，从这一点说 VM 应用和 VM 容器应用并无差别。因此以下主要关注 Kubernetes 环境的不同场景：

- Pod 仍然正常：比如没严重到 OOM 失去响应、但还是需要分析性能问题，因此主动或被动的触发了 Dump；这种情况下用户要熟悉的是如何将 Dump 文件拷出来，一般也不会在现场分析，而具体分析也不是本文关注的事、由各技术栈的开发人员负责。拷贝命令类似：
  ```
  C:\>oc cp nginx-debug-xxx-1:/etc/nginx/nginx.conf ./nginx.conf
  tar: removing leading '/' from member names
  ```
- 因 OOM 等原因导致容器重启、但 Pod 本身未销毁：可以使用非 `Memory` 类型的 [emptyDir](https://kubernetes.io/docs/concepts/storage/volumes/#emptydir) Volume 来保留故障现场的 Dump 文件，这应该是最常见的一种场景。大致的 Pod 配置类似：
  ```
  ...
      volumeMounts:
      - name: dump
        mountPath: /dump
    volumes:
    - name: dump
      emptyDir: {}
  ```
  当然还要指定应用的 Dump 目录，这个不同技术栈的方式肯定有差别，Java 类似添加命令参数 `-XX:HeapDumpPath=/dump/hdump.hprof`，完整示例参见 [*oom-demo3.yaml*](why-here.md)。
- 出问题的 Pod 已彻底销毁：无法找回所生成的 Dump 文件，通常 OOM 只会导致容器重启、但不会销毁 Pod，也就是说这种情况不算常见。
- 另外注意 OOM 会分 JVM OOM 及容器 OOM 两种状况，如果故意将 JVM 内存配置为远超过容器的资源限制，则会引起容器本身的 OOM；而这种情况下不是能不能获取到 Dump 文件的问题，而是说容器的 `OOMKilled` 是不会触发应用如 Java 的 Dump，但这种情况本来就没必要分析 Java Dump。

我们专门创建了 [*java-oom-dump-demo*](why-here.md) 项目，演示从故意触发 OOM 到获取 Dump 文件的整个过程。总的来说这仍只是比照传统 VM 手工做法的解决方式，实际上更应做的是利用如 SkyWalking 之类的 [**Continuous profiling**](https://grafana.com/docs/pyroscope/latest/introduction/continuous-profiling/)，但这不再是应用本身或 Kubernetes 平台的调整优化，**必须有可观察性平台提供支持**。
