---
title: LeetCode 和模糊测试
---

## LeetCode

对于我来讲，刷 LeetCode 的困难有以下几种。

### 概念

虽然对时间复杂度、空间复杂度也不算外行，但是能马上反应过来自己所写的程序算 O(1)、O(log n)、O(n) 还是在这次刷题中锻炼出来的，粗暴解题也是直奔空间换时间而去。

显然题目也不会多解释比如 BST 的特性，而会将"左子树小于当前节点小于右子树"当作常识，了解这才是解题的第一步，至于前 / 中 / 后序遍历等等等等也是刚补的课。

如果说以上内容还有些模糊的概念，那 DP（动态规划）什么的就完全没印象了，只能算从头学习。

### 编程实现

除了少量场景，我们在真实业务开发时对算法的使用基本就是通过现成的库来实现，我不会自己去写 FIFO 的 Queue 或者 LIFO 的 Stack，但是在这次刷题准备时已经练得烂熟：

```
// Queue
q := []int{}
// Push
q = append(q, num)
// Pop
num := q[0]
q = q[1:]
```

如果说上面的还算简单，要熟练掌握 DFS / BFS / Backtracking 等等的基本编程套路，还是花了不少时间，注意这还没到用这些算法去解决问题。

### 问题理解及测试

日常刷题时，LeetCode 对于提交失败的程序会提示具体未通过的测试样例和预期值，实际上到这一步相对来讲就容易解决 Bug 了。因此快速理解问题、快速构建问题的典型分类、厘清边界状况，就是成功实现程序的前提条件，当然，也是导致失败的拦路虎:(

这里重点想讲的是在这个场景 LeetCode 屏蔽了不同语言的差别，全部用同一个界面和数据格式输入测试案例并验证；但如果不用 LeetCode 的 Web UI、而是在自己的开发环境解题，我基本是下面的步骤：

1. 将题目中的示例 Hard coding 到 solution.go。
1. 编写测试程序 solution_test.go 并验证通过（`go test .`）。
1. 在 solution.go 中实现真正的算法并验证通过。
1. 设计更多测试案例添加到 solution_test.go，如验证失败则回到上一步。

以上步骤才是真实的开发场景，所以能方便快速的构建测试并验证也是敏捷开发的重要部分。转到语言工具，以个人较熟悉的 Java、JS 对比，我觉得 Go 语言包括内置的 [lightweight test framework](https://golang.google.cn/doc/code#Testing) 要轻松许多，特别是 Vs Java（虽然比较开发语言政治不正确）。

### 算法

基于 LeetCode 的使用场景，必然不会出现真实复杂的问题，因此使用基本的开发语言元素（if/else、for/while、array/map）就能顺利解题；所以我最初借这次刷题想熟悉 Go 语言的目标也没完全达到，比如 Go 中最重要的 [chan](https://golang.google.cn/doc/effective_go#concurrency) 一点儿都没用到，嗯，其实我妄图用 chan 将程序从 O(n^2) 粗暴降到 O(n)，当然这是不可能的。

算法这一领域对个人来讲是最困难的部分，所以在这一节远没到提炼总结的时候，在此只想说一个学习二分查找法时的心得重点。虽然二分查找的用途很容易理解，但是真正自己动手去实现时对边界的处理却始终不到位、就是说没信心如何写必然通过，也在网上搜到不少说明看完貌似理解、一做又原形毕露，正如[【二分查找】详细图解](https://blog.csdn.net/qq_45978890/article/details/116094046)所述：

> 二分法的思想十分容易理解，但是二分法边界处理问题大多数人都是记忆模板，忘记模板后处理边界就一团乱（👁:“我懂了”，✋:"你懂个🔨"）

直到我在力扣找到一种做法（很可惜丢失了链接，重新找了很久，应该是作者在某个二分查找问题的一个题解中的评论），具体实现类似我模仿着在 [First Bad Version](https://leetcode.cn/problems/first-bad-version/) 中的解法：

```
func firstBadVersion(n int) int {
    left, right := 1, n
    for right - left > 5 {
        mid := left + (right - left) / 2
        if isBadVersion(mid) {
            right = mid
        } else {
            left = mid
        }
    }
    for i := left; i <= right; i++ {
        if isBadVersion(i) {
            return i
        }
    }
    return -1
}
```

不熟悉 Go 可以当伪码看，总之就是根据 left、right 边界获得 mid 值、然后根据 mid 的情况来调整左右边界快速逼近。而上面的做法通过拆分出两个 `for` 循环回避了边界相遇时的麻烦：

1. `for right - left > 5` 完全不考虑边界相遇时如何处理，只需快速逼近到一小段距离（可以是 5 也可以是 3、4、6、7）然后就转到下一步。
1. `for i := left; i <= right; i++` 就是纯粹的遍历了，因此处理逻辑也非常简单直观；而 O(5) 和 O(n) 完全不是一回事，前者是一个常量约等于 0、后者是变量才决定了算法的性能。

由于分成了两步、每一步的处理逻辑就明确了很多（比如以下提及的疑问）；而即使分成了两步，O(log n) + O(5) 也约等于常规二分法的 O(log n)。

这样做肯定会有人认为是画蛇添足、觉得一步到位也很清晰啊，在现场同样有人评论说这种做法是 Hack 式的（明显不是赞赏、而是认为比较低级），如果说在追求优雅算法的 LeetCode 场景还有那么一点道理，但是在真正的工程实现上，我觉得这种做法是非常优雅的 Hack（褒义）！如果不能理解这个判断可以看这个问题[官方题解](https://leetcode.cn/problems/first-bad-version/solution/di-yi-ge-cuo-wu-de-ban-ben-by-leetcode-s-pf8h/)下的评论（或者搜其他[二分查找](https://leetcode.cn/problemset/all/?topicSlugs=binary-search&page=1&difficulty=EASY)题的求助）：

> 有点迷茫什么是否取等号什么时候不取等号了
>
> 对于二分法一直有一个疑问，就是边界值问题；第1个，终止循环条件left<right还是小于等于；第二个，left=mid+1这里要不要加1，同样right=mid-1要不要减1
>
> 为什么最后是return left不能是return right啊
>
> 为什么，right = mid，而不是right = mid - 1呢

总之个人认为这是我在刷题时学到的最有价值的算法实现！当然在真实开发中甚至连这种方法都要避免（最终是不要自己实现）。另外还有一个是在 [Python solution from a beginner (some easy steps to follow to solve dp)](https://leetcode.com/problems/min-cost-climbing-stairs/discuss/657490/Python-solution-from-a-beginner-(some-easy-steps-to-follow-to-solve-dp)) 中 pccisme 的讲解和提炼对我理解 DP 帮助甚大，这里就不详述了。

## Bug 和模糊测试

### Bug

在我刷第三题 [1493. Longest Subarray of 1's After Deleting One Element](https://leetcode.com/problems/longest-subarray-of-1s-after-deleting-one-element/) 时就发现了 LeetCode 的问题，这是我的 [AC](https://leetcode.com/submissions/detail/807576266/)（Accepted Code）：

> 74 / 74 test cases passed.

但是在 LeetCode 的 [Console](https://leetcode.com/problems/longest-subarray-of-1s-after-deleting-one-element/) 使用以下 Testcase（`[1,1,0]`） 验证我的程序时：

```
Wrong Answer Runtime: 2 ms
Your input [1,1,0]
Output 1
Expected 2
```

至于为什么会出现两种不同结果？因为在提交我的代码时，LeetCode 是使用预先准备好的诸多 Testcase 来检查程序，而在 Console 中使用的是我自己设计但未包含在 LeetCode 的 Testcase、通过同时运行官方算法和我的程序并对比输出结果、不一致则判错（当然理论上也存在我对它错的可能）。

所以**对 LeetCode 解题场景的 Bug 定义并不是官方算法是否有漏洞，而是说 LeetCode 未提供足够覆盖度的 Testcase、导致了有缺陷的算法被 Accepted**。虽然不了解背后的实际运作，但是能大致推测 LeetCode 在准备 Testcase 时首先会实现官方算法程序，然后对其中容易出错的地方针对性的设计 Testcase；而且应该会准备多种不同算法的实现（比如既可以 DFS 也可以 BFS、可以用递归也可以直接循环），因为不同做法的易错点可能完全不一样；当然即使不考虑算法或具体问题，仅针对测试数据的典型或边界情况（最短、最长、空、奇偶等等）设计 Testcase 也是应有之义。总之 LeetCode 出 Bug 即使是小概率事件但也是必然的，实际上没多久我就发现了[第二个](https://github.com/aitchjoe/leetcode-fuzzing/tree/main/valid-number)。

虽然解题很爽，但挑战出题者当然更酷，在有了这些好消息（对我）后都没顾得上刷题就投入到下一节中去了。

### 模糊测试

很明显凭我的算法水平完全不可能快速的找 LeetCode Bug，正好当时在学习 Go 语言并接触到了模糊测试，先介绍一下大致的概念。模糊测试并不是一种新的技术，早已用于安全领域，按 [Wiki](https://zh.m.wikipedia.org/wiki/模糊测试) 的定义：

> 模糊测试 （fuzz testing, fuzzing）是一种软件测试技术。其核心思想是將自动或半自动生成的随机数据输入到一个程序中，并监视程序异常，如崩溃，断言（assertion）失败，以发现可能的程序错误，比如内存泄漏。模糊测试常常用于检测软件或计算机系统的安全漏洞。

可以暂时将其理解为一种暴力测试：尽可能检查全部输入的结果是否正确。显然这种测试也不局限于安全领域，我们可以用其检查任何性质的程序逻辑，但是如以上 Wiki 所述，在安全领域可以用程序异常如崩溃等作为判断标准，对于程序貌似正常、仅仅是返回结果不符合需求的情况，我们又如何简单轻松的"检查全部输入的结果是否正确"？这个是使用模糊测试的一大难点，注意这里的关键是简单轻松，如果针对每个案例去深入了解、针对性设计那又是另一个话题了。

回到找 LeetCode [茬](#bug)这一场景，以上的难点却是迎刃而解，因为我们不需要判断算法的每一个输入结果是否正确，我们只需运行所有已通过的算法、比对同一输入的结果是否一致，如果不一致显然就是 LeetCode 准备的 Testcase 未覆盖到、将该输入添加为新的 Testcase 即可（Bug fixed）。而这种做法最大的好处是完全不用费脑筋分析 LeetCode 每一个问题的具体情况，所有问题的 Fuzz testing 程序都是同一套路、机械复制。实际上模糊测试也是分 Blind fuzzing 和 Guide fuzzing 的，从名称就可以看出 Guide 是需要去设计的，我个人觉得在不少场景（比如现在这个）如果不能无脑 Blind fuzzing，那么模糊测试的价值就不大了:)

有了以上思路之后我创建了 [leetcode-fuzzing](https://github.com/aitchjoe/leetcode-fuzzing) 并陆续测试了二三十个问题，很遗憾只找到两个 Bug，但也不意外目前 2450 道题再排除 Easy level（即使 Testcase 不够也不容易出错），能有十几个 Bug 了不起了？但是要全面测试 LeetCode 也不提供 API 自动获取，目前只能手工查找 Go AC 并拷贝粘贴。。。。

总之借这件事入门了模糊测试，见下一节的扩展讨论。另外提一下模糊测试解决不了 LeetCode testcase 的点火问题，这一阶段仍然是需要人工设计的。

### Go Fuzzing

虽然上一节说了可以暂时将模糊测试看作暴力测试，但实际上模糊测试并不是机械的枚举每一个可能输入，[Go Fuzzing](https://golang.google.cn/security/fuzz/) 提及了：

> For an input to be "interesting", it must expand the code coverage beyond what the existing generated corpus can reach. It's typical for the number of new interesting inputs to grow quickly at the start and eventually slow down, with occasional bursts as new branches are discovered.
>
> You should expect to see the "new interesting" number taper off over time as the inputs in the corpus begin to cover more lines of the code, with occasional bursts if the fuzzing engine finds a new code path.

也就是说模糊测试会评估每一个输入的效果、推测其是否更容易发现问题，因此理论上它会比暴力测试更快找到 Bug。但实际上由于当前 Go 实现上的限制，比如 "The fuzzing arguments can only be the following types: string ..."，而 LeetCode 问题的输入通常不会是泛泛的 string、而是包含有最短最长以及字符集等限制的 string，因此在我的测试程序内部仍然要自己生成输入数据，而这对于 Go fuzzing 就变成了黑盒状态？当然也就不能发现哪个 Input 更 interesting 了。总之我目前是将 Go 的模糊测试工具用成了一个暴力测试，这还需要继续研究，比如 Go 的 mutator 等机制是否可以解决？

另外提一下模糊测试结果和通常的含义不一样，如以下命令：

```
go test -fuzz=Fuzz -fuzztime=55m .
```

只是表示在 55 分钟的模糊测试中没有发现问题、并不表示绝对没问题，如果不加这个参数则会无休止的运行下去。

## 真实世界

如上所述，LeetCode 对于提交失败的程序会提示具体未通过的测试样例和预期值，实际上到这一步相对来讲就容易解决 Bug 了；哪怕在考试时不做具体提示、仅仅告知对错（还有 Testcase 成功的百分比），这对于真实世界，已经是拉开了巨大的鸿沟。

更何况在真实世界，不止是对错，还有 KPI 统计不出的好与差。

甚至做对了却评判错。

所以在刷完题之后，Welcome To The Real World :)
