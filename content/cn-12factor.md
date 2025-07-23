---
title: 云原生与十二因子
---

## 说明

### 云原生

按[云原生计算基金会](https://www.cncf.io/)的定义，云原生在技术上的表现是：

> 云原生的代表技术包括容器、服务网格、微服务、不可变基础设施和声明式API。

具体到我们的情况，简单说就是应用经过一定的改造后可以部署运行在我们的 [Kubernetes](k8s-tldr.md) / [OpenShift](openshift-tldr.md) 容器平台。但显然如[云原生改造](cn-transform.md)中所强调的，我们不是为了上云原生而云原生，我们期望达到的目标是：

> 结合可靠的自动化手段，云原生技术使工程师**能够轻松地对系统作出频繁和可预测的重大变更**。

如 DevOps、敏捷等一样，频繁和可预测代表着应用能够**持续、稳定、高速的进化**，而最终回馈的则是业务上的收益。但上述目标显然也不是云原生独占的，任何一个高水平、专业的 IT 应用都会有这样的追求，而所采取的手段或者说方法论也殊途同归，比如早在 [2011](https://en.m.wikipedia.org/wiki/Twelve-Factor_App_methodology) 年就提出的十二因子 / 十二要素，以及 2010 年的 [Continuous Delivery](continuous-delivery-notes.md)。

### 十二因子

十二因子是 Heroku 创始人为构建 SaaS 应用所提出的方法论指导，基于 Heroku 这个 PaaS / 公有云平台及其上数十万计应用的实践总结，其中的一个主要目的就是：

> 适合部署在**现代的云计算平台**

符合十二因子的云原生应用是适合部署在现代云平台的应用，但反过来，现在提供的云平台也应该是方便云原生应用部署和运维的，这貌似循环定义，但本来这两者就没有绝对的因果，专业的应用和专业的平台都会持续向对方提出更高的要求，**是两者的相互促进和磨合最终形成了现代的云**；而"现代"这个词代表的就是当前先进的和主流的，之前是 Twelve-Factor App 和 Heroku，到现在则是云原生应用和容器、Kubernetes，如 [Heroku](https://www.heroku.com/about) 现在所宣称的：

> Heroku is a container-based cloud Platform as a Service (PaaS).

而虽然十二因子这个词在业界特别是云计算领域多有提及，但具体到每个因子（to provide a **shared vocabulary** for discussing those problems），却并未如微服务、无状态这些术语有足够的辨识度，个人认为是各因子的命名相对不够直观、有些内容和当时 Heroku 平台的具体技术联系紧密、以及官方[中文](https://12factor.net/zh_cn/)翻译比较生硬等因素的影响。另外注意由于这些理念提出已有十多年之久，可能有些我们（或者部分人）觉得天经地义的做法在当时也是需要突出强调的。因此以下我们将对不够直观的部分做一些说明、并结合我们的实际情况和当前新技术的发展做相应补充，当然也不会完全照搬官方文档的详细内容（强烈建议阅读[原版英文](https://12factor.net/)）。

## 十二因子

### 基准代码

*一份基准代码，多份部署*

其中强调的重点：

- 使用版本控制系统管理应用代码。
- 一份基准代码可以等同为比如 Git 中的一个 Repo。
- 同一个应用不应该有多份基准代码或者说 Repo，而反之不同的应用也不应位于同一个 Repo。
- 因此，同一个应用针对不同环境的多份部署也应该源自同一 Repo，当然可以是不同的 Branch 或 Commit。

我们的补充及更新：

- 对于当前的微服务架构，显然一个微服务应该对应一份基准代码。
- 但是到现在这也不是必须遵守的准则了，[Monorepo](https://about.gitlab.com/direction/monorepos/) 同样是在业界有众多成功实践的方式：
  > Monolithic repositories, more commonly known as monorepos, are a software development approach where code for **many projects is stored in a single repository**.
  >
  > Monorepos provide a number of advantages such as reduced complexity, code reuse, easier collaboration amongst teams, and streamlined dependency management. Additionally, there are several drawbacks such as ......
- 原文提及了 SVN，但就像现在有人会问除了 GitOps 外还有没有 SvnOps？理论上可以有，但实际新的工具几乎不会考虑对 SVN 的支持。
  - 目前我们还有大量代码使用 SVN 管理，应该考虑迁移 Git 了。

### 依赖

*显式声明依赖关系*

其中强调的重点：

- 通过依赖清单，确切地声明所有依赖项。
- 通过依赖隔离工具来确保程序不会调用系统中存在但清单中未声明的依赖项。
- 12-Factor 应用同样不会隐式依赖某些系统工具。如果应用必须使用到某些系统工具，那么这些工具应该被包含在应用之中。

我们的补充及更新：

- 以上主要强调的是运行时依赖，实际上还有设计时（开发阶段）依赖，以下说明。
- 关于 Java 应用：
  - 当前基于 Maven / Gradle 管理 Java 程序依赖已是一个标准做法。
  - 在 Java 应用中 classpath 起到了依赖隔离工具的作用，参见 [Is there anything like VirtualEnv for Java](https://stackoverflow.com/questions/7300148/is-there-anything-like-virtualenv-for-java)。
  - 但 Maven 依赖并不会包含 Java 本身，也就是说不会明确指定应用所运行的 Java 版本。
    - 对于设计时，可以通过 [Maven Enforcer](cn-transform.md#maven-enforcer) 解决。
    - 运行时获取 Java 版本有简单方法，但是应用自己检查版本并在不匹配时 Fail Fast 的做法并不常见，通常是在部署时检查。
  - 如果说以上重点中的前两条已基本做到的话，我们很多应用忽视了工具依赖这一条。
    - Maven 应该算开发工具而非系统工具，但是使用 [Maven Wrapper](cn-transform.md#maven-wrapper) 方式跟代码走已是主流做法。
- 关于 NodeJS 应用：
  - 当前基于 NPM / Yarn 管理 Vue.js 等前端程序依赖已是一个标准做法。
  - 我们目前主要是将前端应用部署为静态文件服务，因此也没有依赖隔离的问题。
  - 目前 NPM 没有类似 Maven Wrapper 的做法，但至少在 CI 中是可以明确指定 NodeJS 等的版本。
  - 注意使用 NPM 指定依赖软件的版本范围而非精确的版本有一定的方便之处，但必须使用 [package-locks](https://docs.npmjs.com/cli/v6/configuring-npm/package-locks) 以避免自动升级可能带来的破坏性影响。这个我们遇到过实际案例，可以说这种做法违背了显式依赖、但通过版本锁定机制解决了，或者说版本锁定机制就是显示依赖。
- 如以上 Java 一段所述，目前语言级别的依赖管理不会或者说很难涉及到更下层的依赖管理，但是使用容器时可以在 [Dockerfile](container-run-tldr.md#dockerfile) 明确指定基镜像的 Java 版本等，彻底避免了 [Dependency hell](container-tldr.md#general)，可以说容器是解决依赖问题的终极手段。
  - 如以上 NodeJS 一段提示的，Dockerfile 中的基镜像应该指定精确版本，因为它没有版本锁定机制。
- 并不是只有应用开发需要注意依赖，运维脚本同样如此，比如我们在使用 Ansible 部署可观测性平台时显式定义了 [*prerequisite*](why-here.md) 任务（环境依赖、工具依赖）。

### 配置

*在环境中存储配置*

其中强调的重点：

- 代码和配置严格分离，但注意并不是指所有配置，这里定义的"配置"主要是指在不同部署（预发布、生产环境、开发环境等等）间会有很大差异的情况。
- 代码不应暴露任何敏感信息。
- 推荐将应用的配置存储于环境变量中。
  - 与一些传统的解决配置问题的机制（比如 Java 的属性配置文件）相比，环境变量与语言和系统无关。
  - 配置文件的问题是分散在不同的目录、有着不同的格式，而这些格式通常是语言或框架特定的。
- 不建议将配置按照特定部署进行分组（或叫做"环境"），这种方法无法轻易扩展。

我们的补充及更新：

- 是否分离还是要看配置的具体性质而非教条的遵守，一些共性的即使只是某类环境下共性的、非敏感性配置，放在代码中也没有问题甚至更方便。
  - 使用集中配置管理如 Spring Cloud Config Server 后，一种做法是将所有的配置全挪了过去，但更合适的还是只将以上定义的那种"配置"放过去，而本地开发环境更没必要。
- 即使代码本身有访问权限控制，也**绝对不要**将敏感信息放入代码。
  - 即使是测试环境的数据库密码等也应该从代码分离，既保证和生产环境一致的处理模式、也养成良好习惯。
  - 唯一例外是本地开发环境、只能在本地访问的情况。
  - 虽然不能将敏感信息本身放入代码，但是关于敏感配置项的信息还是应该用代码管理的，比如我们可能会配置一个 secrets.env 文件将所有的敏感项批量 export 到环境，但显然这个文件不能跟代码走，**必须**加入到 gitignore，但是我们可以将其拷贝并脱敏为 secrets.env.example 文件，这样其他人就能直观的了解到有多少敏感项及命名。但这只是一个小技巧，更应该用如下的正规方式。
  - 虽然集中配置管理如 Spring Cloud Config Server 将配置信息从源码中分离出来，但并未针对敏感信息做专门或者说更安全的处理；而 CI / CD 工具如 GitLab 提供了敏感信息的保存和使用机制，包括权限控制、日志脱敏等，但最终应该落到 [Vault](https://www.vaultproject.io/) 之类的方案。
- 按以上说明，相比配置文件更应该使用环境变量；但是当一个应用的配置足够复杂后，只使用环境变量很不直观（包括 Layout 以及比如数组性质的配置等），所以在实际情况中通常是这两种包括更多配置方式结合使用，而不是非此即彼。现代的开发框架或产品自然考虑到了这些情况：
  - Spring Boot 的 [Externalized Configuration](https://docs.spring.io/spring-boot/docs/2.5.6/reference/htmlsingle/#features.external-config) 机制约定了不同配置来源的优先级、以及环境变量名和配置项的映射规则（[Binding from Environment Variables](https://docs.spring.io/spring-boot/docs/2.5.6/reference/htmlsingle/#features.external-config.typesafe-configuration-properties.relaxed-binding.environment-variables)）等。
  - [Promtail](https://grafana.com/docs/loki/v2.3.0/clients/promtail/configuration/#use-environment-variables-in-the-configuration)：You can use environment variable references in the configuration file to set values that need to be configurable during deployment.
- 很显然配置分组也不是一个禁忌，典型如 [Spring Profiles](https://docs.spring.io/spring-boot/docs/2.5.6/reference/htmlsingle/#features.profiles) provide a way to segregate parts of your application configuration and make it be available only in certain environments。
  - 该因子的文中所举反例如"开发人员可能还会添加他们自己的环境，比如 joes-staging"，至少我们目前是要求所有开发人员的本地开发环境保持一致的配置的，而即使将来可能会变也不是现在不用的理由。
  - 环境的划分也有不同的维度，开发或生产是一类，部署到 VM 或部署到 Kubernetes 又是另一类，因此按某个维度聚合一些配置项也就是分组，实际上是方便使用和理解的。
  - 环境变量名的前缀实质就是一种隐含的分组。

### 后端服务

*把后端服务当作附加资源*

其中强调的重点：

- 后端服务是指程序运行所需要的通过网络调用的各种服务，如数据库、消息 / 队列、缓存等。
- 应用不会区别对待本地或第三方服务，也就是说可以在不进行任何代码改动的情况下灵活切换。注意这里的本地实际是指企业内部自主管理的服务。
- 每个不同的后端服务是一份资源，这些资源和它们附属的部署保持松耦合。

我们的补充及更新：

- 相比之上依赖一节可以认为是应用的内部依赖，那么后端服务就是不用和应用部署在一起、需要通过网络访问的外部依赖。
  - 对于外部依赖，我们一般会分为中间件依赖和其他业务应用的依赖，而两者的差别在于前者的 API 通常是标准、稳定的，因此虽然技术上或者说运行时没差别，但是对开发质量特别是自动化测试还是有很大影响。
- 不太理解"应用不会区别对待本地或第三方服务"这句话的历史背景，通过网络访问本地或第三方服务本来就没有区别，但是以文中"将本地 MySQL 数据库换成第三方服务（例如 Amazon RDS）"为例，如果 Amazon RDS 是 [Microsoft SQL Server](https://aws.amazon.com/rds/sqlserver/)，也不可能"不进行任何代码改动"就切换，最起码也要添加 [MS SQL Server Driver](https://start.spring.io/#!type=maven-project&language=java&platformVersion=2.5.6&packaging=jar&jvmVersion=11&groupId=com.example&artifactId=demo&name=demo&description=Demo%20project%20for%20Spring%20Boot&packageName=com.example.demo&dependencies=sqlserver)（或者可能当时 Amazon 只支持 MySQL）。
- 所以个人认为这一点所强调的是尽可能将所有的服务都当作这种资源模式使用、尽量和应用解耦。
  - 对我们来说一个需要重点关注的服务就是[对象存储](cn-transform.md#对象存储)；如果应用的多个实例通过 NFS 等共享网络目录文件，而不是利用对象存储服务，那么在迁移到我们 PaaS 平台时会困难很多。

### 构建，发布，运行

*严格分离构建和运行*

其中强调的重点：

- 基准代码转化为一份部署需要以下三个阶段：
  - 构建阶段是指将代码仓库转化为可执行包的过程。
  - 发布阶段会将构建的结果和当前部署所需配置相结合，随时可以运行。
  - 运行阶段（或者说"运行时"）是指针对选定的发布版本，启动应用。
- 严格区分构建，发布，运行这三个步骤。举例来说，直接修改处于运行状态的代码是非常不可取的做法。
- 能够退回到之前的发布版本。

我们的补充及更新：

- 我们遇到的难以想象但并不罕见的做法是直接将本地编译生成的 class 文件手工替换到生产环境，已经不敢从源码生成完整的运行包了（历史问题，不清楚最近是否还有）。说实话很多人特别是 IT 新人会奇怪为什么还要专门强调"直接修改处于运行状态的代码是非常不可取的做法"：怎么会有人这样做？本来就不应该这样做啊？由于十二因子针对的是当时的众多弊端，但随着历史发展也消失了大半，如果只接触过正规做法有时确实难以理解这些因子到底在强调什么。
- 关于构建、发布、运行这三个步骤，目前的主流做法是 CI 和 CD，可以简单理解为 CI 对应构建、CD 对应发布 + 运行。个人认为不需要要纠结这些词的严格定义，本节的重点是不要犯上面的错误。
- 关于回退：
  - 回退并不是真的消除历史，实际仍然是一个新部署，只不过状态和想要回退的那个点相同而已。
  - 如果仅仅是程序本身回退还比较简单，但如果涉及到数据库变更等就会复杂很多。
  - 要注意源码版本和部署版本是两回事，回退应该是按部署版本或者说部署历史记录、而非源码版本进行，现代工具如 GitLab 支持这种做法。

### 进程

*以一个或多个无状态进程运行应用*

其中强调的重点：

- 应用的进程必须无状态且无共享，任何需要持久化的数据都要存储在后端服务内。

我们的补充及更新：

- 如果以"无状态"取代"进程"这个关键词应该直观很多，现在的技术交流中几乎不用解释。
- 这一条的好处体现在以下并发一节，不太理解为何分成两个因子。
- 以无状态进程运行应用，而有状态的部分则变为以上提及的后端服务。这种做法并没有消除有状态部分的复杂性，但是划分清晰而非混在一起，就可以针对性的治理提升。
- 目前我们 PaaS 平台仅支持无状态应用。

### 端口绑定

*通过端口绑定来提供服务*

其中强调的重点：

- Web 应用本身就是一个 Web Server 并通过网络端口暴露服务。
- 端口绑定也意味着一个应用可以成为另外一个应用的后端服务。

我们的补充及更新：

- 这也是一个绕口的关键词，直接说就是 Web 应用内置 Web 容器，这样就简化了部署同时也降低了对环境的要求和依赖。
- 这已是 Spring Boot 的默认方式，即由源码生成的是一个内置 Tomcat 的 Fatjar，可以直接运行在 JVM，而不需要象 War 包那样要求部署环境具备 Tomcat 或 WebSphere 等。

### 并发

*通过进程模型进行扩展*

其中强调的重点：

- 应用进程所具备的无共享、水平分区的特性意味着添加并发会变得简单而稳妥。
- 应用进程不需要守护进程或写入 PID 文件，相反应该借助操作系统的进程管理器，来管理输出流、响应崩溃的进程、以及处理用户触发的重启等请求。

我们的补充及更新：

- 这一点偏底层，从业务应用开发人员的角度应该考虑的是产品或者开发框架的选型、而非直接实现进程模型。
- 目的是能够方便的进行水平扩展，当然前提是之上进程一节的无状态应用。
- 但真要方便、快速的扩展，还需要依赖于运行平台，目前这个角色 Kubernetes 当仁不让。
  - 而以上提及的"管理输出流、响应崩溃的进程"等等也完全由 Kubernetes 负责了。

### 易处理

*快速启动和优雅终止可最大化健壮性*

其中强调的重点：

- 可以瞬间开启或停止，这有利于快速、弹性的伸缩应用，迅速部署变化的代码或配置。
- Web 服务的优雅终止是指在拒绝新请求的同时能够继续执行完当前已接收的请求，之后才退出。
  - 前提条件是 HTTP 请求大多不应超过几秒钟。
- [后台服务](https://12factor.net/zh_cn/concurrency)的优雅终止是指将当前任务退回队列。
  - 这要求任务是可重复执行的，可以通过事务或者幂等来实现。

我们的补充及更新：

- 如上所述，对应用本身的品质要求较高，而现在上 PaaS 平台的部分应用觉得默认的 30 秒超时都不够用，而大部分应用基本也未遵循 REST 的语义如幂等。
- 容器时代的 [Java](container-tldr.md#java-in-container) 比较尴尬，主要就是不够轻快，该链接有详细讨论，总之结果就是很难快速启停。
  - 如果只考虑技术因素，Go 应该是我们去尝试的开发语言。
- Spring Boot 在 2.3 即支持 [Graceful shutdown](https://docs.spring.io/spring-boot/docs/2.3.0.RELEASE/reference/htmlsingle/#boot-features-graceful-shutdown)。

### 开发环境与线上环境等价

*尽可能的保持开发，预发布，线上环境相同*

其中强调的重点：

- 想要做到持续部署就必须缩小本地与线上差异:
  - 缩小时间差异：开发人员可以几小时，甚至几分钟就部署代码。
  - 缩小人员差异：开发人员不只要编写代码，更应该密切参与部署过程以及代码在线上的表现。
  - 缩小工具差异：尽量保证开发环境以及线上环境的一致性。
- 反对在不同环境间使用不同的后端服务，即使适配器已经可以几乎消除使用上的差异。
  - 不同的后端服务意味着会突然出现的不兼容，即使考虑适配器代价也比想象中要大。

我们的补充及更新：

- 缩小时间差异自然是通过自动化的 CI / CD。
- 缩小人员差异也是当前 [DevOps](devops-talk.md) 实践所强调的重点。
- 保持各环境尽可能一致本就是运维一直强调的要求，目的都是为了尽早发现生产环境可能出现的问题。
  - 但是开发环境例外，比如使用 [Embedded Database](cn-transform.md#dependency-database) 模拟生产环境的数据库。
    - 而到容器时代，连开发环境都不用排除在外了，但是仍有成本：
      - 由于我们桌面机主要使用 Windows、而容器主要基于 Linux，对开发人员机器硬件配置自然要求很高，再考虑到外包用户本就是虚拟机更不可能。
      - Kubernetes 给每个开发人员提供独立的远程容器资源，但一者 PaaS 运维方尚无余力考虑这个需求，二者这种做法的配套工具貌似也未成熟，比如通过 Maven 插件启停远程容器等。
      - Web IDE 或 Cloud IDE 应该也是一个方向，但以 [GitLab Web IDE](https://docs.gitlab.com/user/project/web_ide/) 为例，"Generally available in GitLab 18.0"（2025-06 最新大版本），而且 GA 离真正普及仍差得很远。
  - 从 CI 环境开始就必须和生产环境尽可能保持一致了。
    - 但是也不可能完全相同，比如生产用 F5 硬件，CI 通常不会有 LB 而是直接访问应用、最多也是如 Nginx 之类的软 LB；而我们实际遇到过因为 LB 不同导致的在测试未发现问题但生产出问题的情况。

### 日志

*把日志当作事件流*

其中强调的重点：

- 日志应该是事件流的汇总，将进程的输出流按照时间顺序收集起来。
- 应用本身不用考虑存储自己的输出流，不应该试图去写或者管理日志文件。
- 进程的输出流由运行环境截获和处理。
- 最重要的，输出流可以发送到日志索引及分析系统。

我们的补充及更新：

- 虽然对日志文件的管理比如 Rotate 是由框架实现（实际也是框架所依赖的第三方库），开发人员几乎没什么工作量，但是给框架减负也是有意义的，目前无论 Java 还是 Spring 要考虑的东西太多，不管是否有历史合理性最终结果都是现在的 Java 应用太笨重，从这一点说确实谈不上"云原生"。
- 应用上生产后收集日志集中保存和利用这已是当前的标准做法，但是日志本身还存在很大的治理空间。
  - 尽量使用结构化的日志输出如 JSON、logfmt，而不依赖后续日志收集时的处理，在源头实现更方便准确。
  - 很多应用输出的日志太不讲究，比如应该是 Debug level 的日志通通用 Info 输出、不需要记录 Stack Trace 的 Exception（通常是 Business Exception，完全不需要分析技术上的调用栈）也大量输出、以及醒目的"=================="等等；如果将日志集中平台比作一个广场，无论广场本身的硬件条件多好，也经不住广场上的人乱扔垃圾。相应的开发规范此处不再展开，可以参考阿里巴巴 Java 开发手册中的日志规约（但注意有部分内容已不适合云原生）等，但是无论定了多少规范，具体的操作仍然是依赖开发人员自己的专业判断，我们可以规定不要把所有的内容都用 Info level 输出，但是具体到哪些内容使用 Debug 则只能是应用团队自己判断了。

### 管理进程

*后台管理任务当作一次性进程运行*

其中强调的重点：

- 管理或维护应用的一次性任务的代码和应用程序本身的代码应该一起管理。

我们的补充及更新：

- 目前很多应用并未将数据库 SQL 文件（包括 Schema、基础数据、示例数据等）和程序代码一起管理。
- Spring Boot 同样集成了和 `rake db:migrate` 类似的工具，参见 [Use a Higher-level Database Migration Tool](https://docs.spring.io/spring-boot/docs/2.5.6/reference/htmlsingle/#howto.data-initialization.migration-tool)。

## 小结

整理一下我们因应十二因子而需要考虑的一些规范（更详细的说明见以上每个因子）：

1. 基准代码
   - 还在 SVN 上的项目应该迁移到 Git。
2. 依赖
   - 使用 [Maven Wrapper](cn-transform.md#maven-wrapper)。
   - 使用 [Maven Enforcer](cn-transform.md#maven-enforcer)。
   - NPM 使用 [package-locks](https://docs.npmjs.com/cli/v6/configuring-npm/package-locks)。
   - 使用容器彻底解决运行时依赖，但注意容器基镜像应该使用精确版本。
3. 配置
   - 绝对不要将敏感信息放入代码，即使是测试环境，即使代码本身有权限控制。
   - 小技巧：对于有可能不小心提交到代码库的敏感文件，提前 Git Ignore。
   - 使用专业工具如 Vault 更完善的解决敏感信息的配置管理。
   - 用足 Spring Boot 的 [Externalized Configuration](https://docs.spring.io/spring-boot/docs/2.5.6/reference/htmlsingle/#features.external-config) 机制。
4. 后端服务
   - 使用[对象存储](cn-transform.md#技术债-对象存储)取代 NFS 等网络共享存储。
5. 构建，发布，运行
   - 未上 CI / CD 的项目应该要考虑使用了，无论 Jenkins 还是 GitLab CI。
6. 进程
   - 应用本身尽量无状态化，典型如第 4 条的本地存储应转后端服务。
7. 端口绑定
   - Spring Boot 部署使用默认的 [Executable Jar](https://docs.spring.io/spring-boot/3.5/tutorial/first-application/index.html#getting-started.first-application.executable-jar) 模式。
8. 并发
   - 使用 [Kubernetes](k8s-tldr.md) 实现自动部署、负载均衡、自我修复等等等等。
9. 易处理
   - 应用自身的性能调优，即使由于上游原因也应该考虑缓存、异步等方式尽量缓解。
   - 遵循 REST API 语义。
   - Spring Boot 升级至 3 或更高实现 Graceful shutdown。
   - 技术预研及储备应该考虑一下 Go 了。
10. 开发环境与线上环境等价
    - [DevOps](devops-talk.md)。 
    - 使用容器统一部署环境。
11. 日志
    - 应用需要输出结构化的日志，如 JSON、logfmt 等。
    - 日志输出需要有节制、分 Level。
12. 管理进程
    - 数据库 SQL 文件应该和程序代码一起管理，并能够自动化加载。
    - [Use a Higher-level Database Migration Tool](https://docs.spring.io/spring-boot/how-to/data-initialization.html#howto.data-initialization.migration-tool)。

从以上总结可以看出，其实我们勿需逐条追求十二因子的做法，只要做到以下：

- 技术升级：使用 Git、使用 CI / CD + DevOps、使用 Spring Boot 而非 Spring Framework、使用最新版的 Spring Boot、使用对象存储、使用容器、使用 Kubernetes 等等。
- 用好用足工具或框架：使用 Maven Wrapper / Enforcer、使用版本锁定机制、理解透彻 Spring Boot 的 Externalized Configuration、使用 Executable Jar、使用 Database Migration Tool、遵循 REST API 语义、输出 JSON / logfmt 格式日志等等。

也就是说**只要遵循业界的主流实践、把每个细节都做到位，实际上就已经达到了无论十二因子、Continuous Delivery 或者云原生所提及的大部分要求，自然就会有一个高水平、专业的产出，因此十二因子对我们最大的意义是理解它的思想，而非教条的遵守十年前的规矩**。

## 参考

- [Cloud Native Definition](https://github.com/cncf/toc/blob/master/DEFINITION.md)
- [The Twelve-Factor App](https://12factor.net/zh_cn/)
- [12 Factor Design Methodology and Cloud-Native Applications](https://www.cuelogic.com/blog/12-factor-design-methodology-and-cloud-native-applications)
- [Twelve-Factor Methodology in a Spring Boot Microservice](https://www.baeldung.com/spring-boot-12-factor)
- [浅析云原生12要素](https://zhuanlan.zhihu.com/p/243404169)
- [MRA, Part 5: Adapting the Twelve-Factor App for Microservices](https://www.nginx.com/blog/microservices-reference-architecture-nginx-twelve-factor-app/)
- [The 12-Factor App Methodology Explained](https://www.bmc.com/blogs/twelve-factor-app/)
- [12 Factor App Principles and Cloud-Native Microservices](https://dzone.com/articles/12-factor-app-principles-and-cloud-native-microser)
- [Concise Guide to Twelve-Factor App Methodology](https://www.solutionanalysts.com/blog/twelve-factor-app-methodology/)
- [Revisiting the Twelve-Factor App Methodology](https://codersociety.com/blog/articles/twelve-factor-app-methodology)
- [What is Twelve-Factor App?](https://www.geeksforgeeks.org/what-is-twelve-factor-app/)
