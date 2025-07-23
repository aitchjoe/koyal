---
title: CI/CD TLDR
---

基本的 CI/CD 概念解读，面向全体 IT 人员。

## General

- CI：持续集成，最狭义的技术定义就是从源码生成制品并保存到制品库的过程，这个过程通常在源码提交后自动触发。
- CD：有持续部署（Deployment）和持续交付（Delivery）两种说法，先别绕进去就当作一回事，狭义理解就是将上一步 CI 生成的制品部署到运行环境。
- CI/CD 的一个重点在于"**持续**"，也就是短周期和高频次，通过反馈和迭代让产品持续提升完善（敏捷）。
- 开发人员需要尽早、频繁的提交代码变更，这样才谈得上后续的 CI/CD，因此对于**开发方面**如需求拆分、开发架构、业务应用架构甚至 Git 工作流等等，都有很高要求。
- 而实现高频次的 CI/CD，只可能依赖相应的自动化工具及平台，但是**要实现高频次且有质量保障的 CI/CD，则必须依赖足够的自动化测试**。
- 为了自动化的实现以上 CI/CD 目标，我们还需要相应的 CI/CD 工具或平台，因此请注意通过上下文区分，比如我们提及 CI 时是指 CI 目标任务还是 CI 工具。
- CI 工具：典型如 Jenkins 或 GitLab CI，在 CD 需求出现以前，无论产品或技术在名称中都会标榜 CI，但本质上 CI 工具就是脚本自动化，可以实现任何目标包括 CD，因此注意**一般 CI 工具都能实现 CI 或 CD 目标**。
  - 所以在 GitLab CI 中，不用惊讶我们是基于 `.gitlab-ci.yml` 这个配置文件来实现 CD。
- CD 工具：如上所述，传统的 CI 工具都能完成 CD 任务，包括针对 CD 场景做了一定的优化扩展；但是到云原生时代并考虑到 GitOps，也有了专门的 CD 工具如 Argo CD。
- 按我们当前的实施程度，包括对用户 CI Job 的行为分析，以及自动化测试的匮乏等等，**只能说我们用上了 CI/CD 工具，但远未达到 CI/CD 的目标**。

## Performance at Local Vs in CI

很多刚接触 CI 的用户经常会问为什么在本地编译很快的应用提交到 CI 环境慢很多？这应该算 CI 的入门问题了。

- 虽然都是编译构建，但是本地开发环境和 CI 环境的使用场景是有很大区别的。
- 本地开发的重点是用户能够**尽快看到对代码修改的反馈**，因此用足了缓存、增量编译、热加载等等各种机制。
  - 如果出现某些意外，比如改的代码并没有反应到运行效果上，需要开发人员自己判断到底是改的不对还是编译出了问题，如果是后者就可以尝试手工清理缓存等等。
- CI 首先必须保证的是持续集成的**稳定性**，具体说就是同一个源码 Commit 执行多次必须是一样的结果，不能说这次成功、下次却失败了，因此 CI 选择的是更保守但会慢的方案。
  - 都不说应用的增量编译，即使 Java 依赖包的增量下载都有可能出 Bug（实际遇到过），因此 CI 对缓存的使用很谨慎。
- 还有一个重点是自动化测试，这通常是耗时最长的部分，**CI 并不是运行越快就表示越好**（更可能的是完全没测试），所以如果真正考虑了这一部分，本地和 CI 在其他方面的速度差异自然就不是突出的问题了。
  - 一个道理，CI 也不是 100% 成功就表示最好。
- 而且用户提交了代码并不代表 CI 平台就会马上执行，很可能还要排队等候（Pending）。
- 在日常的使用场景，用户提交代码之前就**应该**在本地验证功能基本无误，也就不需要紧等着看 CI 的结果了，所以真的不用太关心这几分钟的差别、继续自己的其他工作就好。
- 以上的这些道理并不是说就不关心速度，但速度也绝**不是 CI 考量的第一优先级**。

## Defining in CI Server Vs in SCM

这个主题讨论的是 CI 配置应该保存在 CI 服务端还是应用源代码？可能有些 IT 新人会奇怪为什么有这个问题，但在 CI 早期阶段这些配置就是保存在前者的。

- 仍然从 CI 的稳定性说起，同一个源码 Commit 执行多次必须是一样的结果，很显然同一份源码不同的 CI 配置完全有可能产生差别。
- GitLab 在 [2015](https://about.gitlab.com/blog/2015/05/06/why-were-replacing-gitlab-ci-jobs-with-gitlab-ci-dot-yml/) 年就"replacing GitLab CI jobs with .gitlab-ci.yml"。
- [Jenkins](https://www.jenkins.io/doc/book/pipeline/jenkinsfile/)："it's generally considered a **best practice** to create a Jenkinsfile and check the file into the source control repository"。
  - 和 GitLab 彻底废弃旧方式不同的是 Jenkins 还保留了将配置放在服务端，除了历史原因外也可能 Jenkins 毕竟定位是 Automation Server、不只针对有源码的 CI 场景。
- 以上 GitLab 链接详细讨论了两种方案的对比，实际上跟源码走早已是业界主流专业的做法了，在方案选型这件事上，其实应该是**不按最佳实践主流做法走的人必须拿出非常强大的理由来解释**。
- 不止 CI 配置，如工具脚本、数据库 SQL、Markdown 文档等等都随源码一起管理同样是业界主流做法，[十二因子](cn-12factor.md)也强调了这一点。

## Run in VM Vs in Container

CI 平台使用 VM 环境或容器环境运行 CI 任务的区别。

- 在 VM 时代要准备 CI 环境是有很大局限的，以 JDK 为例几乎不可能准备 8u5、8u11、8u20……8u101……8u311 如此众多的小版本，只可能是笼统的 JDK 6 / 8 / 11，而各种工具、各种版本混在有限的 VM 必然也有更多问题。
  - 如果现有的 CI 环境不符合用户要求，用户也只能是静等服务方准备。
- 而容器是 CI 工具的最大利好，因为所运行的容器 CI 环境完全由**用户指定、按需创建**，任意软件、任意版本、任意搭配，不受制于任何外部约束，用户按照自己的节奏做大小版本的升级，当然也就不存在等服务方准备环境了。
- Jenkins [Blog](https://www.jenkins.io/blog/2021/12/08/containers-as-build-agents/) 表达了同样的观点："to define the tools and the specific versions of those tools that you want to use in your pipeline so that those items are **not being mandated or managed by others**"。
- 当然不是说容器模式就没问题，由于容器 CI 环境都是每次新建，就决定了这个过程必然会慢于已准备好的 VM 环境、缓存共享也更麻烦等等，但相比之上的好处，在 CI 场景完全可以接受，这个在以上 Performance 一节已有讨论。
- 一个 CI 任务通常会拆分成多个子任务，比如云原生 Java 应用的一次 CI 会先生成 Jar、然后根据 Jar 生成容器镜像，这个工作能在同一个 VM 完成，但是由于容器不像 VM 混部了多个工具，通常需要拆分成多个容器来完成，因此速度上会更慢。
- 注意以上的差别本质上不在于 VM 和容器本身、而是对两者的**调度编排**上，如果能由用户指定按需创建 VM 一样可行，事实上 GitHub Action（可以理解为 GitHub 版本的 CI）使用的就是 VM，但它同样也会存在容器方案的问题。
- 另外注意 CI 服务器的部署环境和每次 CI 任务运行的环境是两码事，以上讨论的是后者，完全可以 CI 服务器部署在 VM、CI 任务运行在容器，后者也是典型的 Serverless 场景。

## Declarative Vs Imperative

虽然本质上 CI/CD 就是跑自动化脚本，但毕竟需要针对 CI/CD 的使用场景做一定的优化，我们以 GitLab CI 为例比较两种方式：

```
pdf1:
  script: xelatex mycv.tex
  artifacts:
    paths:
      - mycv.pdf
    expire_in: 1 week

pdf2:
  script: xelatex mycv.tex && curl -T mycv.pdf ftp://ftp.example.com --user user:secret
```

- 即使不了解 GitLab CI 的语法应该也能大概看懂以上代码吧，都是将 Tex 文件转为 PDF 并保存。
- 显然前者是声明式而需要操心命令细节的后者是命令式，而且后者也做不到自动过期这个需求；当然声明式是有前提的，必须是 CI 产品已设计实现相应功能。
- 由于 CI 或者说自动化的场景非常丰富，绝无可能做到百分百声明式，实际上主要功能仍是基于用户定制的脚本。所以这个主题要比较的是哪个产品在 CI 常见场景能够更声明式一些、用户理解起来更直观一些、使用更简单一些、配置的行数更少一些等等等等，这个从 [Keywords](https://archives.docs.gitlab.com/14.8/ee/ci/yaml/index.html) 和 [Predefined variables](https://archives.docs.gitlab.com/14.8/ee/ci/variables/predefined_variables.html) 的丰富程度能有一定体现。
