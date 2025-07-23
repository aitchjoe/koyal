---
title: Try GitLab Issues
---

## Why GitLab issues

很多人接触 [GitLab issues](https://docs.gitlab.com/13.11/ee/user/project/issues/) 的第一个问题就是和 Jira、ITSM 比如何？明确的说从功能上肯定要弱于 Jira，典型的包括 Issue 属性（结构化字段）的丰富程度、Workflow 的灵活度或可配置性等等，但是不要急于下结论，我们先看看 GitLab 的 [dogfood everything](https://about.gitlab.com/handbook/engineering/#dogfooding)：

- https://gitlab.com/gitlab-org/gitlab/-/issues
- https://gitlab.com/gitlab-com/gl-infra/production/-/issues

无论 GitLab 自身的开发、还是 GitLab 云服务的运维，都是基于 GitLab issues 在支持，我们也知道人家所达到的高度；或者再看看类似的 [GitHub issues](https://docs.github.com/en/github/managing-your-work-on-github/about-issues) 所支持的项目：

- [Spring Boot](https://github.com/spring-projects/spring-boot/issues)
- [Kubernetes](https://github.com/kubernetes/kubernetes/issues)
- [1000+](https://github.com/search?l=&o=desc&q=stars%3A%3E1000&s=stars&type=Repositories)

回过头对比一下传统企业的 IT 水平，可以明显看出，基于更高大上的项目管理流程管理工具、有着更丰富的管理功能，**并不代表**高质量的最终产出（开发的应用、运维的平台），总不能说虽然我开发的应用差、但是我项目管得好吧。当然不提这个对比，谁也能想到人才是决定最终品质的主要因素，所以问题变成了：

- 那如果用上更多的管理工具，产出是不是就能进一步提升？

这个问题倒是有一个直观的证明，就是 GitLab 提供了 [Time Tracking](https://docs.gitlab.com/13.11/ee/user/project/time_tracking.html) 功能用于记录预估和花费的工时，但实际上 GitLab 自己 Dogfooding 的 Issues（以上链接，包括 [Merge requests](https://gitlab.com/gitlab-org/gitlab/-/merge_requests?scope=all&utf8=%E2%9C%93&state=merged)）完全没有使用这个功能，没有通过预估时间来**管理**安排计划进度、也没有通过花费时间来**管理**考核工作量，而这一点在 [GitLab Values](https://about.gitlab.com/handbook/values/#measure-results-not-hours)（GitLab 价值观）中也明确强调了：

> **Measure results not hours**
>
> We care about what you achieve: the code you shipped, the user you made happy, and the team member you helped.

当然 results 通常很难度量，但这**绝不是**找一个好度量的 hours 顶上去的理由！而且认真想想难道工时就容易度量吗？每个开发人员配一个计时钟，开始编程按一下结束编程按一下？思考的时间算不算进去？记录的时间真不真实？如果信任他们还需要计时吗？如果不信任每个人再配一个计时员？总的来讲诸如工时之类的管，有没有意义、有多大的意义、实际上能不能落地，即使不完全否仍这种需求的合理性，也请真正考虑一下能起到多大效果、有多大的概率是反效果。当然这已远不是讨论一个工具了，更深入的思考参见 [Twitter Notes](twitter-notes.pdf)，但下面仍会继续这个议题，针对 GitLab issues 相比若干个**管理**系统（ITSM 等）所缺失的管理功能点，具体情况具体分析，这也是本文的一个重点。

事实证明不"管"的、简单的工具是可以帮助团队最终产出高质量结果的，同时这些团队对所谓的管理功能无感甚至反感。但疑问仍然存在：

- 我们水平低，是不是还是用"管"的工具比较好？
- 如果不管了，出的纰漏谁负责？

如上所述，这远不是一个工具的范畴，以上的 Twitter Notes、包括 [DevOps](devops-talk.md) 都是在讨论这个大命题，但也没有简单的答案。因此回到本节的主题，我们无法告诉你用了 GitLab issues 就一劳永逸，但事实是：

* GitLab 已证明足够可行。
* 我们也知道"管"的现状。

所以我们可以把主题改成 **Why not try GitLab issues**，而我们也率先在自己项目作了真实尝试（以下 [Real cases](#real-cases) 一节），并实际体会到了 **Less is More** 并不是一句空话。

## GitLab CE Vs. GitLab EE Vs. GitHub

上一节还有一件事没有说清楚，GitLab 有免费（CE）/ 收费（EE）多个版本，具体的功能比较参见 [Self-Managed Feature Comparison](https://about.gitlab.com/pricing/self-managed/feature-comparison/)，收费版本包含更多高级功能其中自然有部分是管理向的，而 GitLab 自己 Dogfooding 的 Issues 很显然使用了最高版的功能。但是以和 Issues 相关的几个重点收费功能为例：

- Scoped Labels：[Introduced in GitLab Premium 11.10.](https://docs.gitlab.com/13.11/ee/user/project/labels.html#scoped-labels)
- Issue Weights：[Introduced in GitLab Starter 8.3.](https://docs.gitlab.com/12.10/ee/user/project/issues/issue_weight.html)
- Multiple Issue Assignees：[Introduced in GitLab Starter 9.2.](https://docs.gitlab.com/12.10/ee/user/project/issues/multiple_assignees_for_issues.html)
- Issue Dependencies：
  - [Introduced in GitLab Starter 9.4.](https://docs.gitlab.com/12.10/ee/user/project/issues/related_issues.html)
  - [The simple "relates to" relationship moved to GitLab Free in 13.4.](https://docs.gitlab.com/13.11/ee/user/project/issues/related_issues.html)

暂且不讨论这些功能算不算管理功能、是更方便了一线还是方便了管理，仅仅从以上标记的引入时间，就可以看出 GitLab 在大部分时间（当前版本为 13.11）使用的仍然只是简单功能；总的来讲这一节的意外并不影响上节的结论，而且实际上 GitLab 自己在毫无限制的情况下也没使用一些所谓的管理功能，如前面提及的 Time Tracking，典型的还有基本没用起来的 [Requirements Management](https://gitlab.com/gitlab-org/gitlab/-/requirements_management/requirements)（仍然是基于简单的 Issue 在做）。当然这个也可以解释为惯性，但是可以看到上面的 Scoped Labels 是很快就广泛使用了（批量替换？），但 Issue Weights 等又没怎么用；实际上分析一下 GitLab 自己使用了哪些功能、无视了哪些功能，是一个有趣的课题。

以个人的判断，GitLab 毕竟也是个商业公司，它有自己的价值观和最佳实践，也就是说有自己的取舍；但是面向客户，如果你非要这个功能、还愿意花钱买，那么在无伤大雅的前提下就给你这个功能，它没有闲心和精力去提升客户的真正竞争力，厂商莫不如是，当然说的好听点也可以是努力满足客户需求、不设任何限制。但是在另外一些人的眼中，GitLab EE 所添加的一些内容，可能不是加分项、而是减分项。

**使用 GitLab 还有一个隐含而巨大的好处就是之上提及的 Dogfooding**，无论产品文档多完善、实施专家多专业，一个最真实、基本公开、持续发展的原厂高水平参考项目，对于愿意自主钻研的用户，其意义无论怎样强调都不为过。事实上这种情况非常罕见，我们使用 OpenShift、也能看到它的源码，但是我们并不知道红帽具体在怎么运维 [Red Hat OpenShift cloud services](https://www.redhat.com/en/technologies/cloud-computing/openshift/openshift-cloud-services)；或者我们使用 Spring 框架、也能看到它的源码，但是基于 Spring 开发企业应用和 Spring 的自身开发基本不是一个路数，并没有太大参考价值。而使用 GitLab issues、参考 GitLab 自己怎样使用 Issues，却是一个完全匹配且公开的场景，似乎 GitLab 都没有认识到这个价值？

但正因为这样，使用 GitLab CE 版，却无法完全学习 GitLab 的做法了。而 GitHub，虽然也区分免费收费版，从 [Compare all features](https://github.com/pricing#compare-features) 看，除了容量、安全等方面的一些限制，在功能上几乎所有版本都是一致的，但是由于 GitHub 自身闭源，却又无法了解它是怎么基于自己的 Issues 管理自己的开发、运维 github.com。

所以我们只能综合学习这两者。以 Scoped Labels 为例（作用是同一个 Scope 下的 Label 在同一个 Issue 下是互斥的），无法参考 GitLab，但是我们可以参考 GitHub 上 Kubernetes 的做法，它在 [cncf-cla](https://github.com/kubernetes/kubernetes/labels?q=cncf-cla) 这个 Scope 下定义了 `cncf-cla: yes` 和 `cncf-cla: no` 两个 Label，注意我们参考的是互斥（还有[层级](https://github.com/kubernetes/kubernetes/labels?q=area)）Label 的命名方式，而实际上 GitHub 并未自动控制其互斥，但要知道 [Naming things](https://martinfowler.com/bliki/TwoHardThings.html) 才是最困难的事啊。至于效果，如果不是想精确的统计某个 Label 的 Issues 数量（管理？究竟拿这个能做什么？），做事的人自然会一眼区分矛盾的 Label、[/relabel](https://docs.gitlab.com/13.11/ee/user/project/labels.html#assign-and-unassign-labels) quick action 也很方便。

所以在我们使用 GitLab issues 的实践中，实际对标的是 GitLab CE + GitHub，在以下提及的 GitLab isues 均是指这两者。

## Real cases

[*openshift-ops*](why-here.md) 是我们 PaaS 平台的运维项目，我们首先在这个项目使用了 GitLab issues，但一开始不是作为 Issue tracking、而是 Checklist。

先说一下背景，OpenShift 升级是一件很复杂的事，我们在[*升级文档*](why-here.md)（该链接需要相应权限）中详细总结了完整步骤：

1. 判断升级必要性。
1. 确认升级路径：跨度过大的升级可能无法一步到位、需要中间版本过渡。
1. 更新知识：新版本可能会简化一些内容，因此不应该僵化的按照老的升级步骤走。
1. 下载升级软件：OpenShift 4 并不能简单的通过内部镜像服务器下载最新软件，需要手工下载。
1. 验证：OpenShift 测试集群上也有用户的"生产"任务，都需要提供生产级别保障，因此需要先在 PaaS 团队内部的试验集群上确认升级过程。
1. 检查当前环境：OpenShift 4 有些异常状态不影响使用但会阻止升级，需提前解决。
1. 事务工作：如变更申请、通知用户等。
1. 备份。
1. 实施升级。

以上的套路对绝大部分升级场景都适用，从中可以明显看出文档的大部分内容是无法脚本化自动处理的。由于该任务的专业性，**阅读文档、熟悉背景知识是绝对无法绕过的，但是通过以下方式，却能够让运维人员更少出错、更有信心**：

1. 将升级文档提炼为 GitLab 的 [Description templates](https://docs.gitlab.com/13.11/ee/user/project/description_templates.html)，主要步骤通过 [Task lists](https://docs.gitlab.com/13.11/ee/user/markdown.html#task-lists) 标记：
   - 参见 [*upgrade.md*](why-here.md)。该链接需要相应权限，如无权限在以下 New Issue 一步也能看到内容。
1. 每次升级前需要创建相应的 Issues（普通用户可以按以下步骤尝试，注意不要保存即可）：
   1. [*New Issue*](why-here.md)
   1. Description - Choose a template - upgrade
   1. 切换 Write 和 Preview 两个 Tab 可以看到实际的 Markdown 内容和预览效果。
1. 每完成一个步骤将相应 Task 标记为 completed、并准备下一个步骤，直至最终完成。真实案例参见：
   - [*北京环境升级至 4.7*](why-here.md)。注意观察其中的 Activity，另外这些步骤在之后的升级过程会有持续优化。

面对运维工作，人们难免会因为疏忽遗漏而犯错，这是人性，而**从方法、工具层面的解决要远好于泛泛的要求运维人员"认真负责"**，Checklist 就是其中看似平常实质却不简单的一种；这也是 GitLab 自己的标准做法，比如它的变更管理，参见 [change_management.md](https://gitlab.com/gitlab-com/gl-infra/production/-/blob/d34bec7e9713ea65f706151648ae407943e32e57/.gitlab/issue_templates/change_management.md)，以及几个典型的真实案例：

- [Upgrade consul helm chart](https://gitlab.com/gitlab-com/gl-infra/production/-/issues/4228)
- [Test a rollback in production in dry-run mode](https://gitlab.com/gitlab-com/gl-infra/production/-/issues/4206)
- [Upgrade Redis persistent to 6.0 in gprd](https://gitlab.com/gitlab-com/gl-infra/production/-/issues/3677)
- [Delete old postgres data dir to free up space](https://gitlab.com/gitlab-com/gl-infra/production/-/issues/2656)

虽然 Checklist、Workflow、Business Process 各有各的说法，但实际也可以将 Checklist 看作一个最简单、顺序执行的工作流；而和 [Jira workflow](https://confluence.atlassian.com/adminjiraserver0815/working-with-workflows-1050546928.html) 不同，GitLab / GitHub issues 已简化到只有 Open、Closed 两个状态，说不是工作流也可。总之按步骤、按流程走只是表象，我们在以下章节将讨论其本质的不同。

## Design

### Checklist manifesto

在解释 Checklist 时搜索了一下网上的资料，发现[清单革命](https://book.douban.com/subject/10788371/)（副标题: 如何持续、正确、安全地把事情做好）深入讨论了这个问题：

> 清单的4大行事原则：**权力下放、简单至上、人为根本及持续改善**。它们不是僵化的教条，而是实用的支持体系

而这也正是我们想表述的重点！先说明我们在设计 Template 时的思路：

- **Checklist 保障的是内行少出错，而不是说外行照着做也能顶上。**
  - Template 中一般会包含一些注释内容，提示用户阅读更详细的技术文档，如果能读懂说明已基本入门，否则也别勉强。
  - 有了 Checklist，用户也必须养成每做一步标记一下、反复检查 Checklist 的**习惯**。
- **Template 不是必须遵守的教条，其中的步骤是否可以略过、是否需要新增完全由人自主决定，而这就取决于人对问题场景的理解。**
  - 选择某个 Template 和启动某个工作流完全不一样，后者必须按已提前设定的流程配置走，而前者和 Office 模板一样，初始化导入模板仅仅是方便用户快速搭出框架，但是导入后和模板已无任何联系，可以任意修改。
  - 没有强制要求完成 Issue 中的全部 Tasks 才能 Close issue，还是由人判断该 Issue 是否真正完成，参见 GitLab 自己众多的真实案例。
- Checklist 就是简单的任务列表，不存在条件分支、循环等高级功能。
  - **仍然**是做事的人自己判断做了什么情况下做什么或略过什么步骤，甚至在 Issue 中直接删除模板里不必要的步骤。
  - 参见[*武汉环境升级到 4.7*](why-here.md)，通过编辑 Issue 直接手工重复多个小版本的升级步骤。
- **Template 是用户基于自己专业领域的积累来自主配置和管理，不依赖所谓的流程配置管理员。**
  - 虽然有些流程产品也能做到自助配置，但是一个复杂流程的配置、测试、在途流程的迁移等等等等，并不真那么自助。
  - 正因为简单（[Markdown](markdown-tldr.md) 的 Task 语法），所以能够真正自主。
- **持续改进，关键是改进的成本很低**（编辑 Markdown 文件）。
  - 可以对比[*武汉环境升级到 4.7*](why-here.md) 和[*北京环境升级至 4.7*](why-here.md) 的不同。
  - 有权限的用户可以看到 [*upgrade.md*](why-here.md) 的 commit diff。
- **Checklist 所起的作用是帮助提醒做事的人，而不是未按流程走的追责**。
  - 这也是可以任意修改的根本原因。
  - 按流程走的一个经典困境就是：**You stop looking at outcomes and just make sure you're doing the process right.** - Jeff Bezos

还有更多的思路在以下展开，但是到这儿完全可以看出：

> **解决问题的主角毕竟是人而不是清单，是人的主观能动性**

### What not to do

想要将 GitLab issues 用好，**决定不用它做什么也很重要，甚至更重要！**

> Deciding what not to do is as important as deciding what to do - Steve Jobs

- **绝对不能将 Issues 作为统计工作量的工具。**
  - 一个 Issue 里的多个 Tasks 可以是多人参与，不能逼着人在参与前就考虑这算不算我的活。
    - 收费版的 [Multiple Assignees for Issues](https://docs.gitlab.com/12.10/ee/user/project/issues/multiple_assignees_for_issues.html) 貌似符合这个需求，但实际并没有工作量占比的选项，从 Use cases 的说明看出，仍然是为了 "**makes collaboration smoother**"，且 "Once an assignee had their work completed, they would **remove** themselves as assignees"，完全不是为了统计工作量。
    - 而明明是为了完成一个任务一个目标（实现一个功能是一个目标，而测试这个功能并不是目标），为了考核参与者工作量，而生生拆成多个 Issues，这是我们在 Jira 中实际所做的。
  - [*升级武汉测试集群物理机网卡固件*](why-here.md)并没有把这个集群 13 个节点机的任务拆成 13 个 Issues，不要逼着人做这种动作。
  - 随手就解决的事不需要记 Issues。
- **Issues 也不是项目管理工具。**
  - 使用 [Milestones](https://docs.gitlab.com/13.11/ee/user/project/milestones/index.html) 可以区分阶段性的任务（Issues），但是 Closed issues 占总数的百分比代表不了进度。
  - 特定的 [Labels](https://docs.gitlab.com/13.11/ee/user/project/labels.html) 或者 [Issue weight](https://docs.gitlab.com/13.11/ee/user/project/issues/issue_weight.html) 能在一定程度表明任务的复杂度，但是解决的快慢、特别是质量的高低仍然是取决于做这件事的人。
  - 项目的真实进展和风险评估，**取决于**团队负责人对一线任务的真正了解，而不是工具统计出的数字；数字只有在非常有限的条件下比如同团队、同性质的任务才有一定的对比意义，不同团队、不同项目之间的这种数字对比等同于厘米 Vs. 公斤。
  - 同理团队成员的真实贡献也取决于负责人的实际了解和公正评判。
- **Issues 也不是追责工具。**
  - 至少在免费版。具体的一个体现是 Issues 的 Description、Comments 内容可以反复编辑但不能保留历史（标题能够），Lock 可以禁止作者编辑 Comment 但不能禁止编辑 Description。
  - 总的来讲 [View the history of changes to an issue/mr/epic description](https://gitlab.com/gitlab-org/gitlab/-/issues/10103) 是一个很有用的功能。
  - 但实际上这个功能也是在 [12.6](https://docs.gitlab.com/12.10/ee/user/project/issues/issue_data_and_actions.html#description) 才启用。

简单说就是**工具只做工具能做到的事，非要逼工具做更多的事、寄希望于工具而不是人，那最终结果只会是掩耳盗铃**。看看工具自己的定位：

- [GitLab issues](https://docs.gitlab.com/13.11/ee/user/project/issues/)：Use issues to **collaborate** on ideas, solve problems, and plan work. **Share and discuss** proposals with your team and with outside collaborators.
- [GitHub issues](https://guides.github.com/features/issues/)：GitHub's issue tracking is special because of our **focus on collaboration, references**, and excellent text formatting.

### References

- 将 Issues 当作一个知识库，特别是 **Debug 知识库**。
  - 从我们自己的实践上来讲，搜索社区相似现象的 Issues 是我们快速定位问题的绝佳手段，也是解决问题的捷径，比如 [*Multus - Error deleting network*](why-here.md) 搜到的 [Issue](https://github.com/k8snetworkplumbingwg/multus-cni/issues/243) - "tryLoadK8sPodDefaultNetwork() may send unnecessary error message"。
  - 理论上应该将 Issues 遇到的共性问题提炼成知识放到 Wiki，但实际上整理知识的投入**远比**想象中要大、时效性差很多。
  - Issues 应该尽可能**公开**，最多通过 Confidentiality 隐藏特定的敏感 Issues。最近遇到的一个**典型反例**是飞书云服务台，在一个案例中我们做了大量讨论，可是在问题解决关闭后，我作为参与者竟然都找不到这些记录了。
  - 而封闭的结果是人们遇到问题后的**第一反应**不是搜索类似情况并尝试自助解决，而只会依赖。
- [Always start with an issue](https://about.gitlab.com/blog/2016/03/03/start-with-an-issue/)："We say **start** with an issue and **not create** an issue, because one might already exist. Make sure to search in All issues (open and closed) to see if your idea has been proposed already."
  - 重复提交也不是大事，难免会有遗漏，但一个用户提交的 Issues 被反复标记为 Duplicate 还是能说明问题的。
- 公开的 Issues 历史对平台方也是一个**督促**：积累了多少问题、多少问题反复发生……这种透明化对平台方也不是坏事，到底是自己没做好、还是投入不够……
  - 反之一个用户提问题的质量（提问题本身也是有技术含量的）同样也会被记录示众。

## Try it

### Issue templates

在面向用户提供支持时，Issue templates 用于引导用户在创建 Issues 时能够将该问题场景的基本要素**一次性**全部涵盖进去，而无需往返沟通多次后才算刚刚开始；虽然不一定使用 Task lists，但本质上也算"怎样提 Issues" 的 **Checklist**。不要小瞧这么做的意义，仅仅是"[发故障现场的链接](how-to-ask-for-help.md)"这个要求我们在一个支持群里强调了无数遍，仍然时常有人甩一个截图就希望我们解决问题；而除非是耳熟能详，我们是需要翻找现场的上下文来排查线索、缩小范围的，这比指导着远程操作一问一答快很多，同样也是节省我们的精力。所以作为服务提供方（开发团队同样是自己应用的服务提供方），应该归纳总结常见的 Issues 场景并提炼为 Issue templates，因为如以上 [Design](#design) 一节所强调的，成本很低；而普通用户无需关注可直接跳至[下一节](#issues-vs-incidents)。

首先参考以下 GitLab 自己的真实案例：

- [开发类](https://gitlab.com/gitlab-org/gitlab/-/tree/master/.gitlab/issue_templates)
- [运维类](https://gitlab.com/gitlab-com/gl-infra/production/-/blob/master/.gitlab/issue_templates)

更多参考：

- [Issues workflow](https://docs.gitlab.com/ee/development/contributing/issue_workflow.html)

### Issues Vs Incidents

Incident 是一个 [modified issue](https://docs.gitlab.com/13.11/ee/operations/incident_management/incidents.html)：

> They represent a **service disruption or outage** that needs to be restored **urgently**. GitLab provides **tools** for the triage, response, and remediation of incidents.

从管理员的视角，Incidents 和普通的 Issues 还是有很大区别，比如告警、自动化处理、SLA 等等；但是从普通用户看来，两者的属性、界面基本一致，几乎看不出差别，因此必须强调使用的场景：

- **只有**在服务中断、需要紧急处理的情况下才提 Incidents。
  - 但是注意，以 OpenShift 为例，如果运行在其上的某个应用出现了服务中断，而其他应用正常，则不应该给 OpenShift 平台提 Incidents（因为 OpenShift 本身并未中断服务），而是给该应用的服务方提 Incidents（如果也使用 GitLab issues 管理的话）；同时应用方也可以给 OpenShift 提**请求支持**的 Issues，当然真情况紧急的话也**无需**走这些流程、直接找支持人员。
  - 仍以 OpenShift 为例，如果出现了某个问题影响了所有应用，但是并不紧急比如小概率 Bug，这个时候还是给 OpenShift 提 Issues 而非 Incidents。

具体的操作参见下一节。

### Start

我们 [*GitLab*](why-here.md)、[*OpenShift*](why-here.md) 平台服务均通过 GitLab issues 提供服务支持，按以下步骤进行，当然如果是**紧急情况**无需以下流程、直接找支持人员。

1. 判断是否应该提 Issues：
   - 简单的操作类、或者知识介绍等问题不需要提 Issues，可以直接在 IM 群发问，但更建议首先尝试不依赖人的自助解决，如官方文档、社区论坛等。
1. 参见以上 [Issues Vs Incidents](#issues-vs-incidents) 一节的说明：
   1. 确认应该在哪一个 Project 提 Issues / Incidents。
   1. 区分应该提 Issues 还是 Incidents。
1. 搜索：
   - Issues：在 Issues - List 搜索是否已有类似问题存在。
   - Incidents：在 Operations - Incidents 搜索是否已有类似故障报告存在。
     - 也可以在 Issues - List 中查询，但是貌似只能过滤 Label 为 `incident` 的记录，并不能真正按 Incidents 的类型过滤？因此还是使用以上方式为好。
     - 根据不同项目的设置，非项目成员的登录用户不一定能访问 Operations - Incidents 菜单，可以使用 Issues - List 查询，但最好是项目管理员调整，参见 [*GitLab 二级管理*](why-here.md)中的 Operations 相关内容。
1. 如果搜索到参见 [Collaborate](#collaborate) 一节的处理，以下为未搜索到的步骤。

创建 Issue：

1. 类型： Issues 或 Incidents。
1. Title：提炼关键词，一句话说清楚。
1. Description：
   1. Choose a template：参见 [Checklist manifesto](#checklist-manifesto) 一节，模板仅仅是方便用户快速输入，没有任何强制，可以选择任何模板或不用模板，由人决定是否符合当前场景。
   1. Write：
      - 参见 [Issue templates](#issue-templates) 一节的说明，模板是帮助用户**一次性**提供该类问题的基本要素，因此虽然可以在 Write 区域任意调整甚至改得面目全非，但最好有给力的理由；而如果总是这样，确认一下是否选错模板，或者模板设计的确实有问题、请和项目管理员沟通调整。
      - 如果模板要求多个要素，但用户认为其中一个没必要，可以直接删除，但也可以保留该要素并解释为何不需要。
      - 当然也可以增加更多的要素，帮助服务方更快的定位问题解决问题。
      - 用户需要熟悉基本的 [Markdown](markdown-run-tldr.md) 语法。
      - 其中 `<!-- -->` 所包含的内容都是为了提示用户，可以直接删除，但是保留也不影响阅读、可以用 Preivew 查看。
      - 尽量使用文本而非截图。
      - 所引用的日志或程序必须使用 [Code blocks](markdown-run-tldr.md#code) 包含。
      - 尽量使用项目提供的 Label，输入时以 `~` 开头就会自动弹出选择提示。虽然非成员用户并不能给 Issues 打 Label，但是包含 Label 有两个好处，一是视觉上更突出，二是同样内容使用预定义好的 Label 当然要比输入文字更规范。
      - **注意脱敏处理**！比如对于寻求支持的 Issues 可能需要提供日志片段；当然实际上日志本就不应包含敏感信息。
   1. Preview：提交前检查排版是**起码的礼貌**。
1. This issue is confidential：设置该 Issues 仅对 Reporter 或更上级别的成员可见。如无非常必须的情况，比如之上可以用脱敏解决的，请尽量**保持公开**！
1. 其他选项：如果创建者是该项目成员则还有其他更多选项，在以下 [Collaborate](#collaborate) 一节补充。

创建后如果着急可以通知相关人员，在相应支持群发 Issue / Incident 链接，注意 IM 只是一个通知工具或者泛泛的讨论聊天，问题相关的讨论应该在 Issues 进行，方便留存和之后的 Reference。

- Issues：可以 `@` 相关的支持人员。
- Incidents：除支持人员外其他用户也应该关注，证明或证否该 Incident 是否存在。

官方操作文档：

- [Issue Data and Actions](https://docs.gitlab.com/13.11/ee/user/project/issues/issue_data_and_actions.html)

#### Project members

以 [*openshift-ops*](why-here.md) 为例：

1. Assignee
   1. 通知：GitLab 可以和很多 IM 产品集成，但我们目前并没有，主要依靠 IM 中的通知。
   1. 分派：
      - 目前坐在一起的几个成员无需任务池、自动分派的高大上机制，也没有权限差别、全部成员都可以分派给任意成员包括自己，也没有一线转二线三线的升级机制，都知道谁更有经验、疑难问题直接转给谁。
      - 即使到规模扩大、人员众多的情况下，也是通过 [*How to Troubleshoot Before Asking for Help*](why-here.md) 类似自动化、智能化手段来解决，扩展用户自助服务的方案，仍然不需要 Issues 象流水线那样的承接重复的事务性处理工作。
1. Labels：从不同的角度给 Issues 分类。
   - 参见 openshift-ops 预定义的 [*Labels*](why-here.md)。
     - 所谓不同的角度如 [kind](https://github.com/kubernetes/kubernetes/labels?q=kind) / [type](https://github.com/spring-projects/spring-boot/labels?q=type)、[triage](https://github.com/kubernetes/kubernetes/labels?q=triage) / [status](https://github.com/spring-projects/spring-boot/labels?q=status)、[area](https://github.com/kubernetes/kubernetes/labels?q=area) / [theme](https://github.com/spring-projects/spring-boot/labels?q=theme) 等这些并没有一个权威统一的规范，要直接解释其实也很难说清晰，看其具体的分类选项反而容易理解。
     - 由于 OpenShift 基于 Kubernetes，以及个人偏好，更倾向于使用 Kubernetes 而非 Spring 的 Labels 分类。
   - 并不是一定要分类，最后的统计精确度**并不重要**，仍然是方便做事的人处理、查找等。
   - 熟练情况下使用 `/label ~label1 ~label2` 等 [quick actions](https://docs.gitlab.com/13.11/ee/user/project/quick_actions.html) 要远比用鼠标点点点流畅，且两个 Labels 对于鼠标点击会是两次记录、而前者只有一个记录。
   - 常见场景：
     - 用户提供的信息不够，是需要 `/label ~"triage/needs-information"` 的，补充后可以 `/unlabel` 或 `/relabel`。
1. 之后就是通过 Comment 的真正互动并解决问题 Close。

#### Users

如 [Start](#start) 一节所述，如果已发现类似情况：

- Issues：
  - 如果该 Issue 已提供解决方案，尝试照此自助解决。
  - 如果没有解决，看情况在该 Issue 补充评论、或者提新 Issue 但关联所找到的类似 Issues。
- Incidents：
  - 在该 Incident 点击 Thumbsup 图标 Upvote，表示确认自己遇到同类问题。Upvotes 越多则越说明是普遍性的平台问题，方便服务方快速定位。
  - 还可以通过 Comment 补充更详细的信息，但是如果仅仅是"+1"就不要浪费评论了，Upvote 就是这个意思。
  - 如果其他用户在 IM 群收到 Incident 的通知，麻烦关注一下，遇到同样故障 Upvote，但没有遇到故障也应该点击 Thumbsdown 图标表示 Downvote，这同样是在给服务方提供排查线索。

之后就是通过 Comment 互动直到问题解决。
