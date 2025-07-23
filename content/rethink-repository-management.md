---
title: 重新思考 Repository Management
---

## 背景介绍

公司使用 Repository Manager 是从内部部署 [Nexus](https://www.sonatype.com/nexus-repository-oss) 服务开始，首先是作为 [Maven Repository](http://maven.apache.org/guides/introduction/introduction-to-repositories.html) 的内部[镜像](http://maven.apache.org/guides/mini/guide-mirror-settings.html)，之后将自主开发构建的 Jar 包也发布到 Nexus；再然后是作为 [npm](https://www.npmjs.com/) 内部镜像；到容器时代，也提供 Docker Image 的镜像（Mirror、Docker Image 中文都译成镜像，为了区分本文中镜像指 Mirror，Docker Image 中文用容器制品），但自己构建的容器制品发布到 [Harbor](https://goharbor.io/)。

随着企业规模的增长，在内容种类、性能压力、安全等方面都提出了更高要求，我们也使用 [JFrog](https://jfrog.com/) 商用产品取代了 Nexus 和 Harbor。但实际上在这些显而易见的非功能性需求之外，有一个被传统 Repository Manager 厂商严重忽视的地方，我们在下一节的概念说明之后继续。

## 镜像 Vs. 自主制品

从上一节的描述就可以看出，Repository Manager 起了两方面的作用：

* 作为外部第三方制品库的内部镜像。
* 发布内部开发的制品。

软件厂商也是按照这个思路在操作，如 Nexus 的 Proxy / Hosted、JFrog 的 Remote / Local Repositories，前者作为镜像后者作为内部库，但是等事情进一步复杂时，一些概念开始变得混乱。

以 Maven 为例，并不是所有的 Jar 包都是这个体系的，如 IBM DB2 的 JDBC Jar，在 [Maven Central Repository](https://repo1.maven.org/maven2/) 或其他 Public Repository 都不存在，只能去相应厂商网站手工下载并上传至 Hosted / Local Repository 供内部引用。从功能上讲，这么做没有问题，绝大多数用户也是如此操作；但是无论从终端用户的使用、还是服务本身的运维，其实是将两件不同性质的事情混淆在了一起。

无论是从 Proxy / Remote Repository 自动镜像，还是用手工方式下载上传到 Hosted / Local Repository（可以看作手工镜像），这些制品本质上都是第三方库；而内部开发构建并发布到 Hosted / Local Repository 的制品，却是企业自己的软件资产。这两者在特性上的差别比想象中要大得多，以下会详细讨论。

因此，我认为**按制品的拥有者区分**为镜像 / 自主制品来看待 Repository Management，无论对用户还是运维方，概念上更加清晰；而 Proxy / Remote / Hosted / Local 等等，退后为技术实现细节。我们以这个区分为出发点，讨论进一步的设计。

## 二级管理

在继续本文主旨前，还需要阐明这个概念。以 [Jira](https://www.atlassian.com/software/jira) 为例，它是按 Project 分组管理 Issues，除了 Jira 系统管理员外，每个 Project 有自己的管理员。系统管理员创建 Project、指定该 Project 的管理员，Project 内的管理工作不再是系统管理员关注的事，从而解脱出来聚焦于系统管理工作。而每个 Project 的管理员获得的好处是只要不涉及到全局、Project 内的事项完全可以第一时间处理，不需排队等待系统管理员的恩赐，而且也更贴近理解一线的需求。

这种模式在很多应用都存在，无论是可以 On-premises 部署的 [Gitlab](https://about.gitlab.com/pricing/#self-managed)、还是云服务的 [Github](https://github.com/)，以及之上提及的 Harbor 等等等等，都有分组的概念，只不过名称叫 Group、叫 Project、Organization、Tenant……但最本质的就是分组而无所谓用什么标准来分，取别的名字反而容易引起误导，比如 [spring-projects](https://github.com/spring-projects) 和 [spring-cloud](https://github.com/spring-cloud) 是 Spring 在 Github 上的两个 Organization，但明显 Spring 才是实体意义上的 Organization 而前两者只是 Spring 的 Project Group。

而分组之后还必须有设置组管理的机制，只有这样才做得到自助服务，用户方获得效率、运维方解脱于太琐碎的操作。当然不是说系统管理员就不做管控，以 IaaS 为例，系统管理员要做的是给某个组配额或其他限定，至于组内是建三台还是五台虚拟机、装什么系统……其实，还有更值得操心的事啊。

但是注意分组或者分层、再加上逐级权限设置，并不代表二级管理，因为事情的核心不在于有没有分组或者有没有权限设置，而是管理工作（权限只是其中一项）可不可以直观、方便的拆分下放，或者说**赋权**（Empowerment / Delegation Management）！

我认为对于企业向的应用软件，二级管理并不是可有可无锦上添花的功能点，**而是必须**。关于这个话题还有更多考量，比如系统-租户-组三级管理等等，完全值得另起一文，在此略过。

## 镜像管理 Vs. 自主制品管理

回过头来，我们看看传统的 Repository Manager 忽略了什么。从镜像的使用场景，无论 Maven、npm、Docker 还是其他，**常用、正规**的 Public Repositories 非常有限，系统管理员初始配置甚至软件默认提供的就足够使用了，真遇到新的 Repository 需要镜像，这种情况频度也不高，找系统管理员解决即可。

但是到自主制品的场景，情况就发生了大的改变。比如各个开发团队对自己制品库的属性定义、成员的权限配置，以及 Labels、Robot Accounts、Tag Retention 等等等等（参考 Harbor 提供的功能），虽然也不算高频操作，但绝不是建组后就不用管了；再考虑开发团队的数量，对于组内算低频操作但聚合起来对孤家寡人的系统管理员同样不可承受，或者，用户不可承受。

而除了自主变更管理，仅仅是能够进入自己的组查看日志、摆脱黑盒状态（比如从 JFrog Artifact Repository Browser 只能看到制品的当前状态，过程、历史都不可见），对于 DevOps 也是很有意义的事。

可以看看新一代的 Repository Manager 怎么在做， 以 [Harbor](https://goharbor.io/) 为例，它 Key Features 的 Management 类别第一条就是 "Multi-tenant"，而 Harbor 主要就聚焦在自主制品管理。其他还有如 [Cloudsmith](https://cloudsmith.io/) 对 [Org/Team Management](https://cloudsmith.io/f/org_team_management/) 的支持，以及所有云服务天生的多租户支持，另外如 Gitlab 集成的 [Container Registry](https://docs.gitlab.com/ee/user/packages/container_registry/index.html#control-container-registry-from-within-gitlab) 也天然继承了它的二级管理。【补充说明】：由于成文很早，所以提及 Harbor 说是新一代，实际现在都 [2.13](https://goharbor.io/docs/2.13.0/) 了，而多租户支持也是现在的共识，如 JFrog 也在努力追赶虽然效果一言难尽。

虽然二级管理非常重要，但并不是唯一的区别：

- **公开或私有**：第三方制品通常都是不设控制、自由获取的，而内部制品几乎不对外公开、即使对不同内部用户也往往有细粒度的权限控制，这也是二级管理的方便所在。
  - 部分商业制品在购买后也不是给所有内部用户使用的，但这又分两种情况，通常是制品本身可以自由获取、只是使用时需导入 License 或者仅是法律限制，这种情况和之上并无不同；而制品本身需要控制访问的情况很少。
- **历史版本**：第三方库通常有新旧版本同时被多个用户使用，而企业内除了少量的共享库，应用制品往往只使用最新版本，可能会回滚到最近几个版本，但再早的版本基本已回不去了。也就是说很多应用的制品库旧版本的保存价值很小，而从这个特性出发，运维上的备份清理明显可以针对性设计。
* **用户视角**：而从用户交互的习惯来看，镜像和自主制品的区别也很明显，前者的视角就是制品，而后者首先是 Group / Project 然后才是制品。

总之，我们以镜像 / 自主制品为出发点，而不是局限于 Proxy / Local，更能看清事情的本质和我们的述求。

## 新部署

按照上面的思路，就会发现无论从运维侧还是用户侧，是都可以将镜像和自主制品拆分开的：

- 运维侧：镜像服务的高可用**相对**不重要，本地只是一个缓存，所有制品都不会真正丢失；当然也就不需要考虑数据备份（配置备份还是需要的）等，但是要多考虑一下预热。所以将镜像服务和自主制品服务拆分部署，虽然变成两个服务实例貌似增加了运维工作，但针对不同甚至矛盾的需求能分开管理和调优，事实上就极大降低了运维的复杂度。
  - 当然真实落地时没这么简单。比如以上提及的需要手工获取并保存到内部的第三方制品，虽然理论上讲丢失了也可以再去找第三方要，但从发生故障需要快速恢复的要求，似乎是保存到有备份的自主制品服务里更好；但这样又会困扰用户，比如我们用 `mirror.example.com` 提供镜像服务、`repo.example.com` 提供自主制品服务……总之这是一个工程权衡的考量。
- 用户侧：如上所述自主制品管理有很多二级管理的工作，而镜像服务甚至都不需要用户登录。

当然这些都**只是一个设想**，现实是我们采购的 JFrog 产品有 License 限制，不能拆分部署；而且 JFrog 的二级管理功能非常有限，有 [sub-administrators](https://www.jfrog.com/confluence/display/RTF/Configuring+Security) 的概念，但 [Permission Targets](https://www.jfrog.com/confluence/display/RTF/Managing+Permissions) 貌似 Repository Group 但从名称就可以看出只是方便批量的权限设置而已，对比之上 Harbor 的组管理功能可以发现 JFrog 是没有这方面的设计的；至于开源的 Harbor 只支持容器制品，目前未找到一个支持二级权限管理的 Universal Repository Manager。
