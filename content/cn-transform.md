---
title: 云原生改造
---

## 背景说明

[云原生](cn-tldr.md)改造的基本技术动作包括：

1. 创建 Dockerfile 用于构建应用的容器镜像。
1. 创建 Helm Chart 用于部署到 Kubernetes / OpenShift。
1. 创建 CI/CD 流水线用于自动化以上工作。

但**更重要**的是这个改造的前置工作或者说值不值得改造的前提条件。2010 年出版的 [Continuous Delivery](continuous-delivery-notes.md) 讨论了应用需具备的良好架构以及配置管理、自动化测试和持续集成等最佳实践（**强烈建议**阅读该链接和完整的书籍内容），而如果一个应用能够达到这些标准，基本上也就是云原生所[追求](https://github.com/cncf/toc/blob/main/DEFINITION.md#中文版本)的"容错性好、易于管理和便于观察的松耦合系统……能够轻松地对系统作出频繁和可预测的重大变更"；再需要做的仅仅是将其中过时的技术或产品替换为容器、Kubernetes 而已，最终你会发现这是最容易的部分。

这里解释一下为什么说最容易？以 Kubernetes 为例，很多人的感觉是概念太多了、很复杂很难，但实际上 [Kubernetes](k8s-tldr.md) 不是一个全新的东西，里面的概念、做法很多都是业界并不陌生的最佳实践，可能换了名字或形式，所以最困难的还是持续的跟进、学习了解和抓住本质，**所有的难都是在偿技术债、在补课**。

总之很遗憾，Continuous Delivery 十五年前的标准仍然远超现在绝大部分传统企业 IT 的水平，拿其中一项说明：

> Prerequisites for Continuous Integration
>
> In fact, **one sign of a good application architecture** is that it allows the application to be run without much trouble on a development machine.

以 Java 项目为例，当一个新成员拿到新的开发电脑，理想状况是装完最基本的软件如 JDK、Git 并克隆代码后就能够一键运行起来，但现实往往相反：

- 还需要安装 Maven 并配置使用内部镜像。
- 开发团队会共享同一个数据库，因此还需要申请到该机器的网络访问权限。
- 也包括共享其他中间件如缓存、MQ，或者 Spring Cloud Config Server、Eureka / Nacos 等等。
- 如果依赖自己团队的其他应用模块，则需要在本地启动或者共享。
- 如果还依赖其他团队的应用，则需要能访问应用的 UAT 环境。

如果不觉得以上工作有"much trouble"，下面会讨论损失了什么；而这还只是我们作为云原生推广团队刚接触待改造项目就碰到的问题，我的时间是解决对方不熟悉的新领域问题、或者疑难问题，而不是把代码跑起来都吃力……

在下面的环节，我们以真实项目的改造经历，给大家说明我们在传统应用的云原生改造中具体做了什么、优先做了什么。但能较完整体现以下的工作、且可以公开的真实项目 [*baseplatform*](why-here.md) 是早在 2021 年做的改造，有些内容不会过时，但有些经过四年可能就天翻地覆了；而近期（2025-05）改造虽然使用了更新的方式，但只是蜻蜓点水。总之虽然真实项目更能反应问题，但毕竟我们不可能一直伴随，主要就是一个启动作用，**真正的效果仍然取决于应用团队自身、取决于长期主义**。

## 专业化改造

之前这一章节的标题叫"前置工作"，但如[云原生漫谈](cn-tldr.md)强调的，所谓云原生就是现代的、当前先进的，而容器、Kubernetes 只是具体落地甚至不算必需的技术概念；因此**解决上面的问题、以及更多已经习以为常的异常，这与其说是"云原生改造"的前置工作，不如说这就是真正应该去做的、专业的云原生改造**！所以我们将云原生改造分为本章专业化改造和下一章容器化改造两部分。而如果在之下讨论的很多工作，你都觉得"本来就应该这样啊、不都是这样在做么"，那么**恭喜**，这最低也超过了 90% 企业项目的水平。

由于没有时间和精力持续更新，而且读者需要的也不只是最新开发框架的做法，即使局限在 Java 项目，不同大版本如 Spring Boot 2 Vs. 3 甚至中版本的特性添加，不同的技术选项比如 Eureka Vs Nacos，这些都决定了本文不可能是云原生改造的具体技术指导；所以本文大幅删减了旧版中容易过时的技术实操内容，而读者要**重点关注的是应该做什么、而不是具体怎么做**，然后你就会发现，2010 年的 [Continuous Delivery](continuous-delivery-notes.md)，2011 年的[十二因子](cn-12factor.md)，几乎覆盖了**每一点**。

### README.md

- `md` 是 Markdown 文件的扩展名，[Markdown TLDR](markdown-tldr.md) 详细讨论了 Markdown 是技术文档的主流选项、IT 业者几乎可以零学习成本的采用。
- README 是一个代码库的标配，在 GitLab、GitHub 等网站上作为项目的默认首页。
  - 这是一个新成员或者团队外合作者了解该项目实操的入口，因此**不要忽略**。
  - 虽然多说一句"去看某处的某文档"也能解决这个问题，但少这个过程是有意义的。
- 至少包含如何在本地开发环境运行应用、以及初步验证应用运行无误的步骤。
- 提供命令行说明，这是最简洁直观、不容易误解的方式。
  - 命令使用 Markdown 的[代码块](markdown-run-tldr.md#code)引用，方便用户在 Web 页面一键拷贝。
  - 这也是为后续的 CI 做准备，如 Continuous Delivery 所述：
    > it must be possible for either a person or a computer to run your build, test, and deployment process **in an automated fashion via the command line**. 
- 随着项目进展，不论增加代码还是修改之前的代码或任何变更，如有必要需**同步** README。
  - 因此将文档放在代码库外是不合适的，应该和代码保持版本同步。
  - **持续的同步更新决定了是真 README 还是假 README**，我们接触过的项目基本都有 README，但团队成员也基本都视若无睹。
- 除了 README 当然还会有更多文档，道理同上。

### Maven Wrapper

- [Maven Wrapper](https://github.com/takari/maven-wrapper)：
  > it's a great idea borrowed from Gradle
- [The Gradle Wrapper](https://docs.gradle.org/current/userguide/gradle_wrapper.html)：
  > developers can get up and running with a Gradle project quickly **without having to follow manual installation processes** saving your company time and money
- Wrapper 已经是通过 [Spring Initializr](https://start.spring.io/) 生成脚手架应用的**标准配置**。

不需要手工安装的原因是 Maven Wrapper 的程序脚本都和项目代码保存在一起，只要将代码克隆到本地，自然就能直接执行 Maven Wrapper 的命令。Maven Wrapper 会首先检查本地缓存是否有 Maven 程序，如无则先下载，然后执行实际的 Maven 程序；而为什么不直接将 Maven 程序放在代码库，自然是体积远大于只有几十 K 的 Wrapper。

### 内部镜像

无论使用 Maven 还是 Maven Wrapper，在企业环境通常都不会直接从外网下载依赖库，而是通过内部搭建的镜像服务，因此通过以上 Spring Initializr 生成的脚手架也做不到一键运行，还需要配置内部镜像地址之类。以前这类工具的配置通常也是手工进行，并保存到软件安装目录或用户目录等等，当然也就不能自动同步到新成员机器上，而现在要做的是将这类配置也保存到代码库。所以这一节的**重点**不是如何配置镜像、而是所有的配置都应该跟代码走，无论是运行时应用的配置，还是设计时工具的配置，如 Continuous Delivery 所述：

> **Keep Absolutely Everything in Version Control**

以下很多内容都适合这个原则。另外注意，除了配置 Maven 下载第三方依赖的内部镜像，如果（也应该）使用的是 Maven Wrapper，还需要配置 Wrapper 下载 Maven 程序的内部镜像，这两个配置是分开的。

### Maven Enforcer

由于本地开发环境可能存在多个 JDK 版本，使用 Enforcer 插件检查 Maven 所使用的 JDK 版本、如果不符合项目要求则出错，具体配置参见官方文档 [Require Java Version](https://maven.apache.org/enforcer/enforcer-rules/requireJavaVersion.html)。

这就是 **Fail Fast**。好处不仅是提高效率，也可以给出更明确的根因提示，否则等到编译过程中由于不兼容而出错，新手不一定反应得过来，**能用工具解决的问题就不要依赖文档或打扰人**。

### 运行时依赖

如 Maven 第三方库这种设计时的静态依赖，已经通过相应技术栈的包管理工具做到了自动下载，因此这里讨论的是应用对于外部服务如数据库之类的运行时依赖。越多这种依赖，就代表越多的准备工作，因此在本地开发时应该尽量减少依赖，准确的说是**减少运行时依赖的手工准备工作**。

实际上本文开头的"共享"也是在解决这个问题，也确实减少了新成员的准备工作，但即使不考虑网络权限申请这种非技术的麻烦，它本质上是有极大缺陷的；最直接的影响就是自动化测试，你可以想想如果两个成员前后脚开始做自动化测试，第一步是初始化数据，前后脚初始化了同一个数据库……总之，解决依赖问题绝不仅是方便本地开发，对后续如 CI 有更深远的影响，因为"run without much trouble on a development machine"同样代表着"run without much trouble on a CI machine"。

但是**注意**到容器时代，解决问题有了另外的思路，详细讨论参见 Container Run TLDR 的 [Development Environment](container-run-tldr.md#development-environment) 一节；但一者我们在使用上仍有限制，二者以下部分解决方法更简洁、并不过时，总之仍是值得我们了解的，特别是"应该做什么"。

#### Spring Cloud

注意当 Spring Cloud 应用迁移到 Kubernetes 后，以下组件就不需要了，在之后 [Kubernetes](#kubernetes) 一节有说明；但如以上强调的，上 Kubernetes 不是云原生应用的必需，基于 Spring Cloud 仍有优化或者说专业化改造的必要。另外注意，在包含 Spring Cloud 模块的前提下通过配置启用或禁用 Cloud 功能、和完全移除 Spring Cloud 模块后的切换配置，这两种方式的具体技术操作是不一样的。

##### 服务注册中心

在 Spring Cloud 应用中，一个服务的实例要在 Eureka 或 Nacos 等服务注册中心[注册](https://docs.spring.io/spring-cloud-netflix/reference/4.3/spring-cloud-netflix.html#_registering_with_eureka)它的元数据，包括服务唯一标识（Service ID / Name）、IP、端口等等；而该服务的调用方也需要根据服务标识从注册中心获取到对方的实例地址，之后才是直接访问实例进行服务调用，详细说明参见 [Spring Cloud LoadBalancer](https://docs.spring.io/spring-cloud-commons/reference/4.2/spring-cloud-commons/loadbalancer.html) 等。

在真实的生产环境，一个服务的实例可能因为故障从注册中心消失、也可能因扩容而更多，随着时间不断变化，因此需要注册中心和服务双方实时同步、动态更新。但是在开发场景，比如本地启动两个服务 A 调用 B，那首先不会考虑多实例高可用，其次调用地址很明确，就是 `localhost:8080` 调 `localhost:8081`，当然 B 也可能是一个外部 UAT 地址，总之在这种情况下完全不需要注册中心来提供动态的服务发现，直接指定即可。

而现代的开发框架支持下，这种调整显然是不需要改动程序的，通过配置启用禁用即可，在 Profile `dev` 的配置中禁用服务注册及发现、同时 ServiceB 指向 `localhost:8081`（硬编码，也可以理解为静态的服务发现），而到 Profile `prod` 时，启用服务注册及发现、自动从所配置的注册中心获取 ServiceB 的地址。

##### 配置中心

用户刚接触 [Spring Cloud Config Server](https://docs.spring.io/spring-cloud-config/reference/4.3/server.html) 等配置中心时，一个常见的误用就是一股脑将所有的配置项全放进去了；但实际上，大部分情况仍应该放在 Spring Boot 的配置文件，只有少量需在部署时确定、或者密码等敏感信息，才放到配置中心。

按照以上的实践指导，在本地开发时信息基本都是确定的，所以主要考虑的是在禁用配置中心后如何管理敏感信息：

- **不要在提交代码中保存敏感信息**，即使项目有访问控制、即使是测试环境的信息。
- 即使密码泄露不造成任何影响，也尽量不要这样做，**一者养成良好习惯、二者倒逼设计**。
  - 可能的例外是本地部署、只能本地访问、仅用于开发的情况，比如以下的内嵌数据库。
- 考虑使用环境变量，或者专门的 Profile 文件但**必须加入 Git Ignore**，一个小技巧是源码中带上该 Profile 已移除敏感信息的示例文件、方便其他成员参考使用。

#### 数据库

Spring 在 [Embedded Database Support](https://docs.spring.io/spring-framework/reference/6.2/data-access/jdbc/embedded-database-support.html) 解释了 Why Use an Embedded Database：

> An embedded database can be **useful during the development phase** of a project because of its lightweight nature. Benefits include ease of configuration, quick startup time, **testability**, and the ability to rapidly evolve your SQL during development.

虽然在真实的服务器环境不会使用嵌入式数据库，但以 H2 为例，它的 [SQL Support](http://h2database.com/html/features.html#feature_list) 包括 MySQL、PostgreSQL 等众多主流数据库；而即使有尽可能保证和生产环境一致的原则，在开发环境取代共享数据库还是利大于弊；当然，到 CI 和测试环境，仍应尽快调整到和生产环境相同的数据库产品，毕竟只是 SQL 兼容，需要尽早发现可能的问题。

而共享数据库**大概率**也是手工建库建表导入数据，现在则**必须将 Schema、基础数据、示例数据等 SQL 文件纳入源代码管理**，并在应用启动时自动初始化，参见 [Initialize a Database Using Basic SQL Scripts](https://docs.spring.io/spring-boot/3.5/how-to/data-initialization.html#howto.data-initialization.using-basic-sql-scripts)；还有更高级的做法 [Use a Higher-level Database Migration Tool](https://docs.spring.io/spring-boot/3.5/how-to/data-initialization.html#howto.data-initialization.migration-tool)，不过比较复杂需要团队自行斟酌：

- 在数据库设计没有基本定型、或者没有上生产前其实好处不大，上生产后的数据库变更强烈建议使用这种方式。
- 但如果不尽早让团队熟悉这种方式，等到上生产前才开始使用则又容易出问题，个人倾向一开始就用。

#### 缓存

以 Redis 为代表，2021 年改造时尝试了 [ozimov/embedded-redis](https://github.com/ozimov/embedded-redis) 取代外部共享的 Redis 缓存：

> Redis embedded server for Java integration testing

但现在（2025）发现该项目已五年未更新了，不过有从它 Fork 且 2024-10 还有更新的 [codemonstur/embedded-redis](https://github.com/codemonstur/embedded-redis)，但总的来讲容器对这种测试用途的模拟产品打击很大，因为容器能非常方便的运行和生产一致的产品，但也有限制，参见 Container Run TLDR 的 [Development Environment](container-run-tldr.md#development-environment) 一节。

不过如果使用的是 Spring 的缓存封装、而非直接调用 Redis API，那就简单了，可以通过配置直接切换到 [Simple](https://docs.spring.io/spring-boot/3.5/reference/io/caching.html#io.caching.provider.simple) cache：

> a simple implementation using a ConcurrentHashMap as the cache store is configured

Spring 对中间件访问都做了一定的封装，比如 [Spring Data JPA](https://spring.io/projects/spring-data-jpa)、[Spring Integration](https://spring.io/projects/spring-integration) 等等，这是框架的天性；而且它还提供了不同程度的封装，比如 JPA Vs. [JDBC](https://spring.io/projects/spring-data-jdbc)。总之封装必然会带来一定的取舍，是否选用只能具体问题具体分析，而且 Spring Boot 在传统企业 IT 的霸主地位并不代表它在每个细分领域都是必然选择。

#### 消息队列

由于消息队列的使用场景比缓存复杂、但又没有关系型数据库标准化，而且我们主要使用的 RabbitMQ 不是基于 Java，因此未找到类似以上中间件的嵌入式替代产品。2021 年尝试的做法是在开发环境禁用 RabbitMQ：

- 消费消息：不连接消息服务自然不会消费消息。考虑过自动化测试或在开发环境将消息接收暴露为 HTTP 服务，这种做法能够验证消息处理的逻辑是否正确，这也是最需要确认的部分，而到真实环境使用 RabbitMQ 标准接口接收消息、在这个环节出问题的概率很小。
- 发送消息：需要修改源码，通过 `@ConditionalOnProperty` 在开发环境启用 Mock，模拟发送消息但实际是将消息内容写入日志。

以上做法只能说勉强达到效果，但需要变更代码比较繁琐，所以并不是理想方式，但未找到更好的方法前仍值得尝试。

#### 其他应用

这也是运行时依赖中最复杂的部分，主要原因在于中间件等依赖在功能、接口上基本都已定型，且生态圈也有现成的针对性方案，比如以上的嵌入式数据库，不仅有 H2、还有 Spring 对 H2 开箱即用的支持。而对于内部开发的应用，只能说一言难尽……以下只能简单的列出不同场景的大致建议：

- 如果是同一代码库的多个模块并启动多个服务，服务之间有依赖：
  - 最直接的做法就是在本地启动不同端口的多个服务。
  - 但注意反思其架构设计：
    - 以真实改造的某项目为例，三个服务一个 Authentication server 一个 Gateway，只有一个是业务应用，前两者完全也应该用现成的专业产品取代，从而转为标准接口的依赖。
    - 如果是业务应用的依赖、又在同一代码库，再确认一下是应该做成 Monolith（这不是贬义词）还是按真正 Microservice 规矩来。
- 如果依赖其他团队开发的应用：
  - 该应用的 API 已基本定型：可以使用 Mock，我们实际的一个改造使用的是 [WireMock](https://wiremock.org/docs/solutions/spring-boot-integration/)。
    - 有疑问"我们的应用依赖几十个系统，没法做 Mock"？但其实并不会依赖几十个系统的全部 API，真正梳理下来可能平均每个系统就调用两三个 API；虽然这部分工作也不小，但**远不是想当然的不可行，实际上依赖越多越应该想办法简化**。
  - 没有定型：那么先反思一下整体规划。

参考方案可以了解一下 [Spring Cloud Contract](https://docs.spring.io/spring-cloud-contract/reference/4.3/getting-started/introducing-spring-cloud-contract.html)，特别是其中"Why Do You Need It"介绍的场景，即使能引入[容器](container-run-tldr.md#development-environment)极大降低应用部署的难度，考虑到无限制的间接依赖，也不可能在开发环境如此部署。不过感觉这个产品即使到 4.3 版本也没有太普及？不确定实际效果究竟有多好、成本有多高？但它确实是针对性的在解决这个领域的问题，其中的思路仍有很大借鉴意义。

### 更多

以上内容是我在接触待改造项目初期所做的主要工作，也就是围绕开篇"one sign of a good application architecture"这个主题展开，毕竟如果我不能在自己电脑就把这些应用轻松跑起来，我还怎么去深入研究？

但真正的专业化改造显然远不止此，由于投入限制，在实际改造过程中还有很多事项我们只提了一下、最多也就是蜻蜓点水，但这些事都是同等重要甚至更重要；而且还有非 Java 技术栈、非服务端应用等等，虽然核心思想不会有太大差别，但毕竟也有不同的技术形式。总之本文无法涵盖太多，以下也只列出有限的几项，更详细的内容请参见[云原生与十二因子](cn-12factor.md)，主要关注每一因子的"补充及更新"部分以及最后的小结。

#### 自动化测试

除了防止 Bug，做自动化测试还有一个隐含但也很重要的好处就是**倒逼架构的优化**。以开发团队共享数据库为例，如果要做自动化测试，就会发现共享数据库是完全不可行、必须解决的问题。

自动化测试是最复杂、最大的一个话题，是决定应用所能达到高度的**最根本因素**；但自动化测试又不是一个纯技术话题，还包括成本考量、需求管控等等，但无论自动化测试能覆盖多少，至少应用架构从设计上应该尽可能方便自动化测试……

#### 升级

版本升级是一个最直观偿还技术债的方式，当然以上所有的做法都是在偿技术债……

一个软件的版本越新，就越能适应新的形势，比如使用的 JDK 版本越高、Spring Boot / Cloud 的版本越高，自然也会越贴合新兴的容器、Kubernetes 以及周边生态圈的各种技术，但注意 Container TLDR 也讨论了 [Java in Container](container-tldr.md#java-in-container) 的天然局限。

另外注意并不是说为了云原生才需要升级，升级本身就是提升代码质量的一个主要手段，是**本来就需要定期投入的**，更多讨论见[代码质量和技术债](tech-debt.md)。不过如 Continuous Delivery 所述，有没有信心重构或升级又取决于：

> **A good automated test suite should give you the confidence necessary to perform refactorings and even rearchitecting** of your application knowing that if the tests pass, your application's behavior really hasn't been affected. 

**因此一个应用能不能轻松的实现云原生改造，完全取决于之前的基础打得多好，很多问题不能逃避**，比如现在，又兜回到自动化测试。

#### CI/CD

云原生改造前，项目就应该具备 CI/CD 流水线，实现如 Java 打包、部署到 VM 的自动化工作，否则也是极大的技术债。CI/CD 最麻烦的**仍然**是自动化测试，可以说现在大部分项目的 CI 只能算自动化构建，远谈不上真正的 [CI/CD](cicd-tldr.md)。

由于也是一个大课题，这里只讨论之上运行时依赖问题在 CI 环境的考量。如前所述，我们需要尽可能保证和生产环境一致，但由于本地开发环境的限制我们做了一定权衡取舍；而到 CI/CD 环境，通常都是 Linux 服务器，我们能够利用容器方便的部署各种依赖，这样就能够更逼真的进行自动化测试（如果有的话）；因此似乎可以启用全部真实依赖，但实际也不是这样：

- 以上[其他应用](#其他应用)一节提及的无限制的间接依赖，这决定了部署完整真实环境的成本基本不可接受，即使使用容器。
- 数据库和缓存应该和生产环境保持一致，比如使用 MySQL 以及对应版本而不是模拟的 H2。
- 消息队列如之前讨论的，如果不启用那未经改造的程序就会出错，但启用的问题是：
  - 消费消息的验证重点是处理消息的业务逻辑，通过消息队列的标准接口接收消息这个环节，是由消息队列和开放框架自身的自动化测试保证不出问题；再考虑到异步消费可能造成的延时，因此测试通常是直接调用处理消息的程序模块，而不是真的发送消息到消息队列。
  - 验证真实消息队列是否收到了发送的消息比较麻烦，同上也没必要，我们只要确认发送消息的内容符合预期即可。
- Spring Cloud 的服务注册中心、配置中心就完全没有必要部署了，因为测试主要验证的是调用了某个服务是否正确、某个配置项是否符合预期行为，至于服务的元数据是来自注册中心还是配置文件、配置数据是来配置中心还是文件，不会影响应用的业务逻辑。
- 还有未提及的 LDAP / SSO 登录等等。

总之，虽然理论上只要和生产环境有差异、就会有检测不到的地方，但实际总归有限制，也要做各种工程权衡，不能百分百一样的地方，就留待蓝绿部署、灰度发布等去做防护吧。

## 容器化改造

上一章的专业化改造工作或者说云原生改造的前置工作，有些很简单，但实际代表的是良好习惯、而形成自然而然的反应远比想象中困难，有的很复杂没有展开；而以下只会简单提及应用迁移到容器、Kubernetes 时要注意的几点，总之对比本章内容，以上占用了更大篇幅，但这并没有偏离重点，这些基础会带来深远的影响：已经打好基础的不会纠结于云原生技术是否复杂、只会热烈欢迎新技术工具克服了以前不足，而反复纠结在这上面的可以再思考一下到底需不需要云原生改造……

### 容器

#### 日志

应用日志在 VM 环境通常是输出到日志文件，然后由同一机器上的日志采集程序持续读取并传输到日志后端存储；而在容器环境，虽然技术上同样可以这样操作，但通常的做法是容器应用输出到 stdout / stderr、由容器运行时将其落盘为宿主机的日志文件，而后续的采集也是在宿主机进行。这个是"把日志当作事件流"，参见云原生与十二因子的[日志](cn-12factor.md#日志)一节。

因此当应用转为容器方式部署时，这一部分的改造工作是必需的，当然在开发框架的加持下应该只是一个配置项的调整，这应该算应用侧云原生改造中**唯一**不属于技术债的部分？

#### Dockerfile

参见 Container Run TLDR 的 [Dockerfile](container-run-tldr.md#dockerfile) 一节，注意里面还讨论了无 Dockerfile 的选项，我们也更建议采用 Jib 方式。

### Kubernetes

#### 日志

在 VM 的容器里输出日志和在 Kubernetes 的容器输出日志还是有差别的，如 [K8s Logging](k8s-logging.md) 所述：

> 在 K8s 上，无论日志存储、还是采集端，都是多应用共享的，因此一是调优很难面面俱到、二是出问题的爆炸半径大很多。

更详细的分析参见以上链接，总之在 Kubernetes 上如果一个应用无节制的输出日志，很可能会丢失日志甚至影响其他正常应用的日志采集；这个问题不是说在 VM 环境不存在，但由于独占因素得到了很大缓解。而关于日志输出的最佳实践应该是开发人员的基本功，此处不再赘述，但从实际情况看，这类技术债广泛存在。

#### 对象存储

部分应用会有多个实例通过 NFS 等共享网络目录文件，虽然 Kubernetes 从技术上来讲也支持挂载 NFS，但我们的 Kubernetes 平台定位是无状态应用，因此不会提供存储高可用的保证；但即使提供，那主要也是给数据库、缓存、消息队列等需要持久存储的中间件使用的，普通业务应用应该将共享文件存储看作类似数据库的[后端服务](cn-12factor.md#后端服务)；而对象存储正是这个领域的主流解决方案，早已不是新技术了。

对象存储通常基于 S3 API（事实标准）进行访问，使用 HTTP 方式或各主流语言的 SDK，无论哪种都需要从之前的文件读写方式进行调整，可能还需要考虑历史数据迁移等情况，因此对用户影响比较大，在计划上 Kubernetes 时需要**重点关注**；但是要清楚，这不是 Kubernetes 带来的问题，而是我们平台方的强制要求，因为我们可以接受因用户的合理要求导致平台运维变得复杂，但**不包括对过时方式的支持**。

另外注意也有技术方案将对象存储又封装成文件访问的，比如 [Amazon S3 File Gateway](https://aws.amazon.com/storagegateway/file/s3/)：

> S3 File Gateway is used for on-premises data intensive applications that need **file protocol** access to objects in S3. 

这种做法倒不需要用户调整，但对象存储毕竟不是文件存储，也不止文件读写这最基本的功能，详细比较参见 [What's the Difference Between Block, Object, and File Storage](https://aws.amazon.com/compare/the-difference-between-block-file-object-storage/)；而当对象存储已经了成为主流方案，那就应该按对象存储的方式来做，欠的技术债不能回避；而且这种做法又回到了开始的 NFS 的状态，需要 Kubernetes 平台方首先在节点机启用并保证高可用，而不像对象存储标准的 HTTP 服务，完全不需要介入。

#### Spring Cloud

Spring Cloud 的服务注册中心和 [Kubernetes Service](k8s-service-tldr.md) 重合、配置中心和 [Kubernetes ConfigMap & Secret](k8s-configmap-tldr.md) 重合，因此当 Spring Cloud 应用上 Kubernetes 时应该移除这些组件，即使技术上不是必需。而移除了这两个支持服务，自然也要移除应用里对应的客户端组件，因此应用也需要从动态的服务发现调整为指向 Kubernetes Service 的静态配置，也就是从 [client-side load-balancer](https://docs.spring.io/spring-cloud-commons/reference/4.3/spring-cloud-commons/loadbalancer.html) 又变回传统的服务端 LB；以及从配置中心调整为通过 ConfigMap & Secret 挂载的本地配置文件或环境变量。

注意 Spring 还有 [Spring Cloud Kubernetes](https://spring.io/projects/spring-cloud-kubernetes) 这个子项目，但定位比较尴尬，包括它自己都宣称：

> While this project may be useful to you when building a cloud native application, it is also **not a requirement** in order to deploy a Spring Boot app on Kubernetes.

总的来说面向 Kubernetes 我们需要的不再是 Spring Cloud 应用，Spring Boot 足够，当然还是可能用到一些 Spring Cloud 的非核心模块如 [Spring Cloud OpenFeign](https://spring.io/projects/spring-cloud-openfeign) 等等。

另外要**注意**的是 Spring Cloud 应用通常归属在一个项目群，虽然技术上全部迁移到 Kubernetes 没有任何问题，但因为各种制约因素，很难同时、统一的搬迁，因此在实际改造过程中还是会有一些麻烦问题，典型如下一节的应用访问控制。

#### 应用访问控制

可能有人想象不到现在还有很大一部分应用的访问控制只依赖网络权限，应用本身未做任何管控。即使不考虑网络权限只能做开 / 关这种两极控制、无法反应更细粒度的权限，到了 Kubernetes 环境，传统的网络权限也无法应对 Pod 的动态 IP，只能对节点机静态 IP 做形式上的"控制"，详细讨论参见 Why So Security 的 [*Case 1*](why-here.md) 一节。

如果调用双方的服务都部署在同一个 Kubernetes 集群，问题也不大，因为我们有 Namespace 的网络隔离，同时通过 NetworkPolicy 实现跨 Namespace 的网络访问，而这是基于应用更确定的元数据而非 IP。但这只是少数情况，大部分还是跨 Kubernetes 和 VM 的状况，包括以上 Spring Cloud 一节提及迁移阶段的问题。总之如果服务访问只依赖网络权限、自己未实现任何安全控制，那么**务必关注**其中的风险和漏洞，而这个问题我们唯一的建议就是先偿技术债；同理，这个危险不是上 Kubernetes 带来的。

当然业界同样存在如 Service Mesh 等不需要应用变更的方案，但这种团队可能宁愿学自己熟悉领域的基本安全也不愿学更新更复杂的 Service Mesh 吧……

#### Helm Chart

参见 [*Helm TLDR*](why-here.md)，和以上 Dockerfile 类似，Helm 不是部署 Kubernetes 应用的唯一方式。

### CI/CD

要做的是改造传统应用的流水线，添加**构建容器镜像**、通过 Helm **部署到 Kubernetes** 等任务，这部分工作由 DevOps 团队提供支持，此处不展开。
