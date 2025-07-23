---
title: 公共容器镜像
---

## TLDR

- 第三方公共容器镜像的使用场景包括作为业务应用镜像的基镜像、以及直接部署的产品镜像。
- **挑选一个可靠适用的第三方公共容器镜像比想象中复杂。**
- 尽量先从权威来源包括官方文档、开源网站确认容器镜像名称。
- 容器镜像仓库主要使用 AWS 或 GitHub 的服务，没有十分把握之前，只使用其中的可信镜像。
  - 用户直接查询仓库的一个主要目的是确认指定镜像的最新可用版本。
- 由于限流和墙的因素，我们很难使用最大的 Docker 仓库，不过 AWS 提供了虽然数量占比不高、但足够覆盖主流软件的拷贝或替代品。
  - 早期使用 Docker 仓库的用户如有可能请尽快转移到 AWS 仓库。
- 和其他种类的很多制品不同，容器镜像 Tag 除了包含容器应用版本之外还包含操作系统的种类及版本。
- 只使用容器镜像的固定 Tag，保证其中容器应用的精确版本、以及所运行的操作系统及版本。
  - 类似 `17.0.15_6-jre-noble`，前面数字为 Java 版本、`noble` 代表 Ubuntu 24.04。
  - 最新版本的 Tag 通过所使用仓库的 Web 界面查询确认。
  - 不要使用 `latest`、`21` 等动态 Tag，不要被很多产品的入门文档误导。
  - 因此用户要有意识持续跟进、手工调整升级 Tag。
- 只使用精简版本，Tag 中的关键词类似 alpine / slim 等或者是 Ubuntu 镜像。
  - 至于使用主流操作系统如 Ubuntu、Debian 等的精简版还是原生精简的 Alpine Linux，这个没有倾向、用户自行决定但尽量保持一致。
- `openjdk`、`eclipse-temurin`、`bellsoft/liberica-openjre-debian` 都是不同厂商基于 OpenJDK 开源项目制作的二进制发行版及容器镜像，请熟悉这些名字。
  - 容器镜像 `openjdk` 已废弃（不是 OpenJDK 项目被废弃），请迁移到后两个镜像。
  - 我们 PaaS 平台用户使用最多的可能是 `docker.io/openjdk` 的 11 / 17 版本（包括对应在 AWS 仓库的拷贝），已经缺少了太多安全补丁，请尽快迁移并升级到至少相同大版本的最新版本。
  - 可能有用户会基于 JDK 而非 JRE 基镜像制作应用镜像，这是没必要的浪费，请尽快调整。
- 请尽快改正早期基于 Nginx 镜像并 `chmod 777` 的做法，直接使用 `nginx-unprivileged` 镜像。

## 背景说明

请大家先通过 [Container Run TLDR](container-run-tldr.md) 了解容器镜像的基础知识，然后才是本文介绍的**如何择选一个可靠适用的第三方公共容器镜像**。就像开发一个业务应用不会从零开始、必然要使用第三方的开发框架或库，在容器领域，也会分如下情况使用第三方镜像：

- 开发团队构建自己应用的容器镜像，这种情况同样不会从零开始，比如 Java 应用通常会基于 [OpenJDK](https://gallery.ecr.aws/search?searchTerm=openjdk&verified=verified) 镜像再添加自己的 Jar / Class 等文件，这里的 OpenJDK 就是所谓的**基镜像**，这些基镜像基本都是开源项目，主要发布到各大公共容器镜像仓库。
  - 可能有企业用户定制一些内部使用的公共镜像，但如无必要、勿增实体，因为这些定制镜像也只会基于外部公共镜像，而主流镜像的升级、补丁是相对频繁的，那么定制镜像跟还是不跟、及时且持续跟进的成本又如何？
- 直接部署厂商提供的**产品容器镜像**，在目前开源当道的情况下，这类镜像通常也发布在各大公共容器镜像仓库，有少量是在厂商自己的仓库但基本也不会设访问限制，总之用户都能公开获取。

当然也有第三方私有镜像需要登录仓库后获取，如 [registry.redhat.io](https://access.redhat.com/articles/RegistryAuthentication)，但要么是我们不会使用的商用产品，要么已经是客户了，而且这都是少数情况，此处不展开。

## 择选

确定使用什么容器镜像有以下多个途径，但不管哪个途径，首先是用户要带有一定的目标或者具备相应的背景知识。比如要做 Java 应用的容器化改造，那自然要先知道 OpenJDK 的概念、或者使用了哪个 Java 开发框架，这些在细分主题里有进一步说明，本节主要讨论以下的共性步骤。

### 可信镜像

1. 第一步从**官方文档**开始，这是**最权威**的信息来源：
   - 如果是应用，那么所使用的开发框架如 [Spring Boot](https://docs.spring.io/spring-boot/3.4/reference/packaging/container-images/dockerfiles.html) 或 [Vue](https://cli.vuejs.org/guide/deployment.html#docker-nginx) 通常会给出基镜像建议。
   - 如果是产品，那么在安装部署文档里通常会包含容器或 K8s 相关的章节，比如 [Run Grafana Docker image](https://grafana.com/docs/grafana/latest/setup-grafana/installation/docker/)、[Deploy Grafana on Kubernetes](https://grafana.com/docs/grafana/latest/setup-grafana/installation/kubernetes/)  等，自然也就能确定所使用的产品镜像。
   - 但并不是所有软件都必然在官方文档里提及容器镜像，比如 [Nginx](https://nginx.org/en/docs/install.html) 文档（2025-06），因此需要从以下步骤确认。
1. 开源软件的**源码网站**，这当然也是**官方来源**：
   - GitHub 等不会只保存源码，也提供了 Packages 存储，能够直接查到如 [Nginx](https://github.com/orgs/nginx/packages?ecosystem=container) 的官方容器镜像；但并不是说 GitHub 项目就必然将镜像发布在同一处，很多会发布到 Docker、AWS 或者同时多处。
   - 尝试查询是否有专门的容器化项目，如我们在 GitHub 的 NGINX org 用 [docker](https://github.com/nginx?q=docker-nginx) 关键词过滤，就可以发现有两个 Repositories，其中就给出了明确的 [pre-built images](https://github.com/nginx/docker-nginx-unprivileged/blob/1.28.0/README.md#image-registries) 提示。
     - 比以上 Packages 方式好的是文档会列出多个（如果有）仓库的官方镜像，我们能更灵活的挑选。
     - 但这种方式没有规律可循，别的软件不一定是这个关键词、也不一定是一个独立项目，所以也只是给出一个尝试的思路。
1. 在各大[公共容器镜像仓库](container-run-tldr.md#public-registry)直接用技术栈、产品关键词如 `OpenJDK`、`Nginx` 等搜索容器镜像。
   - 不经过以上步骤、直接搜仓库的方式只适合已经具备上述背景知识的有经验用户，主要是为了获取最新版本信息；但仍然建议用户时常同步官方指导，比如 Spring Boot 文档建议的基镜像随着版本升级在不断变化。
   - 由于公共仓库会接纳各种用户发布的镜像，并不能保证其中没有恶意软件，因此在自己没有十分把握之前，**只选择有可信标记的镜像**！
     - 不同公共仓库的可信标记不一定相同，[AWS](https://gallery.ecr.aws/) 的是 **Verified Account**，来源是 Docker official images、Amazon、Verified publishers 这三种，其他仓库即使不叫这个名字也应该能很直观的识别。
     - 如果一个公共仓库没有可信标记等类似机制（比如 [Quay](https://quay.io/)？），那就**不要**直接在这个仓库搜索选取。
     - GitHub 仓库比较特殊，它的主视角是源代码项目，因此首先要信任的是这个项目、信任这是官方源码，这实际是上一步的过程。
     - 可信镜像**不代表**绝对不出问题，只是说相对概率会小。
1. 部分构建工具提供了默认的基镜像，比如 [Jib](https://github.com/GoogleContainerTools/jib/blob/v3.4.6-maven/jib-cli/src/main/java/com/google/cloud/tools/jib/cli/jar/JarFiles.java#L85) 或 [Paketo Buildpacks](https://paketo.io/docs/reference/java-reference/)（[Java](https://github.com/paketo-buildpacks/java)、[Node.js](https://github.com/paketo-buildpacks/nodejs) 等），也就是说用户不需要自己定义 Dockerfile 或操心基镜像的选择，而这些主流构建工具推荐的镜像也可以看成一种"官方"、至少是可信的。

### 可用仓库

如 [Container Run TLDR](container-run-tldr.md#public-registry) 所述，虽然 Docker 仓库最大最主流，但由于**限流和墙**的因素，对我们并不是一个好的选择，因此即使按上一节确定了官方镜像，我们仍需要继续梳理选择，但要先熟悉一些容器方面的知识点：

- 如果官方文档里给出的镜像名称只是一个简单的 [nginx](https://cli.vuejs.org/guide/deployment.html#docker-nginx)，那么要知道这是 `docker.io/library/nginx:latest` 的缩写，这里我们先关注它的域名表示 Docker 仓库。
- 文档里给出的可能是镜像名称也可能是仓库地址，这两者都会包含域名，因此要知道 `public.ecr.aws` 和 `gallery.ecr.aws` 代表的都是 AWS 仓库（[ecr](https://docs.aws.amazon.com/AmazonECR/latest/userguide/what-is-ecr.html) 是 Elastic Container Registry 缩写），`ghcr.io` 是 GitHub 仓库（`ghcr` 是 GitHub Container Registry 缩写）等等。

如果一个产品的官方镜像发布至多处，那么**优先选择 AWS 或 GitHub 仓库**。除了大的公共仓库，还有一些软件厂商自己搭建的仓库，只要厂商本身没问题，那么这些小众仓库也可以认为是可信的。因此以下主要考虑如何**在 AWS 仓库查找 Docker 镜像的替代品**（GitHub 通常只保存用户项目本身的容器镜像，不适合这种用途）：

- Docker 官方镜像（前缀 `docker.io/library`）：AWS 的一个亮点就是提供 [Docker official](https://gallery.ecr.aws/docker/) 镜像的拷贝，可能不包括太早的镜像？基本可以说有一个 `docker.io/library/xxx` 就存在一个 `public.ecr.aws/docker/library/xxx`，即使软件官方文档未提及；如果是这种情况，**强烈建议**尽快转到 AWS 仓库。
- Docker 非官方镜像：
  1. [Bitnami](https://gallery.ecr.aws/bitnami/) 制作了大量主流软件的 [Non-Root](https://docs.bitnami.com/kubernetes/faq/configuration/use-non-root/) 镜像，因此可以先确认 Bitnami 是否提供了想要使用软件的镜像。
     - 不过要注意，虽然 AWS 的 Docker official 镜像和 Docker 镜像是一模一样的，但 Bitnami 镜像只是包含了和 Docker 镜像相同的软件，而镜像本身比如软件所在的目录等等是不同的。因此如果是从 Docker 镜像迁移到 AWS 的 Docker official 镜像，那么只用修改镜像名称，但迁移到 Bitnami 镜像，很有可能要调整比如 K8s Pod 的其他配置。
     - 不过用 Bitnami 也有好处，无论什么软件，Bitnami 在文件路径、配置等方面都保持了一致的风格，如果用了一个 Bitnami 镜像，再用另一个完全不同种类软件的 Bitnami 镜像，上手也会更快。
  1. 尝试从其他认证账号找到相同软件，这个只能具体情况具体分析，虽然都是认证账号，但还是如 Bitnami 这样规模大的更可靠。

由于 Docker 的限流和墙是我们搭建 PaaS 平台之后发生的，因此有用户可能还在使用其上的镜像，请**尽快**迁移到以上替代方案。

### Tag

确定了镜像名称和所属仓库后，还有一个步骤就是确认容器镜像的版本也就是 Tag，但是和其他种类的很多制品不同，**Tag 除了包含软件版本外通常还包含容器镜像方面的更多信息**，我们需要在此说明。以 OpenJDK 的容器镜像 `eclipse-temurin` 为例，从官方文档 [Supported tags and respective Dockerfile links](https://github.com/docker-library/docs/blob/master/eclipse-temurin/README.md#supported-tags-and-respective-dockerfile-links)（[实际仓库地址](https://gallery.ecr.aws/docker/library/eclipse-temurin)）可以看出即使某一个具体的 Java 版本也会有大量 Tag 存在：

- 滚动 Tag `latest` 当前指向 `21` 及 `21-jdk` 乃至固定 Tag 也就是最具体的 Java 版本 `21.0.7_6-jdk`。
  - **注意**这是截至 2025-06 的状态，之后肯定会指向更新的 LTS 版本，以下类似情况不再提示。
  - `latest` 或纯数字如 `21` 等动态 Tag 默认指向 JDK 镜像，如果是 JRE 则需要明确使用 `21-jre` 等。
- `17.0.15_6-jre` 会同时指向 Linux 镜像 `17.0.15_6-jre-noble` 和其他 Windows 镜像，这种 Tag 叫 Shared Tag。
  - 像 `17.0.15_6-jre-noble` 这种明确指定了操作系统 / 运行平台的就叫 Simple Tag，详细说明参见 [What's the difference between "Shared" and "Simple" tags](https://github.com/docker-library/faq?tab=readme-ov-file#whats-the-difference-between-shared-and-simple-tags)。
  - Windows 相关 Tag 包含关键词 `windowsservercore` 或 `nanoserver` 等，由于不是我们的使用场景不多提及，详细说明参见 Windows 文档 [Container Base Images](https://learn.microsoft.com/en-us/virtualization/windowscontainers/manage-containers/container-base-images)。
  - 当使用 Shared Tag 时由容器运行时根据宿主机的操作系统版本决定实际使用哪个镜像。
  - 指向 Shared Tag 的 Tag 如 `latest`、`17-jre` 自然也是 Shared Tag。
  - Tag 的"动态 Vs. 固定"和"Shared Vs. Simple"不是一回事。
- 可能有人不理解 `noble` 为什么是 Linux？这实际是 Ubuntu 的版本代号。
  - [Ubuntu](https://wiki.ubuntu.com/Releases) 最近的 LTS 版本包括 24.04（`Noble Numbat`）、22.04（`Jammy Jellyfish`）、20.04（`Focal Fossa`），因此对应的容器镜像类似 `17-focal`、`17-jammy`、`17-noble`，但到最新 Java 就只有 `24-noble` 了，显然我们应该尽量用最新的操作系统版本。
- [Debian](https://www.debian.org/releases/) 最近的版本包括 12（`Bookworm`）、11（`Bullseye`）、10（`Buster`）。
  - eclipse-temurin 没有制作 Debian 版，但 [node](https://github.com/docker-library/docs/blob/master/node/README.md) 有 `24-bookworm`、`24-bullseye` 等 Tag。
  - 注意 node 还有 `slim` 这个特殊关键词，如 `24-slim`、`24-bookworm-slim`、`24-bullseye-slim` 等，这个代表 Debian 的精简版。虽然 Slim 是一个通用词，但似乎只有 [Debian](https://github.com/docker-library/docs/blob/master/debian/README.md#debiansuite-slim) 在用，其他用词包括 `nano`、`minimal` 等，而 [Ubuntu](https://github.com/docker-library/docs/blob/master/ubuntu/README.md) 就没有或者只有精简版（以下有体积对比的讨论）。
- 虽然 Debian 有精简版，但最常见的是设计初就以精简为目标的 [Alpine Linux](https://www.alpinelinux.org/) 发行版。
  - 从 Docker 众多产品的[官方镜像](https://github.com/docker-library/docs/)可以发现，各个软件有基于 Debian 或 Ubuntu 的镜像、但很少两者都有，但同时基本都有基于 Alpine 的精简镜像。
  - 因此 eclipse-temurin 有 `24-jre-alpine` 等 Tag。
  - 由于 Alpine 没有版本代号，所以 eclipse-temurin 的 Tag 又细分为 `24-jre-alpine-3.20`、`24-jre-alpine-3.21`，前面一个数字是 Java 版本、后面一个数字是 Alpine 版本。
  - Alpine 的一个问题是"based on musl libc"，而非 Debian、Ubuntu 主要使用的 glibc，兼容性问题参见 [Is musl compatible with glibc](https://www.musl-libc.org/faq.html)，但对我们的普通业务应用可能并无区别？
  - 还有极度精简的 [Distroless](https://github.com/GoogleContainerTools/distroless) 镜像，连 Shell 都没有，普通应用基本不需要考虑。
- 其他操作系统还包括红帽或 Oracle，比如 [17-ubi9-minimal](https://github.com/docker-library/docs/blob/master/eclipse-temurin/README.md)、[8-oraclelinux9](https://github.com/docker-library/docs/blob/master/mysql/README.md)，但这两者**严格来说只算免费而不是开源**？总之不是优先考虑的选项。
- Tag 中还有更多的关键词，比如 [nginx](https://github.com/docker-library/docs/blob/master/nginx/README.md) 的 `1.28-otel`、`1.28-perl` 等，这些词只能具体产品具体分析，从产品文档中确认，但相对也比较直观。

注意以上内容完全是基于 Docker 公司的设定，包括 Shared Tag 的术语、动态 Tag 的默认指向、代表操作系统及版本的关键词、是在 Tag 区分 JDK / JRE 还是在镜像名就直接分开，等等等等；换一个容器镜像制作商可能就是另一种做法，比如 Bitnami 制作的 [nginx](https://gallery.ecr.aws/bitnami/nginx) 镜像 Tag 就没有用 `bookworm` 而是 `1.29-debian-12`、`1.25-debian-11` 等，甚至 Docker 自己比如 [mysql](https://github.com/docker-library/docs/blob/master/mysql/README.md) 也有不用 `buster` 而是 `8.0-debian`的。总之**不要将以上内容理解为业界通行的规范，能够了解容器镜像 Tag 各部分的由来就好**。以下为挑选 Tag 的建议：

- **只使用固定 Tag**，不要使用滚动 Tag。
  - 在官方特别是入门文档里的示例经常使用 `latest`，可以理解如果使用固定 Tag 那要么很快过时要么得频繁更新，但这种做法来自大量不同产品的文档，还是给用户造成了误导。
  - 在 Tag 中"固定"这个词也没有明确规则，比如 `17.0.15_6-jre` 和 `17.0.15_6-jre-noble` 都固定了 Java 版本，但只有后者还固定了所运行的操作系统及版本。
  - **至少固定容器所含应用的版本，而个人倾向于全部固定**，因为存在制作商在升级时改换默认操作系统或版本的可能。
  - 在 Bitnami 的 [Usage recommendations](https://techdocs.broadcom.com/us/en/vmware-tanzu/application-catalog/tanzu-application-catalog/services/tac-doc/apps-tutorials-understand-rolling-tags-containers-index.html) 中建议是生产环境使用"immutable tags"、开发环境使用"rolling tags"，但个人倾向于**全部**使用固定 Tag，同时开发人员要**有意识**持续跟进、手工调整升级。
- 尽量使用 **LTS** 版本的 Tag，除非确定能够做到持续升级。
  - 但哪些版本算 LTS 这个没有统一规则，只能具体领域具体确认，比如 [Oracle Java SE Support Roadmap](https://www.oracle.com/java/technologies/java-se-support-roadmap.html) 或 [Node.js](https://nodejs.org/en/download)。
- **只使用精简版本**，Tag 中的关键词类似 `alpine` / `slim` 等或者是 Ubuntu 镜像。
  - 以 [node](https://gallery.ecr.aws/docker/library/node) 22.17.0 为例，`22.17.0-bookworm` 为 423M、`22.17.0-bookworm-slim` 为 85.7M、`22.17.0-alpine3.22` 为 57.5M，显然应该使用后两者。
  - node 的 `22.17.0-bullseye-slim` 为 81M，比以上 `22.17.0-bookworm-slim` 略低，但后者是更新的操作系统版本，在体积差别不大的前提下优先考虑新版本。
  - 基于相同 JDK 版本的容器镜像，[eclipse-temurin](https://gallery.ecr.aws/docker/library/eclipse-temurin) 的 `17.0.2_8-jdk-focal` 为 245M，[openjdk](https://gallery.ecr.aws/docker/library/openjdk) 的 `17.0.2-jdk-slim-bullseye` 为 221M、`17.0.2-jdk-bullseye` 为 327M，Ubuntu（`focal`）镜像体积在 Debian 的精简版和完整版之间且偏小，虽然 Ubuntu 没有明确提精简版，但也可以说只有精简版。
- 操作系统：如上所述，一个产品的容器镜像有基于 Debian 或 Ubuntu 但很少两者都有，因此这里的选择聚焦在选主流操作系统的精简版还是原生精简的 Alpine？
  - 以上有提及 Alpine 的"musl libc"的问题，但个人认为对普通应用没什么影响；而且基于 Alpine 制作的产品镜像也是经过厂商检验的，也有足够多的真实用例证明。
  - 由于精简镜像的一个问题是缺乏常用 Debug 工具，因此我们粗略比较一下三个系统的默认工具：
    | 容器镜像                                             | 体积  | curl | nc | ping | ps | telnet | wget |
    | ------                                               | ----- | ---- | -- | ---- | -- | ------ | ---- |
    | `public.ecr.aws/docker/library/alpine:3.22.0`        | 4.14M |   ×  |  √ |   √  |  √ |    ×   |   √  |
    | `public.ecr.aws/docker/library/ubuntu:noble`         | 34.3M |   ×  |  × |   ×  |  √ |    ×   |   ×  |
    | `public.ecr.aws/docker/library/debian:bookworm-slim` | 32.1M |   ×  |  × |   ×  |  × |    ×   |   ×  |
  - 但基于以上操作系统镜像构建的软件镜像并不是说肯定和上面结果一致，在构建过程中同样可以增减工具，比如 [eclipse-temurin](https://github.com/docker-library/docs/blob/master/eclipse-temurin/README.md#supported-tags-and-respective-dockerfile-links) 的 [noble](https://github.com/adoptium/containers/blob/main/17/jre/ubuntu/noble/Dockerfile#L30) 镜像有添加 curl 工具、但 [alpine](https://github.com/adoptium/containers/blob/main/17/jre/alpine/3.21/Dockerfile) 没有。
  - 虽然个人倾向于体积更小的 Alpine，但用户可以综合以上因素自行选择，尽量**保持一致**就行。
    - 如 eclipse-temurin `17.0.15_6-jdk` 默认指向的 Linux 版本是 `17.0.15_6-jdk-noble` 而非 `17.0.15_6-jdk-alpine`，可以将这当成厂商的倾向，但不同产品的倾向并不相同；而且也可能代表不了什么，比如 `17` 默认指向的是 `17-jdk`，但我们用的更多的是 `17-jre`。

## 主流技术栈

根据以上[择选](#择选)各节的共性步骤，用户基本能胸有成竹的确定一个可靠适用的第三方公共容器镜像，但具体到各技术领域，仍有更多细节可以补充。

### Java

Java 领域的关键词是 `OpenJDK`，Java 技术人员应该了解这个词的由来，不过还是解释一个常见误解，OpenJDK 是一个项目或者说品牌，因此具体制品或容器镜像是 JDK 还是 JRE 还需要通过其他信息来确定。总之我们在 AWS 仓库即使搜 JRE 镜像也是用 `OpenJDK` 这个词，但也有了下面的疑问，因为有多个认证的[结果](https://gallery.ecr.aws/search?searchTerm=OpenJDK&verified)：

- `amazoncorretto/amazoncorretto`
- `docker/library/eclipse-temurin`
- `docker/library/amazoncorretto`
- `docker/library/openjdk`
- `docker/library/ibm-semeru-runtimes`
- `docker/library/sapmachine`

这实际上是不同厂商基于 OpenJDK 制作的容器镜像，那么即使是同一个 Java 版本，我们又应该选择哪个镜像？如果将场景局限到 Java 开发框架 Spring，我们看看 Spring Boot 文档的建议：

- [3.4](https://docs.spring.io/spring-boot/3.4/reference/packaging/container-images/dockerfiles.html)：`FROM bellsoft/liberica-openjre-debian:24-cds`
- [3.3.0-M2](https://docs.spring.io/spring-boot/docs/3.3.0-M2/reference/html/container-images.html#container-images.dockerfiles)：`FROM eclipse-temurin:17-jre`
- [2.6.7](https://docs.spring.io/spring-boot/docs/2.6.7/reference/html/container-images.html#container-images.dockerfiles)：`FROM eclipse-temurin:11-jre`
- [2.6.6](https://docs.spring.io/spring-boot/docs/2.6.6/reference/html/container-images.html#container-images.dockerfiles)：`FROM adoptopenjdk:11-jre-hotspot`
- [2.2.13](https://docs.spring.io/spring-boot/docs/2.2.13.RELEASE/reference/htmlsingle/#containers-deployment)：`FROM openjdk:8-jre-alpine`
- [2.1.18](https://docs.spring.io/spring-boot/docs/2.1.18.RELEASE/reference/htmlsingle/)：无基镜像建议。

另外容器镜像构建工具 [Jib](https://github.com/GoogleContainerTools/jib/blob/v3.4.6-maven/jib-cli/src/main/java/com/google/cloud/tools/jib/cli/jar/JarFiles.java#L85) 默认的基镜像是 `eclipse-temurin`，Paketo Buildpacks 的构建镜像是操作系统镜像（有多个选择），在构建过程中下载安装 Java 文件，而 Java 二进制文件使用的是 [BellSoft Liberica JRE](https://github.com/paketo-buildpacks/bellsoft-liberica/blob/v11.2.4/buildpack.toml#L157) 或 JDK；注意 Paketo 还给出了更多的 [Supported JVM Vendors](https://github.com/paketo-buildpacks/jvm-vendors/blob/main/README.md#supported-jvm-vendors)，有兴趣的可以进一步研究。

综上，OpenJDK 的容器镜像就有这么多选择，我们来逐个排除：

1. 先排除小众且明显是商用机构的 `sapmachine`、`ibm-semeru-runtimes`。
1. 虽然 [Corretto is distributed by Amazon under an open source license](https://aws.amazon.com/about-aws/whats-new/2022/09/amazon-corretto-19-available/)，但背后的 Amazon 不是一个中立的开源组织，如果不是运行在 AWS 上的话暂不考虑。
1. `openjdk` 已 [**officially deprecated**](https://gallery.ecr.aws/docker/library/openjdk)，**注意**这是名称叫 `openjdk` 的容器镜像、而非 [OpenJDK](https://openjdk.org/index.html) 开源项目。
   - 以 Java 17 为例，现在（2025-06）容器镜像 [openjdk](https://gallery.ecr.aws/docker/library/openjdk) 的最高版本只到 `17.0.2`，对比 [eclipse-temurin](https://gallery.ecr.aws/docker/library/eclipse-temurin) 的 `17.0.15`，差了太多个 [SECURITY](https://openjdk.org/jeps/223) 版本，Java 11 类似；而这两者也是我们 [*PaaS 平台*](why-here.md)用户使用最多的镜像，**请尽快迁移升级**！
   - 不过奇怪的是既然号称 [DEPRECATED](https://github.com/docker-library/openjdk)，但在 2025-06-10 仍有 [Merge pull request #548 from marchof/java-26](https://github.com/docker-library/openjdk/commits/master/) 更新？
1. [AdoptOpenJDK](https://blog.adoptopenjdk.net/2021/03/transition-to-eclipse-an-update/) 已迁移到 [Adoptium](https://adoptium.net/)，相关产品命名为 Eclipse Temurin。

所以我们最终是要在 [**Eclipse Temurin**](https://adoptium.net/temurin) 和 [**BellSoft Liberica**](https://bell-sw.com/libericajdk/) 这两个 OpenJDK 发行版之间选择：

- 首先从网上的调研，[Eclipse Temurin vs BellSoft Liberica](https://cn.bing.com/search?q=Eclipse+Temurin+vs+BellSoft+Liberica) 没有明显倾向，用哪个都不算错；即使有性能或体积上的少量差距，对于我们远未到海量请求、需要极度压榨机器性能的场景，至少也不是决定性因素。
- [The Eclipse Foundation is an international non-profit association](https://www.eclipse.org/org/)，而 BellSoft 是 [COMPANY](https://bell-sw.com/about/)。
- 从热门的角度 Temurin 要胜一筹，因为更早使用，还比如 Jib 默认值，以及容器镜像 [openjdk](https://gallery.ecr.aws/docker/library/openjdk) 在废弃时建议的替换对象不包括 Liberica。
- Spring 最新背书的是 Liberica？可能也不算背书，只发现了文档示例中的改变，并未找到 Spring 推荐用户从 Temurin 转 Liberica 的官方提示或说明。
  - BellSoft 的[文档](https://bell-sw.com/blog/how-to-dockerize-a-spring-boot-app-with-the-smallest-base-image/)认为是 [used and recommended by the Spring team](https://bell-sw.com/announcements/2021/09/02/BellSoft-VMware/), 但该链接的内容是"A support agreement of VMware and BellSoft for Spring Boot"（[SpringSource will become a division within VMware](https://spring.io/blog/2009/08/10/springsource-chapter-two)），从这个角度，感觉不是一个纯技术考量？
- AWS 仓库有 [BellSoft](https://gallery.ecr.aws/bellsoft/) 但不是认证账号，从下载的角度使用 Docker 的 Liberica 不太方便。
- 注意从 Spring Boot 3.4 启用 [CDS](https://docs.spring.io/spring-boot/3.4/reference/packaging/container-images/dockerfiles.html#packaging.container-images.dockerfiles.cds) 功能时使用的是 `bellsoft/liberica-openjre-debian:24-cds`，理论上基于同样 OpenJDK 的 `eclipse-temurin` 肯定能实现，但从开发人员友好的角度，BellSoft 也提供了 [How to use CDS with Spring Boot applications](https://bell-sw.com/blog/how-to-use-cds-with-spring-boot-applications/) 等帮助文档，而 Eclipse 的尚未搜到，因此不知道实际上会多折腾。
  - 还有 [Native Image](https://docs.bell-sw.com/liberica-nik/latest/how-to/build-native-image-from-springboot/)。

个人倾向是 Eclipse Temurin，决定因素是 Eclipse **更中立开源**，但用户也可以尝试 BellSoft Liberica，特别是在新的技术特性上，不过要注意下载限制。

另外注意官方指导文档 [Spring Boot with Docker](https://spring.io/guides/gs/spring-boot-docker) 太久未更新，示例使用的基镜像还是 `openjdk:8-jdk-alpine`（截至 2025-06），而应用运行通常基于 JRE 就行（Tag 中的 `8-jdk-alpine` 才表示是 JDK，`openjdk` 只是一个镜像名称），没必要还带 `javac` 之类的工具；更早文档有给出解释"Not all applications work with a JRE (as opposed to a JDK), but most do."（但[链接](https://spring.io/guides/topicals/spring-boot-docker/#_smaller_images)已失效）。不管如何到现在这应该不是一个问题，至少也应该先从 JRE 镜像做起。

### Node / Nginx

Node.js 前端开发我们以 [Vue](https://cli.vuejs.org/guide/deployment.html#docker-nginx) 为代表，实际部署主要使用的是 `nginx` 而非 `node` 镜像，因此这里主要提示一下 Nginx 镜像的问题，无论是用于前端应用静态网站还是 Nginx 代理。参见 Container Run TLDR 的 [Unprivileged Image](container-run-tldr.md#unprivileged-image) 一节，Nginx 我们应该使用 [docker-nginx-unprivileged](https://github.com/nginx/docker-nginx-unprivileged/blob/1.28.0/README.md) 而非 [docker-nginx](https://github.com/nginx/docker-nginx/blob/1.28.0/README.md) 的镜像，由于早期没有 `unprivileged` 镜像，很多用户使用的就是后者，因此需要**尽快调整**：

- 镜像名称从 `docker.io/nginx` 或 `public.ecr.aws/docker/library/nginx` 调整为 `public.ecr.aws/nginx/nginx-unprivileged`。
- 最新 Tag 通过 [AWS 仓库](https://gallery.ecr.aws/nginx/nginx-unprivileged)查询获取。
