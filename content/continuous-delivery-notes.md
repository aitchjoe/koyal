---
title: Continuous Delivery 摘译及笔记
---

摘译自 [Continuous Delivery](https://www.amazon.com/Continuous-Delivery-Deployment-Automation-Addison-Wesley/dp/0321601912/)，2010 年出版，作者 Jez Humble 和 David Farley。

除了几本经典著作，IT 书籍通常只看近三四年的，差点错过这本好书。如果不吃力的话**强烈建议阅读原文**（也有[中文版](https://book.douban.com/subject/6862062/)），但是对其中提及的具体技术手段及产品，包括评论的内容（摘译和笔记主要是在 2016.5.18-30），注意有过时的可能。本文只记录了最基础的第一部分，*以下斜体文字为译者注释评论*。

## 第一部分 基础

### 第一章 软件交付的问题

#### 常见的反模式

- 在很多项目中，软件发布是一个**手工密集型**过程。运行环境、所依赖的第三方软件经常是由运维团队手工创建安装，应用程序本身是手工拷贝到生产环境，配置信息手工拷贝或者通过应用服务器的控制台创建，基础数据仍然是手工拷贝，最终应用被手工启动。
- 以上就是紧张、焦虑、精神过敏、惴惴不安的由来。在以上步骤中有太多容易出错的地方，任何一步没有正确执行，应用都可能出问题；但是更紧张、焦虑、精神过敏、惴惴不安的是：不一定马上出问题、不清楚哪会出问题、不确定哪一步做错了。
- 反模式：手工部署软件。
  - 绝大多数情况下终归会有手工操作，在这里主要是指整个部署过程被拆成分离的阶段、由不同的人员或团队进行，各步骤之间的配合、团队间的沟通，很容易出现一些人为的疏忽；即使没有这些问题，不同的执行顺序、耗时仍然可能引起不同的结果，这很少会是好事。
  - **比较明显的迹象**：
    - 依赖于手工测试确认系统是否运行正常。
    - 不是几分钟能搞定。
    - 半夜三更加班。
  - 进化：需要手工操作的只应该有两件事，选择待发布的软件版本和部署环境、点击发布按钮！总之是需要**尽可能全面的自动化**。
    - 如果不能自动化，出错很难避免，区别仅在于错误是否明显。
    - 不能自动化、依赖手工操作，自然就不能严格重复，也就谈不上有信心，时间会浪费在排查部署过程中出的错。
    - 手工操作通常会要求有操作文档，但是维护文档又是一件复杂耗时通常也会牵涉多人的任务，在实际情况中文档很难保证完备且时时更新。而自动化的部署脚本如果未完成或更新，在部署时自然就会被发现。
    - 文档通常会假定读者具备一定的基础，作者往往也会基于自己的水平和思路习惯，导致不同的人按文档操作的结果不一定相同。
    - 手工部署依赖于专家或熟练工，期待他们别休假吧。
    - 手工部署既无聊又重复，但是又要具备一定的专业技能，有价值的人不应该耗在这件事上面，而是做更具创造性的工作。
    - 手工测试耗时，代价很高。
    - 手工操作也无法保证操作人员真的遵循了文档规范，反而是自动化过程更容易被审核。
- 反模式：开发完成后才着手部署生产或类生产环境。
  - **比较明显的迹象**：
    - 生产或类生产环境被限制访问，或者不能及时提供，或者没人关心。
    - 开发团队和部署团队不怎么合作。
  - **更恶化的迹象**：
    - 在开发设计时，对生产环境有一些错误的估计并不罕见；因此发布周期越长，开发人员接触了解真实状况越晚，修正起来就越麻烦。
    - 在大型组织里，整个发布流程会跨开发、DBA、运维、测试多个团队，协调沟通的代价非常大，陷入"申请审批的烂泥塘"也不罕见。
  - 进化：测试、部署、发布都应该集成到开发流程中去，使其**变为开发工作日常**的一部分。
    - 开发、测试、部署团队在项目开始就应该一起工作。
    - 不仅要测试我们开发的软件，还要**测试部署过程**。
- 反模式：手工配置管理。
  - **比较明显的迹象**：
    - 运维团队需要很长的时间准备发布环境。
    - 很难回到之前的状态，包括操作系统、应用服务器、数据库及其他基础设施的设置。
    - 同一个服务器集群的成员可能会有不同的操作系统版本、第三方库、补丁级别等。
    - 生产环境的系统配置是手工直接修改。
  - 进化：测试、Staging、生产环境的所有配置，都应该从版本管理中获取并通过自动化过程来启用生效。只有这样，才能够精确重建应用及基础设施的每一个部分，包括操作系统、补丁、操作系统配置、应用栈、应用配置、基础设施配置等等等等。
    - 所有对生产环境的变更都应该被记录和审核。
    - 不允许手工改动测试、Staging、生产环境，只能通过自动化过程。
    - 变更必须先提交到版本管理、再通过自动化过程在各环境启用。
    - 出现意外时能使用相同的自动化过程回滚。

#### 怎样实现更高目标？

- 本书两个最主要的目标之一是**缩短交付周期**，即从决定发起一个变更（无论是改 Bug 还是增加新特性）到用户真正可用的时间间隔。
  - 可用性的一个重要部分是软件质量。
  - 但质量并不等于完美，因为交付速度也非常重要。
- 为了达到更短的周期、更高的质量，我们需要**频繁的、自动化的**发布，因为：
  - 如果从构建、部署、测试、发布整个流程不能够自动化，则每一次的执行都有可能因为人为因素干扰产生不一样的结果，即不可精确的重复。
  - 只有频繁的发布软件，新版本的增量变更才能保持较小，出问题的风险相应降低且更容易回滚。
- 频繁的、自动化的发布中，反馈很重要，有以下三个准则（*Feedback 在此仅是字面翻译，具体内容见下*）：
  - 任何变更必须触发反馈进程。
    - 一个运行的软件应用包括四个部分：**可执行代码、配置、主机环境和数据**。其中任何一处变更，都可能改变应用行为。
    - 源码变更会引起执行代码变更，为了管控这个过程，构建及测试必须自动化；每次代码提交时自动构建及测试，这种实践即**持续集成**。
    - 在所有环境的执行代码必须完全一致。
    - 在各个环境不同的内容应该作为配置信息管理，所以环境的配置变更必须被测试。
    - 主机配置、数据的变化也需要被测试。
    - **反馈进程就是要尽可能全、尽可能自动化的测试任何变更**。
  - 快速反馈。
    - 快速反馈的关键就是自动化！**重复性的工作让机器去做**。
    - 代码提交阶段的测试应该：
      - 快。
      - 足够广泛，覆盖 75% 或更多代码，一旦通过则表示基本可信。
      - 一旦失败则表示有严重错误、不应该被发布到任何环境，因此比如界面配色之类的测试不应包含到该测试集。
    - 下一阶段的测试：
      - 可能会慢很多，应该考虑并行测试。
      - 如果出现部分失败，经评估后可以发布到某些环境，比如一个严重 Bug 的修正急于上线。
      - 该测试环境应尽可能和生产环境一致，因此不仅能测试软件功能，还能**测试部署过程**有没有问题。
    - 要想做到快速反馈，也需要**关注开发流程的改进**，特别是怎样使用版本管理、怎样组织代码等。
      - 开发人员应该频繁提交代码变更。（*而不是混合积累了多个功能点、按时间周期比如一天提交一次。*）
      - 代码组件的合理拆分。
      - 大多数情况下，避免版本分支。（*分支得越久，合并时积累的差异就越大、越不容易掌控，在后续章节会有更详细的讨论。*）
  - 交付团队必须响应反馈进程。
    - 所有介入交付流程的人员必须关注反馈，包括开发人员、测试人员、运维人员、数据库管理员、架构专家、管理人员等，如果不能日常就一起工作（我们更希望这样，即**跨职能团队**），那么频繁的交流是必须的。
    - 通过公开的 Dashboard 广播消息、或者其他通知机制，确保相关人员能够及时响应。
    - 当反馈结果需要人工介入时，该工作优先于团队的日常工作。
- 实用效果？
  - **一个老套的反对意见是这些做法太理想化了，可能小的团队有用，但不适合我们高大上的项目。但本书描述的所有技术和原则，已经在不同规模的组织、各种情况下的真实项目中得到了检验。**
  - 本书的实践受到了精益生产的影响，而事实上，精益生产的哲学已经在变为软件开发领域的主流。

#### 收益？

上述实践使得发布过程**可重复、可信赖、可预期**，极大的缩短了发布周期，更快、更高质量的交付给用户。

- **团队赋能**（*赋能授权是近年来应最多的商业语汇之一。赋能授权的意思就是授权给企业员工——赋予他们更多额外的权力。逻辑上来说，这样做意味着为了追求企业的整体利益而给予员工更多参与决策的权力。理论上,赋能授权是为了消除妨碍员工们更有效工作的种种障碍，其思想出发点是企业由上而下地释放权力——尤其是员工们自主工作的权力,使员工们在从事自己的工作时能够行使更多的控制权。*）
  - 允许测试人员、运维人员、技术支持人员**自助服务**，不需要象以往那样发邮件、提申请、各式效率低下的填表沟通。
  - 很方便的部署任一版本到任何环境还有以下好处：
    - 测试人员可以很轻松的部署一个旧版本以比较新旧差别。
    - 技术支持人员可以部署一个特定版本以重现故障。
  - 团队成员越能够掌控自己的工作、不需花太多时间等待，他们的产出质量越高，相应最终的软件质量也就越高。
- 减少错误
  - 这里主要是指由于配置管理缺乏或较差、在部署到生产环境时引起的错误。
  - 一个运行良好的应用，除了正确的代码、代码的正确版本，还包括正确的数据库 Schema 版本、负载均衡的正确配置、正确的 Web 服务 URL 等等等等。当我们提及**配置管理**时，就是指通过一整套机制流程来识别、掌控上述信息的全集。
  - 某实际案例：所有测试环境都是通过应用服务器的控制台来手工配置。
    - 开发环境的配置信息已保存到版本管理。
    - 测试环境的配置信息未使用版本管理，且每个测试环境的配置都有不同，包括配置字段的顺序不同、字段有缺失、有不同值等，因此**很难区分**哪些配置字段是重要的、哪些是冗余的、哪些是不同环境下一致公用的、哪些要依赖于环境等等。
    - 作为结果，雇佣了 5 人的团队负责这些事儿。
  - 即使是简单的把配置信息纳入版本管理，就已经**迈进了一大步**。
  - 当然下一步就是自动化处理，而不是运维人员一遍一遍的敲键盘。
- 降低压力
  - 只要软件发布还是个大事件，人当然就会感到紧张。
  - 但是如果是频繁的发布、每次发布的增量变化很小、发布只是按一个按钮等几分钟甚至几秒、万一有问题也能飞快的回滚，那么发布的风险被极大的减弱，所有的不快自然也会显著的降低。
- 灵活部署
  - 但是要达到这个目标，不仅是本书所提及的实践，**应用本身也必须有良好的设计**。
- 熟能生巧
  - 采用同一种部署方式部署到所有环境，不应该针对 QA、UAT、生产环境有特殊的部署策略；这样的话，每一次部署都是在验证我们的部署机制是否正确、都是对最后部署到生产环境的一次预演。
  - 有一个例外即开发环境，但仍然希望尽可能保持一致。

#### 待发布版本

- 当代码发生变更后，是否应该发布，不应该寄希望于猜，而是由构建、部署、测试整个流程来决定。
- 传统的做法是只有在经过耗费时间精力的手工测试之后，才能确认是否发布；但如果已经具备全面的自动化测试，那么软件质量应该很高、手工测试只是对业务功能完成度的一个确认。
- 如果是在开发阶段之后才开始测试，以我们的经验，这就代表软件质量的**必然降低**，缺陷刚出现时才是最容易被发现和修正的。
- 每次代码提交都是一个潜在的发布版本。
  - 只有在测试环境被验证，才能知道新的代码是否已破坏系统？这个集成阶段通常是开发过程中最难掌控、最不可预期的部分。
  - 既然集成很痛苦，很多团队会推迟做、更少做，但这只会让事情变得更糟。
  - **在软件领域，如果做某事很痛苦，那么降低痛苦的方法是更频繁的做这件事，而不是逃避。**
  - 持续集成的实践就是将集成的频度做到极限直至每次提交，这样才能尽可能快的发现问题解决问题。
  - 如果**测试覆盖足够充分、测试环境足够接近生产环境**，那么这个软件事实上就是一个可发布的状态。

#### 软件交付原则

要想有效的交付软件，以下原则是必须遵守的。

- 建立可重复、可信赖的软件发布流程。
  - 发布软件可以是一件轻松的事，如果你已经上百次的测试验证了整个发布流程。
  - **可重复、可信赖取决于**：尽可能的自动化，尽可能的将构建、部署、测试、发布的所有内容纳入版本管理。
  - 部署软件归根结底是三件事：
    - 创建和管理运行环境，包括软硬件配置、基础设施、外部服务等。
    - 安装应用的正确版本。
    - 配置应用，包括数据或状态。
  - 配置信息可以保存在版本管理或数据库，并通过脚本自动处理；虽然硬件本身不能做版本管理，但是使用虚拟化技术以及 Puppet 等工具，创建流程仍然可以完全自动化。
- 尽可能的自动化。
  - 当然有些工作是无法自动化的，比如 Exploratory 测试需要依赖有经验的测试人员，**但是不能被自动化的工作远远少于大部分人所认为的**。
    - 验收测试可以被自动化，数据库的升级和降级（*不是指数据库软件、而是应用的数据库 Schema 及基础数据等*）也可以自动化，甚至网络和防火墙配置也能自动化。
  - 当然，也不是说一开始就要全做到，可以从瓶颈着手。
- 尽可能的纳入版本管理。
  - 包括需求文档、测试脚本、自动化测试案例、网络配置脚本、部署脚本、数据库创建/升级/降级/初始化脚本、应用栈配置脚本、库、工具链、技术文档，等等等等。
  - 变更集合应该有唯一标识。
- 如果做某件事痛苦，那么就更频繁的做这件事，把痛苦提前。
  - 这是一个通用的原则（*不局限于软件交付领域*），但实际上也是**最有效果**的。
  - 比如写文档痛苦，那么在开发时就去写、而不是留到最后，且要求只有文档完成才代表整个开发工作完成，当然这个过程中也有自动化的可能（*如 Java Doc*）。
- 品质优先（Build Quality In）。
  - **越早发现问题，解决的代价越小。**
  - 本书的实践也是为了尽早的发现问题，即把痛苦提前。
  - 测试不是一个分离的阶段，当然就更不是一个在开发阶段之后的阶段了。
  - 测试不应该只是测试人员的事。
- 完成代表可发布。
  - 只有交付给用户才代表真正完成！当然在实际情况下，部署到类生产环境、至少能被内部用户所访问，也可以算做功能性的完成。
  - **没有 80% 完成！**任务要么完成、要么就是没完成。可以估算剩余的工作，但终归只是估算，当估算的数字收不住的时候，常常会引起相互指责。
- 每个人都要为交付负责。
  - 在太多项目里，开发把工作扔给测试，测试又扔给运维，出问题时难免吵成一锅粥。
  - 如果是小型组织或者比较独立的部门，你能够会掌控软件发布所需的一切资源，这样太性福了！但实际情况是打破孤岛组织之间的藩篱、实现这条原则会是一个**长期艰苦**的工作。
  - 因此在开始一个新项目时，让交付流程的所有干系人都参与进来、确保足够频度的沟通非常重要，只有持续、足够的交流，隔阂才能消除。
  - 更进一步，要有一个系统工具能够让所有人**都看到**应用的状态、健康程度、构建及测试结果、环境的状态等等，同时提供**自助服务**。
  - 这也是 DevOps 的核心思想。
- 持续提升
  - 应用的首次发布只是开始，应用会不断进化，继续发布新版本；同时，发布流程也要不停改进，这非常重要。
  - **全局优化强于局部优化，因此每个人都有权利了解整体情况，偏向局部优化最终将导致互相指责。**

### 第二章 配置管理

#### 简介

- 配置管理在很大程度上**等同于版本管理**。
- 配置管理不仅管理一个项目所有的相关项，还管理这些**对象之间的关系**。
- 配置管理记录了系统和应用的演进过程，还决定了团队之间如何协作，而后者往往在配置管理策略中被忽略。
- 最显而易见的配置管理工具当然是版本管理软件，但采用它只是第一步，一个成功实施的配置管理策略应该：
  - 能够精确的重建任何环境，包括操作系统版本、补丁级别、网络配置、软件栈、应用及配置。
  - 能够容易的将上述内容的增量变更部署到任何环境。
  - 能够很容易的搜索追踪变更：谁？什么时候？做了什么样的变更？
  - **团队的每个人都应该享受到以上的便利，这一点非常重要。**
- 本章主要解决的问题：
  - 两个前提条件：把所有内容纳入版本管理、依赖管理。
  - 管理应用配置。
  - 管理应用所依赖的整个环境。

#### 版本管理

- 版本管理不仅是保存历史记录，也提供了软件交付的**协作机制**。
- 将所有内容纳入版本管理，使应用整个生命周期的变更保持在可控状态。
  - 虽然版本管理系统也经常被叫做源代码管理系统，但绝不限于管理源代码，还包括测试、数据库脚本、构建部署脚本、文档、应用所需的库和配置文件、编译器等工具集等等，可以让团队的新成员能够马上投入工作。（*取决于所使用的具体技术，如 Maven，就不需要保存应用所需的库，而只需直接依赖库的 POM 配置信息，但这个还需要 Nexus 等的支持。*）
  - 当然还包括操作系统和软件栈的配置信息、DNS zone files、防火墙配置等等。
  - 甚至还包括开发环境的配置文件，保证开发团队的每个成员使用相同的设置。（*比如 facebook/react 下有很多以"."开头的配置文件如 .editorconfig。*）
  - 还有很多项目会保存应用服务器、编译器、虚拟机及其他工具链的二进制映像文件。（*作者对于这种做法倾向于支持，但是本人还有疑惑：是保存映像文件本身、还是只保存配置？特别是基于现在的 Docker 等技术如何处理？*）
  - 没必要保存编译结果。
  - 版本管理还带来一个潜在的好处，即可以很轻松、没风险的删除不需要和过时的内容，这对于维护大型复杂的配置信息很有用。（*虽然通过版本历史能够很容易的恢复，但是很多人的习惯仍然是注释掉、而不是删掉，本来的用意是备忘提醒、但最终的结果只会是干扰。*）
- 频繁提交，提交到主干。
  - 在有些团队，提交的周期是以天甚至周来论，这是有问题的，只有频繁提交，才能享受到版本管理带来的更大好处。更具体的说，如果每个人不能频繁提交到主干，则不可能安全的重构应用，合并工作将变得异常复杂。
  - 很多时候在开发新功能时，人们会建立一个版本分支，等做的差不多的时候才考虑合并回主干。但是我们**反对这种做法**：
    - 这种做法和持续集成背道而驰，因为集成问题只有等合并时才会发现。
    - 如果几个开发人员建立多个分支，问题将成指数倍复杂。
    - 虽然有些工具提供自动合并功能，但解决不了语义冲突。
  - 更好的做法是**渐进的开发新功能**，而仍然频繁提交到主干，以保证整个软件一直在集成且处于可工作状态。将在 13 章更详细的讨论该问题。
  - 为了保证你的提交不破坏整个应用，以下两个实践很有效：
    - **在提交前，运行提交测试集**。（*原文中 Commit 和 Check-in 会交替使用，有微妙的区别，有时代表技术含义，有时指流程阶段，在此统一翻译为提交。*）该测试应该尽快完成（少于 10 分钟），但还是要相对完备，至少能排查一些明显的回归错误。
      - 有些持续集成软件提供 pretested commit 功能，可以将这部分测试从开发机转移到集成服务器上去运行。
    - 渐进的引入变更，外在表现是：通常**一天应该有好几次提交**、最低限度也应该是一天一次。可能刚开始这样做会觉得别扭，但是这绝对会带来非常大的效果。
- 提交说明不能太抽象。
  - 一行很简短的说明作用不大，通常要有多行比较详细的说明语句，包括第一行概述。
  - 如果有其他项目管理工具，应该包含相关链接。

#### 管理依赖

- 应用常见的外部依赖包括：
  - 第三方库：通常是二进制文件（如果不是使用解释型语言），不会被我方开发人员修改。
  - 组件模块：我方其他团队开发，可能处于正在开发的阶段，变更频繁。
- 关于依赖管理更具体的讨论见 13 章，在此仅关注依赖管理对配置管理产生影响的一些关键点。
- 管理外部库。
  - 是否将外部库纳入版本管理尚有争议，比如 Maven，如果没有本地缓存，那么都需要从 Internet 下载外部库。（*针对 Maven 这种场景，个人认为并无争议：在已部署内部 Nexus 服务器的情况下，上述限制不存在；其次 Maven 是通过 POM 配置来管理依赖信息及依赖的传递，如果要版本管理具体的外部库文件，反而很怪异。*）
  - 将外部库的二进制文件纳入版本管理的好处是更大程度的保证重建结果一样，但是问题是检出的体积和时间将变大。
- 管理组件。
  - **应该是二进制依赖**，而不是源码级依赖，这很重要。

#### 管理软件配置

- 配置和执行代码、数据是组成应用的三个关键部分，配置信息可以在构建、部署、运行阶段改变应用的行为。
- 必须**以对待代码的方式对待配置，即恰当的管理和测试**。
- 配置和灵活性。
  - 灵活性是有代价的，最常见的**反模式**是"万能配置"，常会作为一个需求提出，这个不但无益、严重情况下可能会摧毁一个项目。
  - **而提出这种需求通常是基于这样的迷思：动配置要比动源码风险小。但事实上：**
    - 如果改动代码，有很多现成的方法比如编译器、自动测试来检测错误，起到一定的保护作用。
    - 反而配置信息通常是没有约束、未测试的状态，比如将配置里的一个外部服务调用 URL 改成一个非法的字符串，大部分系统只会在运行时才知道出了错。
  - 但并不是说配置就是天生邪恶的，而是说应该被小心和一致的管理。现代编程语言通常会有各种特性和技术来帮助减少错误，但大多数情况下配置信息并未得到这些保护，也很少有测试来验证在测试或生产环境这些配置是否正确，因此**部署阶段的冒烟测试是一个必要手段**。
- 配置的方式。
  - 配置信息可以在构建、部署、测试、发布阶段被注入，但总的来讲在构建或打包阶段注入配置信息并不是个好主意，因为这违反了以下原则：在所有环境运行的执行代码必须是完全一致的。
  - **在部署阶段配置你的应用**，指定它所具体依赖的服务如数据库、消息服务器、外部系统等等。
  - 如果你能够控制你的生产环境，应该**使用部署脚本来获取这些配置信息**并提供给你的应用实例。（*注意这里说的控制生产环境，并不是指你是否运维人员、有没有权限控制，而是本书涵盖了服务端应用、以及客户安装应用如 MS Office 等多种场景，而后者只能是客户自己去安装部署的，你控制不了。*）
  - 可以在应用启动时和运行时配置你的应用：
    - 启动时：通过环境变量、命令行参数。
    - 运行时：通过注册表、数据库、配置文件或外部的配置服务（提供 SOAP/REST 接口）。
  - 强烈建议组织中所有应用的所有环境下的所有配置信息，都应该通过一致的机制集中管理，这表示你的所有变更、管理、版本控制和覆盖，都有**单一来源**。虽然这很难，但是我们见识过没做到的后果。
- 管理应用配置。
  - 考虑应用的配置管理时，问问自己这三个问题：
    - 配置信息如何编排？
    - 部署脚本怎样访问配置信息？
    - 在不同环境、应用、版本下，配置信息有何不同？
  - 可以在数据库、版本管理系统、目录服务器等保存配置信息，版本管理系统是最现成的；应用配置应该和源码保存在同一个存储库（*应该是指缺省配置和开发环境的配置，和下一段对比*）。
    - 虽然数据库等提供了很方便的存储和远程访问，但是如果不能保存变更历史以便稽核和回滚的话，意义不大。
  - 测试、生产环境的配置信息应该和源码分开保存，不在同一个存储库。
    - 总的来讲，这部分内容和版本管理的其他内容的**变更频度是不一样的**。
    - 如果采用了这种做法，一定要**小心追踪哪个版本的配置信息对应哪个版本的应用**。
    - 这种做法对于安全相关的配置信息更有意义，比如密码和数字证书等需要限制访问的情况。
  - 不要在版本管理里保存密码，应该在部署时输入。
  - 访问配置信息。
    - 最有效的方式是有一个**集中的配置服务**供所有应用访问。
  - 配置信息编排。
    - 配置信息依赖于应用的版本及运行环境。
    - 一些用例：
      - 应用的新版本经常也会引入一些新的配置项或者去除一些旧的，要确保在部署新版本后能获取新的配置项、回滚后又能用到旧的。
      - 将新配置扩展到各个环境，要保证各环境的新版本应用都能获取新的配置项，但是配置项的具体值应该根据不同环境被正确设置。
      - 如果数据库服务器改变了地址，应该能非常容易的、将所有涉及到旧地址的配置项变更指向新地址。
      - 使用虚拟化技术管理运行环境，因此配置虚机的信息也应该作为应用配置的一部分管理。
    - 我们管理跨环境配置信息的一个策略是将生产环境的配置设为默认配置、然后其他环境根据情况覆盖相关项的默认值（注意配置防火墙，以防万一疏忽的话不会从别的环境访问到生产环境）。
      - 但有些组织权限隔离比较严格，会将生产环境配置信息专门单独保存。
  - 测试验证系统配置。
    - 以相同的方式测试应用、构建脚本以及配置信息。
    - 首先确认外部服务地址的配置是否正确。比如在部署脚本里要检查指定地址的消息服务的运行状态、最低限度也应该能够 Ping 通，如果所依赖的任何服务不可用，则部署失败，这就是个很好的**冒烟测试**。
    - 其次运行一些专门制作的**冒烟测试用例**，检查和这些配置项相关的功能是否运行正常。
- 集中管理应用配置。
  - 在中大型组织里，大量的应用配置信息会集中管理，这会让事情变得复杂，更何况有些遗留系统的配置还晦涩难懂。
  - 对于服务端应用，清晰了解应用的当前配置非常重要；通过运维监控工具，可以看到每个环境的每个应用实例的版本、配置信息等等，这些工具包括 [Nagios](https://www.nagios.org/)、[OpenNMS](http://www.opennms.org/) 等。
  - **实时访问**这些信息非常重要。
  - 在所有项目的初始阶段就要考虑配置管理，尽可能的使用同一种方式。
- 应用配置管理原则：以对待代码的方式对待配置，包括管理及测试。
  - 在代码里保存配置项的所有选项，但是配置项的取值应该保存在别处；配置项的设置和代码的生命周期完全不同，且密码等敏感信息也不应该保存到版本管理。
  - 必须标记配置项属于哪个应用、哪个版本及哪个环境，这样才能够通过自动化过程、从配置库获得特定的取值。
  - 配置项命名应规范清晰。
  - 配置信息也需要模块化及封装，以保证某个配置项的改动不会对明显没关系的其他配置项产生影响。
  - 在部署或安装时测试配置信息。

#### 管理环境

- 没有应用会是孤岛，每个应用都会依赖硬件、软件、基础设施及其他系统。具体的环境管理见 11 章，在此主要讨论和配置管理相关的内容。
- **管理环境的配置和管理应用的配置一样重要。**
- 在 The Visible Ops Handbook 书中，作者将手工配置环境称为"艺术工作"，为了降低环境管理的成本和风险，**环境也应该成为一个可以批量重复生产、且耗时可预期的产品**，否则对于开发进程将是一个持续的拖累。
- 环境管理的关键是**完全的自动化处理**，创建一个新环境应该比调整一个旧的更容易。
- 能够复制环境至关重要：
  - 在可预期的时间内重建一个环境以回到之前的良好状态，肯定强于花好几个小时还不确定能不能搞定。
  - 拷贝生产环境用于测试很重要，从软件配置来说，测试环境就是要尽可能的复制生产环境，以尽早发现配置方面的问题。
- 各种环境配置信息包括：
  - 操作系统版本、补丁级别、配置。
  - 安装的附加软件包，版本及配置。
  - 网络拓扑。
  - 应用所依赖的外部服务，包括版本及配置。（*这不应该算应用配置么？还是说环境不仅仅指主机？*）
  - 其他在环境中存在的数据或状态，比如生产数据库（？）。
- 要实现有效的配置管理策略，应该遵循两个原则：**二进制文件和配置信息分开管理、配置信息集中统一管理**。如果系统的每个层面都能做到，那么新建环境、升级系统、安全启用新配置等会变成一个简单的自动化过程。
- 虽然将操作系统本身纳入版本管理不切实际，但是管理它的配置很正常，结合一些工具如 [Puppet](https://puppet.com/)、[CfEngine](https://cfengine.com/) 等，集中管理配置操作系统并不困难。
- 第三方软件也应遵循上述原则，无需用户交互即可直接从命令行安装、使用版本管理配置信息。事实上，这也是评估第三方软件的一个重要考量点。
- 在配置管理术语里，正确部署处于良好状态的环境被称为**基线**。一个自动化的环境管理工具能够创建、重建项目历史中的任一基线。一旦应用的主机环境发生变更，就应该保存这些变更、创建新基线版本，并将应用版本与基线版本关联起来。
- 本质上，对待环境和对待代码的方式是一致的：**渐进的产生变更、将变更纳入版本管理、每次变更必须测试**以保证新环境下应用未被破坏。
- 环境管理工具。
  - 使用 Puppet 等工具就不需登入每台机器手工操作。
  - 而虚拟化技术更加简化环境管理过程，只需创建虚机镜像作为基线。
- 管理变更流程。
  - 生产环境应该被完全锁定，只能通过变更管理流程进行变更。即使再小的变更都有可能破坏系统，因此必须通过测试才能进入生产，而要做到这点，环境变更的操作必须脚本化并纳入版本管理，整个过程和代码的变更是相同的。
  - 测试环境的做法类似，仅仅是审批更简单，除此以外，测试环境的管理、部署、配置应该和生产环境是一样的机制。当然，如果生产环境昂贵的话，测试环境不需保持同等档次。

#### 小结

- 配置管理是本书其他所有内容的基础，缺少配置管理，则不可能做到持续集成、发布管理、部署管道等等，同时配置管理对交付团队的协作也有巨大的正面影响。
- 再次强调，配置管理不只是挑选和使用一个工具、虽然这也很重要，但**最关键的，还是引入一系列最佳实践**。

### 第三章 持续集成

#### 简介

- 在开发阶段，软件功能长期都处于不可工作的状态，这种情况在很多项目都存在，原因可以理解：在完成以前，没人想去尝试整个应用。虽然开发人员仍在源源不断的提交代码甚至跑单元测试，但是并不会真的在生产相似的环境里运行它。不觉得这很怪异吗？
  - 如果长期在分支上开发，或者将验收测试推迟到开发结束，那么这种现象更加突出。
  - 很多项目会在开发结束时有一个漫长的集成阶段来合并分支或者验收测试，搞不好在这个时候才突然发现开发的功能并不是用户真正想要的，总之结果很不可预期。
- 持续集成的目标就是**保证软件始终是处于一个可工作的状态**。
  - 持续集成要求一旦有人提交变更，整个应用将被构建并运行完备的自动化测试集。
  - 更关键的是，只要构建或测试失败，开发团队必须停止所有其他的工作来解决这个问题（*修正错误或回滚*）。
  - 因此应用在每次变更时最多只有几分钟是处于不可用的状态。
- 持续集成首次在 Kent Beck 的 Extreme Programming Explained（1999）中提及，主要思想是既然集成是好事（*传统开发也有集成，比如在每天夜间或其他周期*），为什么不始终都做？在集成这个领域，"始终"意味着每一次变更被提交到版本管理时，**"持续"也意味着比你想象的更加频繁**。
- 持续集成代表着**范式转移**（*指行事或思维方式的重大变化*）。没有持续集成，通常是在测试或集成阶段才知道是否出问题；而使用持续集成，每次提交都能马上反馈（注意**前提是必须有足够完备的自动化测试集**），而**越早发现问题、解决问题的代价就越小！**因此对于专业团队来说，**持续集成是一个极其重要、必不可少的实践准则**，甚至和版本管理一样重要；而最终，是更快、更少缺陷的交付软件。
- 本章内容主要面向开发人员，但是项目管理人员也应该关注。

#### 实现持续集成

- 准备工作：
  1. 版本管理。
  1. 自动化构建。
     - 构建必须能够在命令行执行，而不是只能依赖 IDE，这样才能够被机器自动化执行。
     - 虽然现在的 IDE 工具能够很方便的构建或运行测试，但是我们认为仍然需要可以通过命令行运行的构建脚本：
       - 可以在持续集成环境自动化运行并稽核。
       - 构建脚本也应该被当作代码对待，应该被测试并不断优化以保证简洁清晰，而这是 IDE 生成的构建过程做不到的；当项目越来越复杂时，这会变得更加重要。
       - 使构建更容易理解、维护和排查错误，并且也更容易和运维人员协作。
  1. 团队共识。**持续集成是一种实践，而不是一个工具**。它要求开发团队的成员必须遵守一定的纪律，每个人都应该**渐进的、频繁的提交小幅度变更到主干**，且同意修复变更错误是最高优先级的工作。**如果不认可这些，持续集成并不能带来你所期望的质量提升**。
- 持续集成系统基础。
  - 开始使用持续集成服务器时，**先尝试遵循以下简单的流程**，一旦你准备提交你的最新变更：
    1. 检查构建是否正在运行？如果是，等待它完成；如果失败，则应该和团队成员一起先解决这个问题，然后才能提交。
    2. 如果构建已完成且测试成功，从版本库获取该版本的最新变化内容并同步到自己的开发环境。
    3. 在自己的开发环境运行构建脚本及测试，确认在自己电脑上仍然是全部正常工作（或者使用持续集成工具的 personal build 功能）。
    4. 如果本地构建成功，提交代码到版本管理。
    5. 等待持续集成服务器完成你提交的变更所引起的构建。
    6. 如果构建失败，停止你手头的所有其他工作，立即在你的开发环境解决这个问题，然后再转到第 3 步。
    7. 如果构建成功，可以继续下一个工作。
  - 如果每个团队成员在每一次提交时都遵循上述步骤，那么你就能确认应用可以在每台和持续集成服务器配置一致的机器上都能正确工作。

#### 实现持续集成的前提条件

- 频繁提交。
  - **频繁提交到主干可以说是持续集成最重要的实践准则**，一天至少应提交好几次。它能带来以下好处：
    - 这样做每次的变动就会较小，因此出错的可能性也会变小。
    - 出错后也更容易回到上一个好的版本。
    - 和其他人的变更产生冲突的可能性也会变小。
    - 能鼓励开发人员主动去尝试一些新的构想，因为能很容易的回滚。
    - 也可以有规律的休息一下。
    - 甚至当你的电脑完全坏掉时，损失也不大。
  - 很多项目使用版本分支来管理大的团队，这对于持续集成来说是错误的，从定义来说，你如果是在分支上开发，你的代码就没有和其他开发人员的代码集成。我们认为分支只应在非常有限的几种情况下创建，具体讨论见 14 章。
- 建立完备的自动化测试集。
  - 如果没有完备的自动化测试，那么一个成功的构建仅仅代表应用可以被编译成执行代码，**当然这对有些团队来讲也是一大进步**，但是具备各个层面的自动化测试才能真正保证你对应用的信心。
  - 有很多自动化测试种类，我们将在下一章详细讨论，在此我们主要关注在持续集成构建阶段运行的三种测试：单元测试、组件测试和验收测试。
    - **单元测试**通常不需要启动整个应用，也不会和数据库、文件系统或网络等交互，也不需要在和生产相似的环境运行。单元测试应该非常快，即使是大型应用的全部单元测试集，也应该在十分钟内完成。
    - **组件测试**通常也不需要启动整个应用，但是可能会和数据库、文件系统或者外部系统（不一定是真实的）产生交互，运行时间长于单元测试。
    - **验收测试**检验应用是否符合业务需求，既包括业务功能，也包括系统特性如容量、可用性、安全等等。应该在生产相似的环境运行整个应用来做验收测试，时间很长，如果未使用并行测试听说有超过一天的。
  - 结合以上三类测试，那么将有足够的信心来保证任何变更都未破坏现有功能。
- 构建和测试越快越好。
  - 理想情况下，提交前以及在持续集成服务器上的编译和测试过程应该在几分钟之内（*此处持续集成服务器上的测试不是指全部测试，见后文*），十分钟是极限，五分钟好一点，九十秒以内最理想，对于小项目来说十分钟已经非常长了。
  - 这个要求和之前提及的完备测试集似乎自相矛盾，但是可以通过一系列技术手段在保证相同测试覆盖率的前提下缩短时间，包括瓶颈定位和优化等，这也是在持续集成中需要经常进行的实践。
  - 等复杂到一定地步时，需要将测试过程拆成多个阶段：
    - 提交阶段：
      - 编译软件，运行单元测试，生成部署包。
      - 这一阶段的测试在提交前（*在本地开发环境*）、以及每次提交后在持续集成服务器上，都应该执行。
      - 在提交阶段加入一些简单的冒烟测试非常有用，包括一些简单的验收测试和集成测试以确认最常用的功能没有被破坏，做到更快的反馈。
    - 第二阶段：
      - 运行验收测试，还可以包括集成测试、性能测试等等。
      - 在提交阶段的测试都通过后才执行本阶段的测试，通常耗时很长，如果超过半个小时就应该考虑并行测试了。
  - 验收测试最好按功能区域分组，当某个区域发生变更时，可以先关注这个区域的测试集是否通过，很多测试框架都支持这个功能。
  - 当应用大到拆分成独立模块时，需要仔细设计组织这些子项目，既从版本管理的角度，也要从持续集成（*测试*）的角度，详细讨论见 13 章。
- 管理开发环境。
  - 本地开发环境所使用的自动化流程必须和持续集成、测试、生产环境中的一致，而这需要：
    - 除了源码，包括测试数据、数据库脚本、构建脚本、部署脚本，应该全部纳入配置管理。
    - 第三方库和组件的依赖必须纳入配置管理，Maven 等可以帮助解决这个问题。
    - 保证自动化测试、包括冒烟测试，能够在开发机执行。对于大型应用，会涉及到配置中间件和运行轻量级的内存数据库或单机版数据库，这可能比较麻烦，但是每次提交前可以在开发环境运行冒烟测试，这能够极大的提高应用质量。**事实上，一个良好应用架构的标志就是：能够很容易的在开发环境上运行应用**。

#### 使用持续集成工具

- 持续集成软件最基本的功能就是轮询版本管理系统、检查是否有新的提交，有的话则检出最新版本、运行你的构建脚本进行编译、执行测试，最后通知你结果。
- 醒目的通知。
  - 持续集成工具带来的一个重要好处就是**可见性**，即每个人都能毫不费力的看到构建状态，比如很多持续集成软件都可以在所有人的桌面安装一个窗口小控件来显示构建状态。
  - 然而这种可见性可能会让客户怀疑你们的开发质量，如果他们看到了这些失败通告。但是事实正相反：每一次构建失败，就代表了一个问题被发现、避免了混入到生产环境。当然，让客户真正安心可能没这么简单，最根本的还是努力保证构建成功，但绝对不要降低可见性。
  - 还可以在构建时进行代码检查和分析，如测试覆盖率、冗余代码、代码规范、代码复杂性及其他代码健康度指标，分析结果同样要公布出来以保证可见性。

#### 必须遵循的实践准则

- 构建失败时不要提交。
  - 当其他开发人员提交了代码变更但是在持续集成服务器上构建失败时，我们不应该继续提交变更、触发新的构建，尽量保持局面简单等待问题能尽快解决。
- 始终在提交前在本地运行提交阶段测试集。
  - 软件支持的话也可以在服务器上运行（*以下有具体讨论，但要注意这和服务器运行提交后的构建测试是两码事*）。
  - 提交前的测试就类似于文章发表前的校对工作，虽然我们希望提交足够轻量级，但是它也应该足够正式。
  - 可能有人会问：既然在提交到持续集成服务器后的第一件事就是编译代码并运行提交阶段测试集，那么我们为什么在提交前还要在本地也测试？
    - 提交前的正规做法是需要从版本管理检出最新的内容，如果在你上次检出之后又有其他开发人员提交，那么双方的变更合并在一起可能造成新的错误，因此检出后需要在本地确认不会破坏构建。（*理论上仍然存在本地测试后别人又有新的提交从而无限循环下去，但概率变小很多。*）
    - 还有一个经常出现的提交错误是只提交了修改的内容、而忘了加入新增的文件，因此提交前后两次测试的结果不一样将会缩小怀疑的范围，比如这个原因，或者以上提及的别人也提交了新变更。
  - 很多现代的持续集成软件提供 pretested commit、personal build 或者叫 preflight build 的功能，服务器可以直接从你本地获取变更并在服务器端构建测试，如果构建成功则会替你提交变更，但是这个功能最好是配合分布式版本管理系统进行。（*注意到分布式版本管理后，commit 的含义已有变化，可能 push 更接近通常所说的 commit，请注意通过上下文区分，文中所说提交多是通常的含义。*）
- 提交测试通过后才能继续前进。
  - **出错很正常，我们的目标是尽早发现错误尽快解决错误，而并非不切实际的期待完美零错误。**
  - 每个提交变更的开发人员在提交期间，必须负责监视持续集成服务器的构建进程，在完成编译及通过提交阶段测试集之前，他不能开始任何新的工作，也不能开会或吃饭。（*不是完整的测试，那个耗时很长，而提交阶段测试集相应很快。*）
  - 当且仅当提交构建成功，开发人员才能开始新的工作，如果失败，则能马上响应解决：或者提交新的补丁，或者回滚后慢慢修复。
- 构建失败时不要回家。
  - 这并不是要求你加班到很晚来解决问题（*而是说应该尽量避免这种处境*）：
    - 足够频繁和足够早的提交变更，**保证有足够的时间解决问题**。
    - 留到明天再提交：很多有经验的开发人员会设一个截止时间，比如下班前一小时不再提交，把提交作为下一天的头一件事。
  - 避免不了时则回滚，**总之不能干扰别人的工作**。
  - 严格的构建纪律很重要，可以设专职的构建管理员，监督**谁引起构建失败谁负责**解决，如果有人违反这个原则可以直接回滚他们的提交。
- 随时准备回滚。
  - 在大的项目里，构建失败的情况每天都可能发生，虽然使用 pretested commits 有助于消除这种情况。（*不太理解，个人认为 pretested commits 仅仅是把执行场所从本地开发环境挪到了持续集成服务器，本质上还是解决不了多人版本冲突造成的构建失败，还是说因为在服务器能执行的更快，所以出问题的时间窗口会更小，降低了概率？*）
  - 正如飞行员所接受的教导，每次飞机着陆时都要假定会出问题，随时准备放弃着陆、拉起后等待下次机会。一旦提交后构建失败，**最重要的就是尽快让应用恢复到可工作状态**，不管因为什么原因如果我们不能很快的解决问题，那么就应该回滚这次提交，将问题留在本地开发环境研究解决。归根结底，我们**使用版本管理的首要目标就是能够方便的回滚**。
- 在回滚前做有限的尝试。
  - 可以在团队订立这样的规矩：如果构建失败，尝试在十分钟内解决问题，如果超时则回滚。当然如果已经修正并在本地编译测试中了，有时也可以放宽一下，如果成功则皆大欢喜，否则无论是在本地还是提交后构建失败，则马上回滚至可工作状态。
  - 有经验的开发人员通常都不会破例，对其他人构建失败超过十分钟就会很嗨皮的回滚。
- 不要注释掉失败的测试。
  - 上一条规矩的后果是：为了让自己的变更提交成功，开发人员经常会注释掉失败的测试。这样做是错误的。
  - 如果以前通过的测试突然失败了，需要仔细分析它的原因。是回归错误？还是测试的某个检查条件不再有效？还是测试的应用功能已经改变？这都是需要花时间和相关方讨论确认的：
    - 如果是回归错误，修改应用程序。
    - 如果是检查条件变更，修改测试程序。
    - 如果该功能已经去除，删除相关测试。
  - 如果有紧急的功能需要上线，或者需要和客户更深入的沟通，注释掉测试只能作为最后的手段偶尔使用。但是这种做法**非常危险**，建议在大屏幕上显著广播被注释掉的测试数目，也可以加一些临界值，比如注释数超过测试总数的 2% 则构建失败。
- 对由于你的变更引起的所有测试失败负责。
  - 如果你的变更通过了你自己写的所有测试，但是别人写的测试失败了，这通常表示你引入了一个回归错误。是你的责任来解决所有的测试失败，因为是由于你的变更提交引起的。这么做在持续集成里很**显然**，但其实很多项目里并没有这样做。
  - 要做到这点，你必须能够访问所有相关代码，而不是每个开发人员只拥有他所负责功能的部分代码。要有效的实现持续集成，**每个人都应该能访问完整的代码库**。
  - 如果必须要做代码管控，那么一定要保证开发人员之间的良好沟通。但这只是次优解，更应该做的是努力移除这些限制。
- 测试驱动开发。
  - 完备的测试集对于持续集成至关重要。在下一章详细讨论的自动化测试里，持续集成最重要的效果就是快速反馈，而这只有在具备足够高的单元测试覆盖率才有可能（验收测试覆盖率也很重要，但是耗时太长）。
  - 而要获得满意的单元测试覆盖率，以我们的经验（我们并不是教条主义者），唯一的方法就是测试驱动开发。
  - 简单介绍一下测试驱动开发，当要开发一个新功能或者修复一个缺陷时，开发人员首先编写一个测试作为应用程序预期行为的一个可执行规格说明。这样做不仅能驱动应用设计，也可以作为回归测试甚至应用程序的文档。

#### 鼓励遵循的实践准则

- 极限编程
  - 持续集成是 Kent Beck 书中描述的十二项极限编程实践之一，虽然它自身就能给开发团队带来巨大的变化，但是结合其他实践，如测试驱动开发、共享代码等等，将更加有效。
  - 重构意味着在不改变应用行为的同时小幅、增量的提升代码质量，而持续集成和测试驱动开发能够鼓励开发人员放心大胆的去尝试。
- 违法架构原则时构建失败。
- 测试过慢时构建失败。
  - 正如之前提及，持续集成最好是小幅、频繁的提交。如果提交测试耗时过长，将逼着人们更少的提交。
  - 因此如果某个测试超过指定时间则认为构建失败，上次我们定的规矩是不超过两秒。
  - 这种做法可能是双刃剑，比如持续集成服务器可能因为其他的压力导致运行过慢。
  - 注意我们这里讨论的是测试的效率，而不是性能测试。
- 出现编译警告或者违反代码规范时构建失败。
  - 有人认为出现编译警告就构建失败太严厉了，虽然这有助于保证良好的编程习惯。
  - 一个折衷的方法是对比上一次的提交记录，如果警告数或者 TODO 数增加了，则构建失败。

#### 分布式团队

（*不同地域、跨时区的流程配合等内容略过*）

- 集中管理持续集成。
  - 将持续集成作为一个集中服务（云服务）提供给大型、分布式的团队。
  - 以一个完全自动化的流程提供新环境，由交付团队自助获取。
  - 很方便的自主服务，通过完全自动化的方式创建新环境、配置、构建及部署。这个对于开发团队**至关重要**，如果一个团队总是要发邮件被动等待分配一个新的持续集成环境，那么他们会绕开这个流程在自己能掌控的机器甚至桌面上做持续集成，哦，更坏的就是不做了。
- 技术相关。
  - 对于跨区域的团队，带宽很重要。
  - 应该考虑分布式版本管理系统如 Git。

#### 分布式版本管理系统

- 分布式版本管理系统是对团队协作的一次革命（*绝不仅仅是把代码分布式保存、而仍按照以往 SVN 的方式来用*）。
- 分布式版本管理的核心特性是每个存储库都包含了项目的完整历史，这意味着没有哪个库是特殊的，除非人为约定。对比集中式版本管理，本地的变更必须先提交到本地存储库然后再推送到其他库，从其他库获取的更新也必须先同步到本地库。
- 分布式版本管理提供了新式、更有效的协作方式，如 [GitHub](https://github.com/)，开创了开源项目合作的新模式。但是这个模式挑战了持续集成的一个基本原则：所有变更必须提交到主干。
  - GitHub 很常见的模式就是先 [forks](https://help.github.com/articles/about-forks/) 再 [pull requests](https://help.github.com/articles/using-pull-requests/)，因此没有做到真正的持续集成，在合并阶段也经常失败。
  - 必须指出，基于分布式版本管理，仍然可以使用提交到主干的开发模式来完美的实现持续集成。即简单指定某个库为主库，持续集成服务器监控它即可，同时所有开发人员必须将变更推送到这个库。
  - 采用和持续集成不一样的方式，同样可以建立高质量的软件，但是必须满足：
    - 规模较小、非常有经验的提交者。
    - 经常合并，避免差别太大。
  - 这种方式比较适合大多数开源项目和小团队，但是对中大型规模的全职团队并不适合。
  - 即使在传统的持续集成系统里，分布式版本管理也是个非常优秀的工具。

### 第四章 测试策略

#### 简介

- 太多项目仅仅依赖人工验收测试来保证功能和非功能需求，即使有自动化测试存在，也总是缺乏维护和过期，还是需要补充大量的人工测试来保证。
- W. Edwards Deming 指出："**减少依赖大规模的检查来保证质量，而是从根本里改进流程和构建高品质产品**"。
- 测试是涉及整个团队的跨职能活动，从项目开始就应该持续进行。
- 构建质量意味着：
  - 编写各个层面的自动化测试（单元测试、组件测试、验收测试），并始终作为部署管道的一部分在运行，只要发生变更，无论是应用、配置或者环境、软件栈。
  - 手工测试仍然是构建质量很重要的一部分：Showcases、可用性测试、探索性测试等等，同样需要在整个项目周期持续进行。
  - 构建质量也意味着持续改进提升自动化测试策略。
- 在理想情况下，从项目开始测试人员就应该和开发人员、用户一起合作编写自动化测试，这些测试作为应用行为的可执行规格说明，如果全部通过，就意味着用户的需求已被完整正确的实现。每次发生变更时，持续集成系统都将执行这些自动化测试，因此也起到了**回归测试**的作用。
- 这些测试不仅测试系统的功能，容量、安全及其他非功能需求都应该尽早确定，并着手编写相应的自动化测试集来保证这些需求，越早发现问题，解决的代价越小。系统的这些非功能项测试给开发人员重构和调整架构提供了实证。
- 尽早采用正确的实践准则，是可以达到理想状态的，越晚越困难。达到一个高水平的自动化测试覆盖率，是需要时间和周密计划的，确保团队成员在学习实践自动化测试的同时开发仍在持续进行。
  - 遗留系统更麻烦，但是仍可以从包括自动化测试等方法得益，以提升系统质量。
- 测试策略的设计就是一个识别项目风险并排定优先级、以及采取相应对策缓解消除的过程，测试能够约束并鼓励采用好的开发实践，完备的自动化测试集甚至提供了最完全、最新的应用文档，**以可执行的形式，明确的不仅是系统应该怎么工作，还包括它实际是怎样工作的**。

#### 测试类型

- 根据测试是业务相关还是技术相关、是用于支持开发过程还是评估系统，分为以下类型：
  - 支持开发过程（**支持开发过程的测试应该都是自动化的**）：
    - 面向技术：单元测试、集成测试、系统测试。
    - 面向业务：功能验收测试。
  - 评估系统：
    - 面向技术：非功能验收测试（容量、安全等），手工加自动化。
    - 面向业务：Showcases、可用性测试、探索性测试，为手工测试。
- 支持开发过程的业务测试。
  - 这部分测试通常叫功能测试或验收测试，确认业务需求的是否真的被满足。在敏捷开发中验收测试非常关键，因为对开发人员它回答了"我怎么知道我真的做完了？"，对用户它回答了"这是我真正想要的吗？"。
  - 总的来说每个需求或故事都有一条标准的用户操作路径，这叫做 happy path，通常会表示为"Given（测试开始时系统的主要状态），when（用户执行一系列操作）、then（系统的新状态）"，在测试中这也被称为 given-when-then 模型。
  - 但是除了最简单的应用，通常还有其他操作路径 alternate path 和出错路径 sad path。因此测试需要考虑从不同的初始状态经过不同的操作路径达到不同的预期状态，等值分区和边界值的分析将在保证覆盖度的前提下减少测试数量，但难免还是要依赖直觉来挑选更效的测试。
  - 验收测试应该尽量在接近生产的环境运行，人工验收测试典型的运行在 UAT（用户验收测试）环境，即无论在配置还是应用状态，都尽可能的接近生产环境，当然对外部服务的调用可能是模拟的。
  - 自动化验收测试的好处：
    - 反馈快速：不需要测试人员即可基本确认是否完成需求。
    - 减轻测试人员的工作量。
    - 将测试人员从无聊的重复性工作中解放出来，专注于探索性测试和更有价值的工作。
    - 验收测试也代表了强效的回归测试集，这对于大型应用大型团队非常重要，因为无论框架还是某个模块的变更，都可能对其他组件产生影响。
  - 回归测试非常重要，横跨了上述象限的分类。**回归测试代表了自动化测试的整个集合**，确保你的变更不会破坏之前的功能，因此也更容易更有信心重构。每一次写自动化验收测试的用例，就是在丰富你的回归测试集。然而，维护自动化验收测试是有代价的，因此有人反对建立大型复杂的自动化测试集，但采用正确的做法并借助适合的工具，是可以极大降低成本的，最终收益将远超投入。
  - 但是也并非所有任务都能自动化，易用性、一致的界面风格等等，很难自动验证，探索性测试也不可能自动化。但是测试人员仍然可以使用自动化工具辅助测试，比如自动设置测试场景及准备测试数据等。
  - 如果其他类型的回归测试已经完备，自动化验收测试仅覆盖完整的标准路径（happy path）和比较重要的其他路径，我们认为也是安全有效的。通常只有超过 80% 的代码覆盖率才算是完备的测试，但是要注意单独的覆盖率并不是太有意义的指标，测试质量非常重要；另外 80% 是指各类测试都超过 80%，而不是加总到 80%。
  - **怎样知道你的自动化验收测试到底做得好不好？很简单，看你有多大的信心重构甚至调整应用架构？**（*如果连 JDK 升级都认为是个大事件，那应该是对测试没信心吧。*）
  - 因为每个项目都有特殊之处，所以我们要注意到底有多少时间花在重复性的手工测试上？当同一个测试重复了多次，我们就需要考虑自动化它。
  - 验收测试应该和UI交互吗？
    - 验收测试通常是运行在尽可能真实的环境下的端对端测试，这意味着在理想状态下，应该直接操作应用的界面。但是大部分 UI 测试工具和UI结合得太紧密，以至于 UI 有轻微的调整都会破坏测试，这并不是想要的结果，如果应用行为发生了变化那么测试确实应该失败，但是网页里一个按钮名称的变化却不至于。因此按这种方式做，为了保持测试和应用的同步，会耗费大量的时间，但收益甚小。在这里可以问问自己这个问题：到底有多少测试失败是因为程序缺陷引起的？有多少是因为需求变化引起的？
    - 有几种方法可以缓解这个问题，一是在 UI 和测试间增加一个抽象层，以减轻因为 UI 变化所引起的工作；二是基于 UI 之后的 API 运行验收测试（这并不代表 UI 就不能包含业务逻辑）。
    - 所有这些并不能完全消除 UI 测试，但是可以将大部分业务逻辑的验收测试从 UI 测试剥离。
  - 最重要的自动化测试是针对主要的标准操作路径，每个需求或故事应该**至少有一个标准路径的自动化验收测试**，这些测试应该被开发人员作为**冒烟测试**以提供快速反馈。这是测试自动化的首选目标。
  - 当有时间补充测试用例时，这时需要考虑先选择其他操作路径还是出错路径？如果你的应用相对稳定，那么优先考虑其他路径，而如果系统缺陷很多经常崩溃，那么就先测试出错路径。
- 支持开发过程的技术测试。
  - 这部分测试由开发人员编写和维护，包括单元测试、组件测试和部署测试。
  - 单元测试不应该和数据库、文件系统和外部服务等交互，总的来讲也不应该和系统的其他组件交互，因此能够非常快速的反馈。单元测试应该覆盖至少 80% 的代码执行路径，形成回归测试的关键部分。
  - 组件测试因为涉及到环境设置、IO操作等，会明显比单元测试慢。有时候组件测试也被叫做集成测试，但集成测试这个词用得太泛，因此本书避免使用这个词。
  - 部署测试在每次部署时进行，以检查确认应用是否正确安装、配置、访问外部服务等等。
- 评估系统的业务测试。
  - 这些手工测试确认应用是否真正满足了用户的需要，这不仅是验证应用是否符合需求说明，**也要验证这些需求说明是否真的正确**。我们**从来没有**接触过、甚至听说过一个应用开始就能设计得完美的，当用户开始真实使用时，总能发现有提高的余地。**软件开发应该是这样的迭代过程，能够自然的促进快速有效的反馈循环**。
  - Showcases 是这类测试的一种重要形式，敏捷团队在每次迭代结束时给用户展示他们能够交付的新功能，这种展示必须在开发中频繁进行，以保证对需求说明的任何误解都能尽早发现。
    - 但是 Showcases 也会带来负面影响，用户会忍不住提出更多的想法，这时需要双方一起权衡取舍。但无论如何尽早反馈的利远大于弊。
    - Showcases 就是项目的心跳。
  - 探索性测试是一个创造性的学习活动，不仅是发现缺陷，而且根据在测试中所获取的信息设计新的、更好的测试，甚至带来新的需求。
  - 可用性测试用于发现用户在使用你的软件时有多容易完成他们的目标，这也是评估你的应用真正给用户带来多少价值的终极测试。
    - 通过情景调查以及直接观察用户的使用情况，包括操作的时长、是否按错按钮、多久找到正确的输入框等等，根据这些指标分析出用户的满意度。
  - 最彻底的做法是让真实用户使用 Beta 版本。事实上，很多网站一直处于 beta 状态，一些前瞻性的网站如 Netflix，会持续发布一些新功能给选定的用户（*评估实际使用效果以决定是否全面推广*），很多组织也采用 Canary Releasing 方式在生产环境同步运维多个版本进行对比分析。这种做法是**革命性的进步**。
- 评估系统的技术测试。
  - 验收测试分为两类：功能测试和非功能测试。非功能测试意味着关注的是系统质量、而非业务功能，包括容量、高可用、安全等等。
  - 但是很多项目并不重视非功能需求，虽然用户不会很明确的说需要多大容量或者多安全，但如果系统因为容量问题宕机、或者信用卡被盗，对于用户仍然是很难接受甚至不可接受的。因此很多人认为不应该叫非功能需求，而是跨功能需求或者系统特性。当然在本书仍按约定俗成叫非功能需求，但无论如何，应该和功能需求一致对待。
  - 非功能测试的检查条件和工具相比功能测试有很大不同，这些测试对资源和技能的要求较高，且耗时很长，因此倾向于推迟进行，而且即使是实现了完全自动化，通常运行的频度也会少于功能测试。
  - 但是随着工具和技术越来越成熟和主流化，我们建议在项目开始时就设置一些基本的非功能测试，不管有多简单或者微不足道；而对于复杂或关键的项目，应该投入一定的资源从开始就研究和实现非功能测试。
- 测试替代品（Test Doubles）。
  - **自动化测试的一个关键过程是在运行时将系统的某一部分替换成模拟版本**，按照 Gerard Meszaros 在 xUnit Test Patterns 书中的术语定义，分为以下类型：
    - Dummy objects：仅仅是作为参数传入，但实际永远不会使用到。
    - Fake objects：可以工作但只是一个简单实现，并不适合生产环境，最常见的就是内存数据库。（*注意这里的内存数据库是指如 H2 等简化开发环境的小型嵌入式数据库，而非 GemFire 等可以用于生产环境的大型内存数据库。*）
    - Stubs：针对调用提供预先编排好的结果，如果在预期之外则不会响应。
    - Spies：也是一种 Stubs，但是还会记录一些调用信息，比如模拟 Email 服务会记录消息发送的数量。
    - Mocks：编程实现一些预期的响应，如果在预期之外则抛出异常。Mocks 和 stubs 的区别详见第 8 章。
      - Mocks 是测试替身中经常被滥用的一种形式，很容易误用 Mocks 编写无意义且不稳定的测试，比如使用内部实现的细节、而非接口的约定作为判断条件。

#### 真实场景及策略

- 新项目。
  - 新项目是最容易开始实施自动化测试的。
  - 但是必须要给团队的所有人，包括用户和项目管理员，都带来好处。我们见到过因为花了太多时间在自动化验收测试上而被取消的情况，如果用户真的愿意为了抢占市场而牺牲质量，他们必须在清楚后果的情况下一起做这个决定。
  - 最后，验收测试的编写一定要从用户的角度表达出商业价值，盲目而拙劣的指定验收条件，是造成测试集无法维护的一个主要原因。这也意味着**测试人员必须从开始就介入需求分析**，确保在整个系统进化的过程中，有一致的、可维护的自动化验收测试集在支持。
  - 我们比较过一开始就使用自动化验收测试和事后才引入的情况，我们总是会发现前者代码有更好的封装、更容易理解的意图、更清晰的关注点分离以及更多的代码重用。这绝对是一个良性循环：**在正确的时机进行测试将带来良好的代码**。
- 项目中期。
  - 从最常用、最重要、最高商业价值的用例开始自动化测试，这需要和用户深入沟通，基于这些讨论，从高价值场景的 happy path 着手进行。
- 遗留系统。
  - Michael Feathers 比较极端的将**遗留系统定义为没有自动化测试的系统**，这个标准虽然有争议，但是简单直观。
  - 针对遗留系统，首先需要建立自动化构建过程，然后是自动化的功能测试框架，然后就是自动化测试集。
  - 很多时候，项目发起人并不情愿让开发团队花时间在他们以为的低价值工作上，比如为一个已经运行在生产环境的系统编写测试："不是 QA 团队已经测试通过了吗？"。其实应该很容易理解，这防止的是回归错误。
  - 一开始时，并不需花太多的时间在自动化测试上，可以小幅增量的增加新的测试，但是对于遗留系统的新功能开发，自动化测试必须作为开发完成的一个必要条件。
  - 但这有时候比听起来困难得多，可测试的系统在设计上通常会更加模块化，而遗留系统往往没有一个好的结构。
- 集成测试。
  - 如果你的应用需要通过一些列协议和形形色色的外部系统交互，或者应用内部都是松耦合的模块需要进行复杂的交互，那么集成测试将非常重要。
  - 集成测试和组件测试的边界比较模糊，在此集成测试是指测试应用组件和它所依赖的服务之间的交互是否正常工作。
  - 编写集成测试和验收测试的方式是一样的，但是集成测试会运行在两个环境：
    - 和所依赖的真实外部服务、或者外部服务的复制品进行交互。
    - 和自己建立的、外部服务的替代品进行交互。
  - 要注意不要影响到生产环境，除非自己也是生产环境，或者有方法通知调用的服务这是用于测试的 Dummy object。使用以下方式保证不会出现疏忽：
    - 防火墙隔离。这种方式还有一个附带的用途，就是测试当外部服务不可用时的行为反应。
  - 在理想情况下，服务提供方应该提供一个和生产服务完全一致的复制品（除性能外），供调用方测试，但实际上，调用方经常需要自己开发替代品用于测试，因为：
    - 外部系统已经定义好了接口，但是实际开发仍在进行（包括接口也有可能修改）。
    - 即使外部系统已开发完，但是并没有部署一个测试服务器给你用，或者测试系统太慢、太多缺陷以至于无法频繁的运行自动化测试。
    - 测试服务存在，但返回结果尚未定型，因此不可能作为测试的依据。
  - 测试替代品可能会很复杂，这取决于它是否要记住状态。
  - 注意，不仅要模拟外部服务的正常调用，也要模拟预期之外的行为，一个远程调用经常会因为各种网络原因引起故障，比如拒绝连接、丢包、接受连接但无响应、非常慢等等等等。测试替代品应该能模拟这些情况。
    - 同时也应该测试应用在这些不可控的情况下的反应，比如使用断路器等模式提高系统的容错性。
  - 当应用被部署到到生产环境时，自动化集成测试同时也充当了冒烟测试，还可以作为监控分析生产环境的工具。
  - 和外部服务的集成是一件复杂、需要花时间去小心计划的事情，每一次集成一个外部系统，都增加了一份风险：
    - 该服务是否可用？是否行为正常？
    - 服务提供方是否有足够的精力回答问题、修正缺陷、增加定制功能？
    - 可否访问生产系统来测试分析容量和可用性问题？
    - 服务的 API 是否在我方应用中容易使用？还是需要专门的培训？
    - 需要编写维护自己的测试服务吗？
    - 如果外部服务出现问题，我方应用怎样应对？

#### 流程

- **如果沟通不顺畅，验收测试将耗时费力、代价巨大。**
- 在开发之前，测试人员和开发人员就应该尽可能早的在一起讨论验收测试。
- 每个迭代结束时的切换过程很容易变成瓶颈，最坏的情况是开发人员在下一个迭代的中途还要被打断解决之前的问题，因此开发、测试人员在整个开发过程中都必须更紧密的配合。
- 管理积压的缺陷。
  - 如果采用测试驱动开发以及持续集成，同时具有完备的单元、组件、验收测试集，那么在测试人员和用户使用之前，大多数缺陷都会被开发人员捕获。然而通过探索性测试和真实的使用，还是能够发现新的问题，这部分缺陷通常会作为积压缺陷管理。
  - 虽然希望发现的问题能够立即解决，但终归避免不了缺陷的积压，这时最重要的是大家都清楚的了解情况，而开发人员有责任减少积压。
  - **对待缺陷应该和对待功能一样**，因为解决缺陷也是要花时间的，所以应该由用户来决定是解决一个缺陷、还是实现一个功能的优先级，通常也取决于这个缺陷是否频繁发生、对用户的影响有多大、是否有变通方案等等。因此在任务列表里，缺陷和功能是可以一起排序的。

#### 小结

- 在很多项目里，测试被作为独立的阶段由专门的人完成，但实际上，只有当测试成为交付团队每一个人的责任、从项目一开始就采用正确的实践、并贯穿项目的整个周期，才能够产出高质量的软件。
- 测试主要关注的就是**建立快速的反馈循环**，以驱动开发、设计和发布。任何把测试推迟的做法，都破坏了反馈，人们不会真正知道项目到底完成多少。
- 将测试结合到交付流程的每一部分是完成项目的关键。因为**测试定义什么是完成，测试的结果是项目计划的基础**。

## 第二部分 部署管道

（*Deployment Pipeline 字面直译，未找到权威 IT 翻译*）

- 第五章 部署管道解析
- 第六章 构建和部署脚本
- 第七章 提交阶段
- 第八章 验收测试自动化
- 第九章 非功能需求测试
- 第十章 部署及发布应用

## 第三部分 生态圈

- 第十一章 管理基础设施和环境
- 第十二章 管理数据
- 第十三章 管理组件和依赖
- 第十四章 高级版本管理
- 第十五章 管理持续交付
