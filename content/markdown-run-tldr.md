---
title: Markdown Run TLDR
---

Markdown 入门实操，面向技术写作者，请首先阅读概念向的 [Markdown TLDR](markdown-tldr.md)。以下除了 Markdown 语法也会强调一些技术写作上的**最佳实践**。

## Code

- 首先提 Code 是因为本文要演示 Markdown 语法的效果，如果不将 Markdown 语法标注为代码，则 GitLab 或飞书默认看到的都是最终效果，当然也可以直接查看本文的"[*源码*](why-here.md)"。
- 代码语法分两种，一个是将嵌入到普通文本中的某个单词或句子标识为代码，通过前后反引号包围来确定，比如`` `this` `` 的代码效果是 `this`；注意显示有反引号的 "this" 使用了包含代码语法的代码语法，这个是无限俄罗斯套娃，先别管这种复杂情况。
- 还有一种代码语法是针对多行代码，叫代码块，通过上下行连续三个反引号来包围，比如：
  ````
  ```
  {
    "name": "joe"
  }
  ```
  ````
  它的实际效果（当然上面又是俄罗斯套娃）：
  ```
  {
    "name": "joe"
  }
  ```
  注意以上语法属于扩展语法而非[基本语法](https://daringfireball.net/projects/markdown/syntax)，实际叫围栏式代码块，但基本上全生态圈都支持，也算最通用的了。
- 很多网站会针对多行代码提供互动功能，当鼠标移到代码块时自动浮现一个拷贝按钮、供读者一键获取，因此即使是单行代码也可以考虑使用代码块语法。
- **除了开发语言代码，运维配置文件比如 JSON / YAML、数据库 SQL、日志片段、Markdown 纯文本等等，都是代码。**
- 即使不是代码，比如帮助文档中提示某某项请输入 `100`，这种用法显示效果也很好。

## Heading

本文此处以上的标题语法：

```
# Markdown Run TLDR
......

## Code
......

## Heading
```

- 通过一到七个连续的 `#` 来标识一到七级标题。
- 除了内容上区分结构和层次，通过网站呈现的标题还带有导航性质，一是文内导航的标题锚点，二是自动生成 TOC（Table of Contents）目录。
  - 但并不是所有网站都支持这种功能，大家可以对比这个同一文档在 [*GitLab*](why-here.md) 和[飞书](https://tkhome.feishu.cn/wiki/HPK4wrkpOimIpCkoh7Ac7vsQn0e)（包括网页版和 APP）的效果。
  - 这个功能很重要，比如接收到带锚点 URL 的用户打开后能**直接定位**到该章节，而不是从头找起。
- 中文标题所生成的 HTML 锚点在传递时会转 ASCII 码，导致链接又长又不直观。
- 如果是英文注意首单词不要全小写，是 `DevOps` 不是 `Devops`，当然正文里一样要注意只不过标题更显眼，另外中英文混排建议加空格。

## Emphasis

- `**粗体**`：**粗体**
- `*斜体*`：*斜体*
- `***粗斜体***`：***粗斜体***
- 既然是强调性质，那么文中只应该小比例出现而**不要滥用**，否则仍然失去了重点。

## Link

-  链接基本语法为：`[显示文本](URL)`。
   - `[Daring Fireball](https://daringfireball.net/projects/markdown/syntax)`：[Daring Fireball](https://daringfireball.net/projects/markdown/syntax)
- **知识一定要串联，多用链接。**
- 引用地址尽量带上**版本信息**，如 [github.com/.../**v0.15.6**/.../run.go](https://github.com/paketo-buildpacks/nginx/blob/v0.15.6/cmd/configure/internal/run.go#L19) 或 [**v1-27**.docs.kubernetes.io/...](https://v1-27.docs.kubernetes.io/docs/concepts/overview/working-with-objects/object-management/#declarative-object-configuration)，否则过一段时间后可能指向的就不是你期望的内容了。
  - Spring 文档会保留所有版本，但 Kubernetes 会移除较旧的版本，但即使这样仍然建议带上版本。
  - 源码使用正式版的 Tag，否则也可能被移除。
- 引用地址尽量带上**文内锚点**，示例如上，像 GitHub / GitLab 都会提供源码中每一行甚至多行的锚点，更不用说文档了，这样读者能直接定位到你期望他们阅读的部分。

## Blockquote

- 引用也分单行或多行引用。
- 单行引用，比如 Russell L. Ackoff 的：
  ```
  > We fail more often because we solve the wrong problem than because we get the wrong solution to the right problem.
  ```
  > We fail more often because we solve the wrong problem than because we get the wrong solution to the right problem.
- 多行引用，比如 Niels Pflaeging 的 X：
  ```
  > Organizations today do not actually suffer from #tech problems - such as #digitalization.
  >
  > They suffer from **#socialtech problems & barriers** - such as #management, #meetings, #decisionmaking, #innovation, #change.
  ```
  注意以上必须插入空行，否则会合并成一行：
  > Organizations today do not actually suffer from #tech problems - such as #digitalization.
  >
  > They suffer from **#socialtech problems & barriers** - such as #management, #meetings, #decisionmaking, #innovation, #change.
- 引用可以嵌套或者和别的元素混用，比如以上例子中的粗体。
- **务必标注引用来源**，尽量使用以上链接语法，否则也需文字说明。
- 没有在一行中标注某部分为引用的语法，但也应该用双引号包围**示意**为引用内容。
- 不要引入过多内容，同时也可以在引用部分使用强调语法**突出重点**，但有个问题就是混淆了到底是原作者强调还是引用者强调？

## List

- 在标题之下，通过列表、缩进进一步**区分层次**。
- 分为无序列表和有序列表，可以嵌套及有序无序混用。
- 无序，以下引自 Programming Wisdom 的 X：
  ```
  - Done On Time
  - Done On Budget
  - Done Properly
  - Pick two
  ```
  - Done On Time
  - Done On Budget
  - Done Properly
  - Pick two
- 有序，以下引自 James Hollingshead 的 X：
  ```
  1. get losts of things wrong.
  1. learn from the mistakes.
     - Most people forget step 2.
  ```
  注意以上的数字随意，展现时会自动递增，如果特意指定行数、插入一行还得调整之下的；
  1. get losts of things wrong.
  1. learn from the mistakes.
     - Most people forget step 2.
- 列表也可以和别的元素混用：
  ````
  1. ```
     ocw project
     ```
  1. ```
     ocw get pods
     ```
  ````
  通常多行代码块的反引号都是从行首开始，但在列表中需空出相应的格数：
  1. ```
     ocw project
     ```
  1. ```
     ocw get pods
     ```
