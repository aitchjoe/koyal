---
title: Container Run TLDR
---

容器入门实操，面向容器用户或潜在用户，请首先阅读概念向的 [Container TLDR](container-tldr.md)。还是建议全体 IT 人员都能尝试，一者即使最粗浅的实操也更有助于理解概念，二者容器基本已等同 Linux 的地位，IT 从业者无论做不做技术都应该亲身感受一下。

但是**注意**本文并不是指示一步一步操作的引导文档，这个无论官方还是网上都有太多，以下列出的主要是网上提及较少的内容或者人家不会碰到的苦难，总之是我们用户在入门过程中容易引起困扰的点。

## Development Environment

- 在容器时代之前，本地开发的一大麻烦是准备应用的运行时依赖如数据库等，当然也有不少设计尝试缓解这个问题，但一般都局限在特定领域；而容器则用极粗暴的方式统一解决了该问题，就是直接大幅度降低所有软件安装部署的难度。
  - 云原生改造的[运行时依赖](cn-transform.md#运行时依赖)一章有详细讨论。
  - 和现在不同，容器诞生初期主要解决的就是开发阶段的麻烦。
  - 到现在也不止是简单的用容器部署中间件等，而是发展成了整体解决方案，如 [Testcontainers](https://testcontainers.com/) 号称"Unit tests with real dependencies"，而 Spring 也 [Improved Testcontainers Support in Spring Boot 3.1](https://spring.io/blog/2023/06/23/improved-testcontainers-support-in-spring-boot-3-1)。
- 虽然容器极大简化了应用部署，但在本地开发环境安装容器运行软件却不是件简单的事。
  - 由于容器的技术原理，并不能在 Windows 环境直接运行 Linux 容器；Windows 容器当然是一个选项，但我们的实际应用部署环境都是 Linux。
  - WSL、WLS2 从我们之前的尝试来看都很难用，而且对 Windows 版本也有要求。
- 目前实际做法是装 Linux VM 在其中跑容器，但外包伙伴本身已是 Windows VM 了。
- 有远程容器支持本地开发的产品方案，但这是另一个需求场景，仍然希望用户能在本地很方便的尝试容器技术。
- 还有 Cloud IDE 方案，它解决的不是本地开发环境的依赖准备问题，而是直接解决了"本地开发环境"。
- 从开发人员的日常使用场景，其实使用 Linux 作为本地开发环境也不是完全不能考虑的选项，比如飞书也有 [Linux](https://www.feishu.cn/download) 版本（Beta）。

## Image Name

- 关于容器镜像的命名或者说 Fully qualified name，无论 Docker 公司还是 OCI（容器标准化组织）都没有专门文档清晰的说明，而这往往是用户遇到的第一个困扰。
- 甚至唯一标识容器镜像的术语是叫镜像名还是镜像 ID 等等也没有明确说法，我们一般采用常见的容器镜像名表示。
- 同样，关于容器镜像名的各部分术语也没有一个明确说法。以常见 `hello-world` 的完整名称 `docker.io/library/hello-world:latest` 为例，`:` 后的部分（`latest`）叫 Tag 这个是一致清晰的，但 `:` 前的是叫 Repository 还是 Registry（域名部分）+ Repository 也没说法，`library` 部分显然是分层、可以一层或多层，有叫 Namespace 的但也未普及。总之在术语方面是一片混乱，不过对实际使用也没什么影响。
- 关于完整镜像名的缩写规则，仍以之上的 `hello-world` 为例，由于 Docker 的先发优势、默认的 Registry `docker.io` 可以省略，Docker 官方默认的 Namespace `library` 可以省略、但注意这只是 Docker 的规矩换一个 Registry 很可能就不同了，最后是默认的 Tag `latest` 可以省略。
- 使用镜像名称缩写虽然方便但**强烈不建议**使用，因为 `docker.io/nginx` 和 `quay.io/nginx` 不保证是同一镜像、甚至有恶意冒充的可能，而有些容器运行环境会配置 Registries Search，比如针对 `xxx`，找不到 `docker.io/xxx` 就自动搜寻下一个 `quay.io/xxx`，这显然是反模式。

## Image Tag and Digest

- 容器镜像的版本叫 Tag，和绝大部分技术栈的 Version 一样，技术上都未限制同一个 Tag 必须始终指向同一内容，只是语义上遵守一个业界约定俗成的规范。
- 和 Java 常见做法不同，容器每次发版时常会打多个 Tag，比如发布 `1.0.0` 时还会打上 `1.0`、`1`、`latest` 指向同一版本，当升级到 `1.0.1` 时这三个又会同时调整指向最新的 `1.0.1`；虽然业界没有统一命名，但我们可以将精确版本的 `1.0.0` 等叫**静态 / 固定** Tag、而 `latest` / `1` 等叫[**动态 / 滚动**](https://techdocs.broadcom.com/us/en/vmware-tanzu/application-catalog/tanzu-application-catalog/services/tac-doc/apps-tutorials-understand-rolling-tags-containers-index.html) Tag。
  - 同理，即使固定 Tag 也只是从语义约定上表示不会变化，但技术上是限制不了的；而 `1.0.0` 也只是 [SemVer](https://semver.org/) 的理论设计，很可能在真实情况中某产品的 `1.0.0` 先指向 `1.0.0-x1` 再调整到 `1.0.0-x2`，所以必须具体产品具体确认。
- 虽然引用镜像时使用动态 Tag 貌似比较轻松、随时自动升级，但我们遇到过**真实案例**，`openjdk:11-slim` 仅仅是从 `11.0.2` 自动升级到 `11.0.3` 就导致应用[*出错*](why-here.md)，所以请大家尽量使用固定版本以避免隐患。
- 数字或 `v` 开头的数字，大家基本都有这代表正式版本的共识，但是注意 `latest` 语义在业界基本是用来指代最新正式版，但我们内部已经习惯用 `latest` 表示最新的测试版本，请大家注意不要产生误会。
- 关于 `latest` 的缓存行为预期：由于容器运行环境通常会缓存镜像避免重复下载，但如上所述 Tag 并不保证指向同一内容，因此用户方必须告诉运行环境是否使用缓存，但不同环境的提示方式是不一样的，比如 Docker CLI 是 `docker run --pull=always`，而 Kubernetes 是 `imagePullPolicy: Always`。但 OpenShift 用户可能会被误导，比如 `latest` 即使不指定 `imagePullPolicy` 也会每次都拉取，这实际上是 OpenShift 根据 Tag 自动添加了相应 `imagePullPolicy` 策略，而非容器运行环境将 `latest` 特殊对待。
- 可以在镜像名称中使用 `@DIGEST` 替代 `:TAG`，使用这个 Hash 值才能真正唯一标识容器镜像，这样做更安全，但是大多场景既不方便也不直观，没有必要。
- 容器 Tag 的命名中还包含更复杂的信息，详细说明参见[公共容器镜像](container-public-image.md#tag)的 Tag 一节。

## Mirror

- 由于 Container Image 翻译成容器镜像、Mirror 也翻译成镜像，请大家通过上下文区分，或者前者也可以叫容器制品。
- 企业内通常会限制了互联网访问，并不能直接下载制品及制品依赖，因此不止容器、任何一个技术栈的 Hello World 在企业内部都不能简单跑起来，以至于我们有一个 [*Hello Hard World*](why-here.md) 的专题。
- 而即使能访问互联网，考虑到安全或网速带宽等问题，我们也建议使用外部制品的内部镜像。
- 因此用户另一个常见的困扰点是如何使用容器制品的内部镜像，我们现在的内部镜像服务是基于 JFrog 的 [*repos.example.com*](why-here.md)（以下简称 repos）。
- 不同技术栈制品的内部镜像使用方式并不一致，这个我们在 Mirror TLDR 中详述，本文只关注容器制品的镜像使用。
- 以 `docker.io/nginx` 为例，在 repos 上的对应内部名称则是 `repos.example.com/docker-io/nginx`，主要就是添加 repos 域名并将原域名中的 `.` 改为 `-`，注意这只是 repos 的规则，并不代表所有产品、或者同一产品的不同部署都遵循该规则，甚至 repos 的管理员也可能疏忽不按规则配置新镜像。
- 但是由最终用户操心如何转换并不是一个理想方式，实际上是可以在容器运行环境配置这种映射规则（由运行环境的管理员负责），而最终用户还是使用官方标准名称，这也是**接口实现分离**的原则。
- 因此用户的**日常使用场景**应该是首先在以下的 Public Registry 查询所使用的容器制品名称，其次确认所运行的容器环境是否已配置该映射，最后即正常使用。

## Public Registry

- 第三方容器制品应尽量从大的公共容器镜像仓库（Registry）中选择，以下会详细列出；而即使是大网站，也应该尽量只使用其中的**官方或认证制品**，对于小众制品**务必再三确认**。
- 注意容器镜像名称中的域名和仓库域名通常不一样，典型如 `docker.io` Vs [hub.docker.com](https://hub.docker.com/)，前者用于下载镜像、后者有 Web 界面供用户搜索查询。
- [Docker Hub](https://hub.docker.com/)（`docker.io`）：最早最大的公共仓库，也拥有最多的主流软件，但由于免费用户的限流导致了拉取镜像频繁出错、方便用户搜索查询的 Hub 也被墙了，因此在以下 AWS 提供相同镜像的前提下**尽量**使用 AWS 的。
- [Quay.io](https://quay.io/)（`quay.io`）：早期较大的仓库，但正规制品主要是红帽系的，其他基本已边缘化。
- [Amazon ECR Public Gallery](https://gallery.ecr.aws/)（`public.ecr.aws`）：在 Docker 和 Quay 之后做大的，有部分因素是 Docker 限流，它的一个主要好处就是提供了 Docker official images（对应 `docker.io/library`）。
- [GitHub](https://docs.github.com/en/packages/working-with-a-github-packages-registry/working-with-the-container-registry)（`ghcr.io`）：GitHub 全家桶的容器制品仓库，没有如 Docker Hub 这样专门的搜索界面，直接进入项目页面查询；虽然早期主流的容器制品较少，但是凭 GitHub 的号召力，位于 GitHub 的开源软件也逐渐把以前发布到 Docker Hub 等处的制品转移或同步过来。
- 其他如[红帽系](https://access.redhat.com/RegistryAuthentication)、Google 系（`gcr.io` / `registry.k8s.io`）等多个仓库主要面向平台方，虽然红帽提供的 [Universal Base Images (UBI)](https://access.redhat.com/articles/4238681) 也适合作为部署到 OpenShift 的应用的基镜像，但并不是业界主流，因此普通用户可以忽略。
- [麒麟](https://cr.kylinos.cn/zh)（`cr.kylinos.cn`）：除了麒麟操作系统也提供了基于该系统的一些软件镜像，但非常有限甚至可以说惨不忍睹，比如到 2025-06 也只提供了 [JDK](https://cr.kylinos.cn/zh/image?name=java) 8，如果不考虑**信创**这是完全不实用的一个仓库，但现在仍需了解。
- 国外仓库的国内镜像：现状是国外主流软件基本不会发布到国内仓库，不过国内互联网大厂或高校会提供一些国外公共仓库的国内镜像，但由于免费等因素状况堪忧，只能作为一个备份方案。
- **普通用户尽量从 AWS 或 GitHub 仓库选择容器镜像**，详细说明参见[公共容器镜像](container-public-image.md)。
- 能够使用以上仓库的内部镜像的前提是 JFrog 管理员已经做了相应的配置，参见以上 Mirror 一节。

## Unprivileged Image

- 早期很多容器镜像都只能以 `root` 或指定 UID 的权限运行，[Kubernetes](k8s-tldr.md) 对此并未限制，但 OpenShift 从一开始就做了严格约束，要求普通应用的容器必须以随机的十位数 UID 成功运行，也就是说需要的是 Unprivileged Image。
- 在早期大量镜像不符合我们 OpenShift 平台要求、但又不得不以其为基镜像的情况下，我们指导用户的是在 Dockerfile 中粗暴的执行类似 `RUN chmod 777 /app` 语句解决权限问题，显然这不是一个正规做法且存在风险。
- 在中期关注到这个问题的是 Bitnami，和 Docker 公司一样，Bitnami 制作了大量主流软件的 [Non-Root](https://docs.bitnami.com/kubernetes/faq/configuration/use-non-root/) 镜像，虽然影响力没有 Docker 大但值得尝试。
- 实际上红帽本身也提供了一些符合 OpenShift 要求的容器镜像如 [Nginx](https://catalog.redhat.com/search?gs&q=nginx&searchType=containers)，早期未意识到去采用，不过品种有限，而且如上一节 UBI 所说也不是业界主流，因此也只作为一个备份方案。
- 当容器 Kubernetes 越来越成为生产部署的主流，这个安全隐患自然不用我们独自操心，而是业界都会关注解决，典型如 Nginx 提供了 [NGINX Unprivileged Docker Image](https://github.com/nginx/docker-nginx-unprivileged)，但不理解 Nginx 为何不废弃 Privileged Image 或者至少主推 Unprivileged Image？
- 请用户**尽量使用**同一产品的 Unprivileged 镜像、尽量避免 `chmod 777` 方式；但是业界并没有一个类似 `nginx-unprivileged` 的镜像命名规范，仍需要用户具体情况具体分析。

## Dockerfile

- [Dockerfile](https://docs.docker.com/reference/dockerfile/) 是一个文本文件，包含用于构建容器镜像的所有指令。
- 由于历史原因叫 Dockerfile，但现在越来越多使用 Containerfile 这个名称。
- 和其他所有制品相比，容器制品是构建过程最复杂、对构建环境要求最高的。一个重点是因为 Dockerfile 里的 [RUN](https://docs.docker.com/engine/reference/builder/#run) 指令可以"execute any commands"，而该命令所产生的后果，如创建新文件、修改系统配置等，很难在不真正执行该命令时就做出精准预期。
- Dockerfile 并不是构建容器镜像的必须项，还有 [*Jib*](why-here.md)、[Buildpacks](buildpacks-tldr.md) 等无 Dockerfile 方式，当然本质是一样的。
- Jib 方式构建 Java 镜像速度最快、对环境要求最低，还有一个亮点是可以在 Windows 环境构建 Linux 镜像；而关键因素是限制了使用场景、不考虑 Dockerfile 里最复杂的 RUN 指令，以上链接有详细分析。
- 对于 Java 应用的容器镜像构建，**强烈推荐**使用 Jib 取代 Dockerfile 方式；同时我们也尝试了[*非 Java 应用*](why-here.md)的 Jib 构建，效果也不错。
- Dockerfile 的起点（[FROM](https://docs.docker.com/reference/dockerfile/#from) 指令）是基镜像的选择，[公共容器镜像](container-public-image.md)有详细说明。
- 详细构建语法参见 [*Dockerfile 解析*](why-here.md)。

## Docker Vs Podman

- 从日常的容器使用上来说，Podman 基本可以替换 Docker。
- Podman 使用普通用户权限运行，而 Docker 要求 root 用户（不确定现在 [Rootless mode](https://docs.docker.com/engine/security/rootless/#known-limitations) 是否成熟）。
- Podman 能够在运行环境配置内部镜像映射任意容器制品仓库，因此最终用户只需要使用官方标准镜像名称，但 Docker 到 2025-06 仍只能 [Mirror the Docker Hub library](https://docs.docker.com/docker-hub/image-library/mirror/)？这是个人认为最难以忍受的点。
- 早期用 Podman 取代 Docker 的一个最大问题是不支持 Docker Compose，但现在 Compose Specification 的 [Implementations](https://github.com/compose-spec/compose-spec/blob/main/README.md#implementations) 已列出了 [Podman Compose](https://github.com/containers/podman-compose)；不过未实际使用，不确定是否能完整取代 Docker Compose 或者有什么限制。
- Docker 的生态圈确实比 Podman 大，以最近（2024）尝试的 Buildpacks 来讲只支持 Docker。
- 建议用户首先选择 Podman / Podman Compose，实在不行再转 Docker。
