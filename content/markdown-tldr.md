---
title: Markdown TLDR
---

Markdown 基本介绍，面向技术写作者，具体入门实操参见 [Markdown Run TLDR](markdown-run-tldr.md)。

## General

- Markdown 是业界技术写作最主流的选项，开源项目的 README.md 基本上是标配。
- Markdown 为纯文本写作，通过一些非常直观的标记进行格式化排版，如`**粗体**`的效果就是**粗体**，即使没有工具渲染成最终效果，看纯文本基本也没妨碍，大家可以直接查看本文的"[*源码*](why-here.md)"。
- 对于 Markdown，简洁高效、易写易读的优先级要远高于五花八门的功能，"逼着"人们注重**内容胜过形式**。
- Markdown 的生态极其丰富，GitHub / GitLab、Wikipedia、飞书等等等等都能够自动渲染 Markdown 格式的文件（微信竟然不支持👎）。
- Markdown 没有统一标准，有很多变种或扩展，但基本语法都是相通的且足够实用；不过反过来讲，如果都等着出一个统一标准，很可能现在 Markdown 就已经消失了。
- 了解 Markdown 的标题、强调（黑体、斜体等）、列表（有序、无序）、引用、代码（单行、多行）、链接等不到十个[语法](markdown-run-tldr.md)，就足够开始写作了，作为 IT 业者无论是否做技术都可以认为**学习成本约等于零**；当然，文章的条理逻辑是任何一个工具都提供不了的。
- 有所见即所得的工具支持 Markdown 创作，但个人觉得没必要，Vi 最高。
- 纯文本非常适合使用 Git 管理版本历史、版本变更对比也更直观，这是 Markdown 最开始在开发项目中兴起、和源码一并管理的重要原因；但即使不是开发项目，一样也可以将文档用 Git 管理起来，这个我们在 [Git TLDR](git-tldr.md) 中讨论。
- 如果想完整学习 Markdown 且不局限于基本功能，可以考虑 [GitHub Flavored Markdown](https://docs.github.com/en/get-started/writing-on-github/getting-started-with-writing-and-formatting-on-github/about-writing-and-formatting-on-github)，至少在扩展语法中它应该是使用最广泛的，而且我们的 GitLab 基本也是跟着 GitHub 在走。

## Techdoc

技术文档的特性或者说技术文档不考虑 Word 之类编辑器的原因：

- 技术文档一个最基本的需求就是对代码片段的引用，尝试了 Office 2021 未找到插入方式，从目前搜索"[microsoft word insert code block](https://cn.bing.com/search?q=microsoft+word+insert+code+block&ensearch=1)" 的结果来看都是间接实现的？
- 注意 Markdown 里的代码语法并不是说只针对开发语言的代码，运维的配置文件、数据库的 SQL、各种日志等等也都是代码，所以**对 Markdown 的使用并不局限在开发人员**。
- 而技术文档在网站呈现时还会有一些互动场景，典型的如当鼠标移到代码块时应该浮现一个拷贝按钮，供读者一键获取。
- 再比如一个 Debug 文档，可能引用的日志有二十行，说明原因就两行，而实际上后者才是关键内容，但读者的视觉重心很可能落在行数多的部分；因此另一个常见需求就是能提供折叠功能，一些细节性但占面积大的内容比如日志、截图等默认折叠，让读者一眼看到重点，但想深入查看细节时又能动态展开。
- 实际上还有一个产品很符合以上的诉求，就是 Confluence，但它最大的问题是格式是私有的、不能脱离 Confluence 网站；比如现在想把 Confluence Page 转移到飞书，那么只能是导出为 Word / PDF 且无法保留完整的格式信息、甚至错乱了。因此虽然 Confluence 是本人引入并[*推广*](why-here.md)的，但是从 2020 年开始就全部转向了 Markdown 写作并保存在 GitLab，当然现在也能轻松的同步到飞书知识库。
- 虽然我们现在大量使用飞书，但也要**警惕**尽量不要使用飞书私有、或者貌似通用的微软私有格式进知识创作，开放标准非常重要，以下有[飞书](#feishu)一节专门讨论。
- 总之考虑到二进制的私有格式、对于轻量级写作显得过于笨重的编辑器，虽然我们承认所提供的功能更齐全、甚至某些功能也很实用诱人，但我们还是选择了 Markdown。

## Markdown Vs AsciiDoc

由于通用 Markdown 只能实现常见的需求，当用户深度使用时就无法满意了，比如 2013 年 [Spring's Getting Started Guides migrated to Asciidoctor](https://spring.io/blog/2013/12/13/spring-s-getting-started-guides-migrated-to-asciidoctor/)。可以将 AsciiDoc 理解为增强的、统一标准的另一个 Markdown，以下是本人在使用 AsciiDoc 过程中觉得重要、但通用 Markdown 不支持的功能：

- [Collapsible Blocks](https://docs.asciidoctor.org/asciidoc/latest/blocks/collapsible/)：以上提及的可折叠功能。
- [Table of Contents](https://docs.asciidoctor.org/asciidoc/latest/toc/)：虽然很多网站在渲染 Markdown 时可自动根据标题项生成 TOC（飞书没有👎），但是不能控制标题层数，因为一般 Level 5 - 7 的是没必要生成到 TOC 的，而 Asciidoctor 是可以[定制](https://docs.asciidoctor.org/asciidoc/latest/toc/levels/)。
- [Assign Custom IDs and Reference Text](https://docs.asciidoctor.org/asciidoc/latest/sections/custom-ids/)：文内引用时 Markdown 会根据标题内容生成 HTML 锚点，但如果是中文会转为 ASCII 码，这样传递的链接又长又不直观，是的，URL 用户友好也很重要，这也是我标题不得不用英文的原因，而 AsciiDoc 可以使用中文标题但定制英文锚点。
- [Admonitions](https://docs.asciidoctor.org/asciidoc/latest/blocks/admonitions/)：虽然可以用粗体（Markdown 基本语法中不能指定颜色）强调给用户的提示警告，但还是这个专用语法效果更好。

当然 AsciiDoc 也有它的问题：

- 我的这篇 [*adoc*](why-here.md) 文档在 GitLab 能正常展现，但是上传到飞书知识库后无法渲染、甚至不能识别为纯文本而只能下载，也就是说生态仍然无法和 Markdown 相比。
- 链接语法较丑陋，它是较长 URL 在前、较短标题在后，如果是纯文本阅读没有相反的 Markdown 舒服，当然也可能是个人习惯问题。
- 语法复杂很多，学习成本较 Markdown 高，但仍然是比较直观的格式，还是值得尝试。

## Graph

在说服伙伴使用 Markdown 时的一个很大阻碍是 Markdown 插图很不方便。

- Markdown 有引用图片的语法，而图片文件是分开保存的，GitLab 展现没问题，但将 Markdown 和图片文件上传到飞书个人空间的同一位置后，飞书不能正确展示图片，这应该是上传图片文件的名称变成了飞书地址。因为引用图片是 Markdown 的基本语法，所以可以认为是飞书的问题，但总归影响了 Markdown 的使用。
- 相比 Word 在同一个文件嵌入文本和图片，Markdown 将两者分开的做法，虽然在保存、传递时不够方便，但不会受制于私有程序、最不济可以阅读文本，所以至少这两个方案没有绝对的优劣。
- 如果是流程图，[GitLab](https://docs.gitlab.com/ee/user/markdown.html#diagrams-and-flowcharts) 等支持使用文本描述、渲染时展现为图形的方式，可以认为是 Diagram / Flowchart 界的 Markdown，当然这个学习成本就比较高了。
- 还有一种貌似原始、但对于简单场景也很实用的做法，就是文本作图，这个可以自己手工画，但也有 [ASCIIFlow](https://asciiflow.com) 这样的工具帮助完成。以下是我们在[*真实文档*](why-here.md)中的图例：
  ```
  +-------------+                         +-------------------------+
  | OCP Cluster |        +-------+        |       O11y Platform     |
  |             | -----> | Kafka | -----> |                         |
  |  Collector  |        +-------+        | Aggregator ---> Backend |
  +-------------+                         +-------------------------+
  ```
- 事实上业界技术文档使用图片已经**非常节制**了，主要是极少量的架构图或流程图，反观我们大量使用的是操作示例的截图。
  - 我们承认这种截图对读者更加友好，但是对于作者，截图特别是重现复杂步骤且保持一致性的截图，这个的工作量比想象中大很多，再考虑考虑比如升级后界面又有了调整、哪怕仅仅是布局方面的？如果读者仍不理解，仍坚持"我不管，我就要直观的截图"，那么请看下一条。
  - 以我的实际观察，文档操作截图的更新频率要**远低于**文字。
- 总之使用 Markdown 在图像方面确实不够方便简单，但请多考虑一下现实、多考虑一下是否真的那么必须？

## Feishu

- 现在我们公司主要使用的是飞书文档及知识库，因此我们主要关注飞书对 Markdown 的支持情况。
- 飞书能自动渲染展现 Markdown 文件，大体无误，但仍不能跟 GitHub / GitLab 相比。
  - 飞书不能根据内容标题自动生成 TOC（Table of Contents，目录），而后两者可以。注意当前 [*gitlab.it.example.com*](why-here.md)（v16.2.7-jh）有 Bug，大家可以在 [*gitlab.example.com*](why-here.md)（14.8.2）查看 Markdown 文件的 TOC 效果。
  - Front Matter 是 Markdown 文件头的 YAML 元数据，虽然不是基础语法但也很流行，飞书不能正常展现但后两者没问题。
- 像 TOC 这种功能，明显不是技术上的难题，我们**非常怀疑**飞书私心并不鼓励大家拥抱 Markdown 等开放格式，以下的区别（基于飞书 Windows 版 7.46.6）都可以算证明。
  - 在飞书的个人文档空间，可以"上传文件"或者"导入在线文档"，对于同一个 Markdown 文件，后者能自动生成 TOC、点击 TOC 也能方便跳转，但前者不行。
  - 导入的在线文档，再下载时只能保存为 Word 或 PDF 文件，已经缺失了原始 Markdown 文件，而上传的文件当然可以再次下载。
  - 在飞书文档空间"导入在线文档"时可以选择"Markdown"类型，而在飞书知识库的导入缺失了这个类型，但 Word、PPT 等都可以。
  - [URL 锚点](https://developer.mozilla.org/en-US/docs/Web/URI/Reference/Fragment)是一个非常有用的功能，分享长文链接时能直接跳到希望对方关注的章节，但一个最简单的文内锚点链接（比如以上的 [Graph](#graph)，在 GitLab 网页的链接为文档 URL 加上 `#graph`）被飞书转成了另一个域名 `internal-api-drive-stream.feishu.cn` 的 URL（在飞书右键复制链接就可以查看）、且不能直接在浏览器打开，也就是说无法分享，这个设计简直可以说是恶意的。
  - 还有一些细节，比如 Markdown 未提供拷贝代码块的快捷按钮，但在线文档有。
- 因为个人很抗拒以上飞书的方式，坚持 Markdown + Git 的做法，但由于 GitLab 只能内部访问、也不方便通过手机阅读，因此只将飞书当成中转站、方便分享。
  - 在飞书知识库 Koayl Handbook 看到的所有文档包括本文，均位于 GitLab 的 [*koyal-handbook*](why-here.md) 项目。
  - 通过自动化[*脚本*](why-here.md)生成 TOC、移除 Front Matter，方便在飞书阅读。
    - 但生成的 TOC 只能放在内容开始处，和导入在线文档的左侧菜单相比仍不方便用户阅读。
    - 还有一个更重要的功能，将 GitLab 相对链接转为飞书链接，而以前是手工添加两个链接，非常麻烦也影响阅读。

---
Welcome to the Markdown world, from PPT.
