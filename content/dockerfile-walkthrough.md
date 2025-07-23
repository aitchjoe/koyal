---
title: Dockerfile 解析
---

Dockerfile 概要介绍参见 Container Run TLDR 的 [Dockerfile](container-run-tldr.md#dockerfile) 一节，本文详细说明其主要构建指令。

## Spring Boot

### Example

[Spring Boot with Docker](https://spring.io/guides/gs/spring-boot-docker) 文档对 Spring Boot 应用的容器化改造说明的非常详实，比 Docker 官方文档 [Dockerfile reference](https://docs.docker.com/reference/dockerfile/) 更适合作为 Java 技术人员构建容器镜像的入门文档，我们先以该文档的示例为基础做进一步的补充解释，同时**提示**其中误导新人或不适合我们环境的地方。

该文档给出了多个 Dockerfile 示例，我们以它推荐的 Example 3 来介绍最基本的指令：

```
FROM openjdk:8-jdk-alpine
RUN addgroup -S spring && adduser -S spring -G spring
USER spring:spring
ARG DEPENDENCY=target/dependency
COPY ${DEPENDENCY}/BOOT-INF/lib /app/lib
COPY ${DEPENDENCY}/META-INF /app/META-INF
COPY ${DEPENDENCY}/BOOT-INF/classes /app
ENTRYPOINT ["java","-cp","app:app/lib/*","hello.Application"]
```

### FROM

```
FROM openjdk:8-jdk-alpine
```

[FROM](https://docs.docker.com/reference/dockerfile/#from) 表示基于该镜像来制作应用的容器镜像，很显然不管是开发一个业务应用还是构建它的镜像，都不会从零开始。而这句示例也是最有问题的地方：

- 首先该文档到 2025-06 仍在使用 Java 8，但现在最低也该用 11 了，而 [Spring Boot](https://docs.spring.io/spring-boot/3.4/reference/packaging/container-images/dockerfiles.html) 不同版本的 Reference 文档都有这方面的更新。
  - 也可能跟该文档属于 Guide 类型有关，而且主体内容也更新过、基本未过时，但无论如何还是容易误导新人。
- 现在也远不是只有 Docker 这家容器公司的年代，因此正规做法是使用完整的镜像名称，如 `docker.io/openjdk:8-jdk-alpine`。
- 构建 Java 应用运行环境基于 JRE 镜像就好，不需要 JDK；但注意 `openjdk` 代表的是一个项目或品牌，`8-jdk-alpine` 才是指 JDK。
  - [旧文档](https://github.com/spring-attic/top-spring-boot-docker/blob/main/README.adoc#smaller-images)有解释"Not all applications work with a JRE (as opposed to a JDK), but most do."，但 [Reference](https://docs.spring.io/spring-boot/docs/2.2.13.RELEASE/reference/htmlsingle/#containers-deployment) 文档示例早就换成 JRE 了。
- 也不应该使用类似 `8` 这样的动态 Tag，而是精确版本如 `17.0.15`。
  - 我们实际遇到过 `openjdk:11-slim` 从 `11.0.2` 自动升级到 `11.0.3` 就导致应用[*出错*](why-here.md)的案例。
  - 当然这并不能避免手工调整到 11.0.3 后出错，但源码貌似未变（`openjdk:11-jdk-slim`）和能明显在源码看出 `11.0.2` 变更为 `11.0.3`，自然是后一种做法对定位问题有很大帮助。

总之选择使用一个基镜像比想象中复杂，[公共容器镜像](container-public-image.md)里有详细讨论，包括以上问题和更多，**强烈推荐**阅读。

### RUN & USER

```
RUN addgroup -S spring && adduser -S spring -G spring
USER spring:spring
```

[RUN](https://docs.docker.com/reference/dockerfile/#run) 命令本身很直观，就是为了随后的 `USER` 用途，而 [USER](https://docs.docker.com/reference/dockerfile/#user) 用于设置容器启动的默认用户和组：

- 在 Dockerfile `USER` 之后的 `RUN` 指令也是基于该用户运行。这个很容易理解，比如 `RUN mkdir /xxx` 自然是希望这个目录在容器启动后能被同一用户访问。
- 以非 root 用户运行业务应用是标准做法，但在 [OpenShift](openshift-tldr.md) 环境，是要求以十位数的**随机 UID** 运行、无权限问题，因此如以上设置 `USER` 也没意义。
  - 随机 UID 是在容器启动时自动添加到 `/etc/passwd`。
  - 而该用户属于 `root` 组，因此要使 OpenShift 的容器应用正常启动，只要 `root` 组用户无权限问题就行；但用户通常不需要直接操心这件事，保证基镜像使用的是 [Unprivileged Image](container-run-tldr.md#unprivileged-image) 即可。
- 通过配置 Deployment / Pod 的 [securityContext](https://v1-32.docs.kubernetes.io/docs/tasks/configure-pod-container/security-context/#set-the-security-context-for-a-pod) 可以指定 `runAsUser`，但在 Kubernetes 和 OpenShift 的默认行为不一样：
  - 如果未指定：Kubernetes 使用默认的 `spring` 用户、OpenShift 为随机生成用户。
  - 如果指定为默认用户的 UID：Kubernetes 没问题，但非集群管理员提交到 OpenShift 时会被禁止，注意这里有两类用户，一个是 OpenShift 平台用户、一个是容器运行用户。

另外如以上 Container Run TLDR 提及，我们在真实场景更推荐采用无 Dockerfile 的 Jib 方式，但 Jib 是不支持 `RUN` 指令的。这种情况要么根据实际应用场景排除 `RUN` 的需求，比如之上不是必须添加 `spring` 这个用户；要么先使用一个包含 `RUN` 的最简 Dockerfile 构建镜像，并作为 Jib 的基镜像……这个话题本文就不展开了。

### ARG

```
ARG DEPENDENCY=target/dependency
```

[ARG](https://docs.docker.com/reference/dockerfile/#arg) 定义变量及默认值，可以在实际构建镜像时通过命令行参数替换变量值：

- 引用变量的语法为 `$VAR` 或 `${VAR}`，以下 `COPY` 指令中有实例。
- `target` 目录 Java 开发人员都很熟悉，但 `target/dependency` 又是什么？Spring Boot with Docker 文档也说明了由来：
  ```
  mkdir -p target/dependency && (cd target/dependency; jar -xf ../*.jar)
  ```
  但是注意随着 Spring Boot 版本升级，该做法也有调整，比如 [3.0](https://docs.spring.io/spring-boot/docs/3.0.0/reference/html/container-images.html#container-images.dockerfiles)：
  ```
  java -Djarmode=layertools -jar application.jar extract
  ```
  而到 [3.4](https://docs.spring.io/spring-boot/3.4/reference/packaging/container-images/dockerfiles.html) 又有了变化：
  ```
  java -Djarmode=tools -jar application.jar extract --layers --destination extracted
  ```
  除了第一种（`jar -xf`），新做法都不是简单的解压，下一节有详细说明。
- 如果使用的不是 Maven 而是 Gradle，则对应的目录是 `build/dependency`，可以调整以上默认值或者构建时通过参数调整。
- 如之上 FROM 一节所述，基镜像应该使用固定 Tag / 精确版本，因此我们还考虑过是不是能将 Tag 也通过 `ARG` 传递，传递的目的是保证 CI 编译的 JDK 和容器运行的 JRE 版本完全一致、且不用重复配置；但 Dockerfile 并不支持，因为"a valid Dockerfile must start with a [FROM](https://docs.docker.com/reference/dockerfile/#from) instruction"，不过我们也可以在 CI 中动态生成 Dockerfile……

### COPY

```
COPY ${DEPENDENCY}/BOOT-INF/lib /app/lib
COPY ${DEPENDENCY}/META-INF /app/META-INF
COPY ${DEPENDENCY}/BOOT-INF/classes /app
```

[COPY](https://docs.docker.com/reference/dockerfile/#copy) 和 `RUN` 是用于在基镜像上添加内容、制作用户定制镜像的主要手段，后者通常是运行一个安装命令、当然也可以是任意可用命令或脚本：

- 和 `COPY` 类似的还有 [ADD](https://docs.docker.com/reference/dockerfile/#add) 指令，[ADD or COPY](https://docs.docker.com/build/building/best-practices/#add-or-copy) 解释了其中的差异，更多用的还是 `COPY`。
- `COPY` 和 `RUN cp` 是不同的，前者是从外部（比如 CI 运行环境）拷贝内容进来，而后者只能拷贝容器内部的文件。

由于该指令的语义很清晰，本节主要解释为什么不是直接拷贝一个 [Executable Jar](https://docs.spring.io/spring-boot/3.4/specification/executable-jar/index.html) 文件、而是先在上一节解压 / 提取然后在此多行拷贝？

- 容器的一个重要特色就是其[分层重用](container-tldr.md#general)的设计机制，比如两个 Java 应用使用了同一个基镜像，当这两个容器应用在同一台宿主机上启动时，则只需下载一次基镜像，这显然可以加速启动并节约存储。
  - 这个共享的基镜像并不代表只有一层，它很可能也有自己的基镜像。
  - 而这个层的划分就是通过 `COPY` 等指令实现的，每个指令所变更的内容就代表增加了一层（即使删除内容也是增加一层）。
- 如文档中 [Example 2](https://spring.io/guides/gs/spring-boot-docker) 的做法，就是只拷贝了一个 Executable Jar 文件；那么之后即使只修改了一个标题的一个字符，重新打包的 Jar 文件的 Hash 值也完全不同，也就是说制作容器镜像时会替换为全新的一层，这显然有很大优化余地。
- 因此制作应用容器镜像时的一个**重要考量**就是区分出应用制品中易变和不易变的部分，并尽量重用不易变的内容。
  - 当一个 Java 应用基本成型以后，所依赖的第三方 Jar 包（对应以上的 `lib`）很少改变，主要动的是业务功能相关的程序（对应以上 `classes`），而通过以上解压 + 多层拷贝的方式，就可以做到新版镜像继续重用 `lib` 层、仅仅重建 `classes` 层，显然这不是多此一举。
  - 但不是说这种做法就不能动 `lib` 层，比如定期的版本升级，只是说 `lib` 的变更频率要远小于 `classes`，这样的收益也足够了。
  - 理论上还可以将 `lib` 划分为不易变和更不易变的部分，但这个就没必要了，因为远没有"第三方库 Vs. 应用"这样边界清晰。
  - 如何区分一个 Spring Boot 应用的易变和不易变部分，这个不需我们自己琢磨，参考官方文档 [Layering Docker Images](https://docs.spring.io/spring-boot/3.4/reference/packaging/container-images/efficient-images.html#packaging.container-images.efficient-images.layering) 即可。
- `COPY` 哪些层的内容是和构建镜像前的准备工作（解压 / 提取）相关的，因此用户不要直接拿 Spring Boot with Docker 文档的 Dockerfile 示例去用，务必参考和应用**相同版本**的 Spring Boot 官方 [Dockerfiles](https://docs.spring.io/spring-boot/reference/packaging/container-images/dockerfiles.html) 文档（进入该链接后在左上方调整版本）。
- 注意以上多个 `COPY` 的顺序不能调换，因为每层的唯一性不只依赖它本身的内容，"Layers are stacked sequentially, and each one is a [**delta**](https://docs.docker.com/build/concepts/dockerfile/#docker-images) representing the changes applied to the previous layer."。

### ENTRYPOINT

```
ENTRYPOINT ["java","-cp","app:app/lib/*","hello.Application"]
```

[ENTRYPOINT](https://docs.docker.com/reference/dockerfile/#entrypoint) 表示该容器镜像的默认启动命令：

- [CMD](https://docs.docker.com/reference/dockerfile/#cmd) 也是类似作用，[Difference Between run, cmd and entrypoint in a Dockerfile](https://www.baeldung.com/ops/dockerfile-run-cmd-entrypoint) 通过试验给出了很直观的说明。
- 当然镜像的默认启动命令也可以在实际启动容器时替换，但不同的运行时做法不一样：
  - [Docker](https://docs.docker.com/reference/cli/docker/container/run/)：`docker run --entrypoint [ENTRYPOINT ] IMAGE [COMMAND]`
  - [Kubernetes](https://kubernetes.io/docs/tasks/inject-data-application/define-command-argument-container/)："The `command` field corresponds to `ENTRYPOINT`, and the `args` field corresponds to `CMD` in some container runtimes."
- 在 [3.0](https://docs.spring.io/spring-boot/docs/3.0.0/reference/html/container-images.html#container-images.dockerfiles) 的配置调整成了：
  ```
  ENTRYPOINT ["java", "org.springframework.boot.loader.JarLauncher"]
  ```
  实际上之前也有这种做法，在[旧文档](https://github.com/spring-attic/top-spring-boot-docker/blob/main/README.adoc#spring-boot-layer-index)还解释了"The Spring Boot fat JarLauncher is extracted from the JAR into the image, so it can be used to start the application **without hard-coding the main application class**"，但不知为何现在又**改回**需要"hard-coding"的方式？包括更新的 [3.4](https://docs.spring.io/spring-boot/3.4/reference/packaging/container-images/dockerfiles.html)，初步搜索未找到官方说明，可能和性能有关？
- 从 [ARG](#arg) 一节讨论的准备工作，到 [COPY](#copy)，到本节，各节的做法都应**一致参考**和应用相同版本的 Spring Boot 官方 [Dockerfiles](https://docs.spring.io/spring-boot/reference/packaging/container-images/dockerfiles.html) 文档（进入该链接后在左上方调整版本）。

## More Instructions

除了以上最基本的容器镜像构建指令，再讨论一些感兴趣的指令。

### EXPOSE

[EXPOSE](https://docs.docker.com/reference/dockerfile/#expose) 是一个容易被初级用户误用的配置项，以为在这里指定了端口号，容器启动后就是通过这个端口暴露服务，但实际上：

> The `EXPOSE` instruction **doesn't** actually publish the port. It **functions as a type of documentation** between the person who builds the image and the person who runs the container, about which ports are intended to be published.

也就是说它只是一个申明，具体启用哪个端口完全取决于程序自身的配置，即使配错了也不影响程序启动，当然这会误导人。

### VOLUME

参见 [Storage](https://docs.docker.com/engine/storage/) 说明，在容器运行时生成或修改的文件数据是不会被持久化的，也就是说容器销毁后都会丢失，因此也提供了 [Volume mounts](https://docs.docker.com/engine/storage/#volume-mounts) 等机制，将容器内的某个或某些文件目录挂载 / 映射到所运行的宿主机的文件或目录上，实现容器数据的持久化。由于存储本身就是一个大课题，而且我们实际使用的 [Kubernetes](https://kubernetes.io/docs/concepts/storage/) 环境更复杂，因此我们不深入讨论，主要针对 Spring [旧文档](https://github.com/spring-attic/top-spring-boot-docker/blob/main/README.adoc#a-better-dockerfile)存在的以下语句做一些分析：

```
VOLUME /tmp
```

这句的意思是在容器运行时需要挂载容器内的 `/tmp` 目录，注意 [VOLUME](https://docs.docker.com/reference/dockerfile/#volume) 的提示：

> The host directory is declared at container run-time

这也很明显，Dockerfile 当然不可能预期到运行时的宿主机目录。我们在这里主要对比有无这句的区别，试验使用 Podman 进行：

1. 运行相同 `VOLUME` 配置的容器镜像并进入容器内：
   ```
   podman run --name demo -d gitlab-registry.example.com/openshift/observability-spring-boot-demo:latest-33461-c200909f
   ```
   ```
   podman exec -it demo sh
   ```
1. 在容器内生成以下两个文件，一个位于 VOLUME 所指定的目录内、一个在外：
   ```
   touch /tmp/this-file-in-dockerfile-volume-folder-xxxxxxxxx
   ```
   ```
   touch /this-file-not-in-dockerfile-volume-folder-xxxxxxxxx
   ```
1. 回到宿主机，搜索以上两个文件：
   ```
   # find / -name *xxxxxxxxx
   /var/lib/containers/storage/volumes/395083296b3acb554165ceaa69b9cae616815d36163457d2101613ef5e516b91/_data/this-file-in-dockerfile-volume-folder-xxxxxxxxx
   /var/lib/containers/storage/overlay/ace2d1117440d054cc55ccaffa7a4cf4dbe48fe942a066b4e93afe8730ad5bb6/diff/this-file-not-in-dockerfile-volume-folder-xxxxxxxxx
   /var/lib/containers/storage/overlay/ace2d1117440d054cc55ccaffa7a4cf4dbe48fe942a066b4e93afe8730ad5bb6/merged/this-file-not-in-dockerfile-volume-folder-xxxxxxxxx
   ```
   - `this-file-in...` 文件在 `volumes` 目录，具体的宿主机目录完全取决于 Podman 的设置或者用户在运行时指定。
   - `this-file-not-in...` 文件在 `overlay` 目录，这是 [top writable layer](https://docs.docker.com/engine/storage/drivers/#container-and-layers)，也就是说容器内所变更的文件其实也全部在宿主机落盘了，最终的差别见下一步。
1. 强行删除容器后再搜索以上两个文件，可以发现只有 `VOLUME` 文件保留下来了，这也是 Volume 机制的作用。
   ```
   # podman rm -f demo
   # find / -name *xxxxxxxxx
   /var/lib/containers/storage/volumes/395083296b3acb554165ceaa69b9cae616815d36163457d2101613ef5e516b91/_data/this-file-in-dockerfile-volume-folder-xxxxxxxxx
   ```

再回到这个设置本身，`/tmp` 从语义看都没必要永久保留，如果说有一点用处那就是该目录暂存了 Spring Boot 应用的 Tomcat 相关临时文件（`tomcat.8080.9262304975881525577`），容器重启会快一点？但如果是新建容器也利用不了，因为是[匿名](https://docs.docker.com/engine/storage/volumes/#named-and-anonymous-volumes)挂载、在 `volumes/.../_data` 之间的那一长串 ID 会变。考虑到每次新建都会在宿主机留下垃圾，倾向于取消这个配置，实际上 Spring 更新的文档已经删除了这一句。

但是没有 `VOLUME` 这个指令也不代表不能持久化，只要在运行时做了挂载；总的来讲它和 `EXPOSE` 类似更多是一个申明，但这也是有意义的，方便镜像使用者了解需要持久化的内容所在。

## Multi-Stage Build

Spring 的[早期做法](https://github.com/spring-attic/top-spring-boot-docker/blob/main/README.adoc#multi-stage-build)还提供了 [Multi-stage builds](https://docs.docker.com/build/building/multi-stage/) 方式：

```
FROM eclipse-temurin:17-jdk-alpine as build
WORKDIR /workspace/app

COPY mvnw .
COPY .mvn .mvn
COPY pom.xml .
COPY src src

RUN ./mvnw install -DskipTests
RUN mkdir -p target/dependency && (cd target/dependency; jar -xf ../*.jar)

FROM eclipse-temurin:17-jdk-alpine
# 大致同以上的 Example，下略……
```

总的来讲，在已经具备 [CI/CD](cicd-tldr.md) 平台的企业环境，我们的**建议**是主要工作放在 CI Job、而不是在 Dockerfile 的 `builder` stage；因为在构建容器镜像之前，我们需要验证提交的代码是否正确，不仅是编译不出错，还会运行测试并生成测试报告等等，只有在验证无误后才有必要生成镜像。显然 Dockerfile 不能也不应该涵盖 CI 的主要场景，所以之上是 `skipTests`；而且 CI 中同样也会执行 `./mvnw install`，直接利用这个 Job 的产出物构建容器镜像就好，没必要再执行一遍哪怕是 `skipTests`。

但这个问题可以考虑的**更复杂**。CI Job 和 Dockerfile `builder` stage 重合的工作主要是下载依赖和编译，但对解释型语言如 Python 是没有编译环节的；而下载依赖，由于不同的 CI Job 很可能运行在不同的容器，因此在 CI Test Job 所下载的依赖要传递给 CI Build-Image Job，通常也是先上传到如对象服务器再下载，如果这样，那后一个 Job 直接从制品服务器下载依赖区别也不大……

到 Spring Boot 3.4 的 Reference 文档[示例](https://docs.spring.io/spring-boot/3.4/reference/packaging/container-images/dockerfiles.html)，仍在使用 Multi-Stage Build 但实际有了重大调整：

```
# Perform the extraction in a separate builder container
FROM bellsoft/liberica-openjre-debian:24-cds AS builder
WORKDIR /builder
# This points to the built jar file in the target folder
# Adjust this to 'build/libs/*.jar' if you're using Gradle
ARG JAR_FILE=target/*.jar
# Copy the jar file to the working directory and rename it to application.jar
COPY ${JAR_FILE} application.jar
# Extract the jar file using an efficient layout
RUN java -Djarmode=tools -jar application.jar extract --layers --destination extracted

# This is the runtime container
FROM bellsoft/liberica-openjre-debian:24-cds
# 大致同以上的 Example，下略……
```

其中 `builder` stage 的 `jar file` 仍是从外部 `COPY` 来的，也就是说依赖前置工作典型如 CI Job，而它的主要工作只是如 [ARG](#arg) 的简单"解压"，按这种用法，以下两种方式差别倒不大：

- CI Job 负责下载依赖、编译测试及打包，Dockerfile 负责"解压"及构建镜像。
- CI Job 负责下载依赖、编译测试、打包及"解压"，Dockerfile 负构建镜像。

总之至少在企业内的 Java + CI 环境，我们建议不要或者非常有限的使用 Multi-Stage Build。
