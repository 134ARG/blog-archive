---
title: "有关编译器&lisp的一些胡思乱想"
date: 2021-04-20T11:40:07+08:00
authors: xen134
summary: 胡思乱想!
draft: false
---

## 思想碎片

(2021.4.21 )

Scanner，Parser，Code generator， Optimizer。Lisp的parser理论上直接是一个AST constructor；或者根本就没有scanner和parser之分，直接尝试从源码（source）构建AST，出错则语法报错，成功则一步完成S+P的工作（构建AST时不仅有语法要求，也有语义要求，而lisp的CFG相当简单直接，又几乎没有复杂NCFG）。

对于top-down parser来说，每一个non-terminal都被实现为一个过程，从\<program>或者类似名称的指代完整程序的symbol开始，一路消减到terminal symbol。这个过程的机制类似lisp中常用的利用递归遍历list。

lisp中基本没有（复杂意义上的）上下文相关语法。同样类型的符号在不同上下文中有不同的语义——严格意义上来讲是存在的，但不同的语义均可通过简单地判断s表达式的第一个元素来确定。而s表达式的首元素只有三种：函数（function），特殊操作符（special operator），宏（macro）。

对于AST的基本成员——除根节点之外其他的结点应该都是terminal symbol。lisp的non-terminal symbol大概只有s-expression这一种，剩下的都是atom，即terminal symbol。所以用lisp来实现lisp的parser是一件简单又自然的工作。

确定有限状态机——从一个状态总可以确定地到达下一个状态的机器。从实现的角度，一是可以表示/定义为状态转换表（数据/代码分离），二是可以直接表示为闭包的集合（代码即数据，来自on lisp：*函数作为表达方式*；一种在面向对象中达到相同结果的方法是将“状态”作为对象抽象）。第二种方式即为状态转换图（原文为diagram，但理解为graph也并无不妥）的直接映射。当实现为表时，需要对每一个状态进行索引（下标，哈希）因表格是无状态的；而实现为闭包时可以通过判断输入直接进入下一状态。表格的优点也在于此：因其不保存状态，更新状态机不会成为性能负担；而闭包表示在每次更改状态机时都需要重新构建。Craft a compiler中提到"The table-driven form is commonly produced by a scanner generator; it is token independent." ，但笔者脑测闭包表示也可达到相同效果（通过实现通用constructor；*新项目*）。此书中只提到了table-driven和"handwired"两种方式，一个原因可能是闭包或者说数据结构表示会占用较大的内存，在实践中**可能**并不现实；而表格占用空间相对较小。此点存疑——因为空间占用和状态机的属性（状态数&转换数）有较大关联，需进一步分析编译器应用场景下的需求。另一个可能原因是前面提到过的通用性。

状态机可以表示为网络/图，即构建网络/图的实现方法也可用于构建（有限）状态机。

---

根据正则表达式输入实现确定有限状态机的constructor——正则表达式的解释器。在计算理论或者自动机理论中正则由状态机定义，且正则和状态机可以相互转换。转换关系：epsilonNFA->NFA->DFA->RE->epsilonNFA。其中只有DFA可以实现，但NFA可以更轻松地抽象部分问题。这点的原因是相较于DFA，NFA不局限于“确定”（一个状态可转换为多个不同状态），同时epsilonNFA的epsilon-transition更进一步，不局限于输入（可看做输入为空字符串，自由转换到不同状态），在抽象某些过程时会有比DFA更简单的表示。

NFA到DFA的转换方法是subset construction，即将NFA中某路径下结果状态的并集合并作为DFA相同路径下的单一结果状态。因该并集是NFA状态集合Sigma的一个子集，所以被称为subset construction。显然，理论上NFA到DFA的时间复杂度为O(2^n)，但在实践中（compiler）似乎并没有这么糟糕：一门正常的人造编程语言是不会有那么高的复杂度的，除非它的设计者失了智。（**实现问题**）

epsilonNFA到NFA的转换需要用到一个叫做closure的概念（此闭包非彼闭包）。简单来说epsilonNFA中一个状态和它所有可通过epsilon transition到达的状态组成该状态的闭包（集合）。转换为NFA后，该状态的transition function的结果为对应闭包中所有状态的transition function结果的并集，即DeltaNFA(p, s) == DeltaENFA(Closure(p), s)。（**实现问题**）

下一个问题是为什么需要状态机的概念。作为人造语言，编程语言的状态是有限的，且不会很大（较自然语言）。这里不同状态指的是不同的语法结构（不一定是语义）；同时常态下scanner只处理CFG，NCFG则交由parser通过AST分析。这种场景下scanner的处理过程可自然而然地被抽象为状态机。状态机这种通用且形式化的抽象标准化了处理流程，同时也降低了人脑负担。在此基础上，状态机的等价物正则表达式被用于token的定义。

一个有意思的点是状态机decision property中对infiniteness的验证。设状态机共有n个状态，如果存在长度在[n, 2n-1]范围内的字符串属于该状态机的language集合，则该状态机的language是无限的。首先明确的一点：但凡language中有长度>n的字符串则状态机的language必定无限（参考把鸽子塞进洞理论；这种情况下状态机必有环路）；这里则是确定了一个2n-1的上限使这个算法（看起来）可以被实现。要证明这一点一种方法是采用反正法：

1. 我们假设最小的acceptable string长度为2n，y为路径上的第一个环路，xyz为一个acceptable string（x，y，z均为字符串）。显然，1<=|y|<=n（必定存在环路），|xy|<=n。又|xz|=|xyz|-|y|，我们又假设|acceptable string|>=2n，所以xz的长度至少是n。又y是一个环路，若xyz是acceptable的，那么xz也是acceptable的。同时|xz|< |xyz|，所以对任意acceptable的长度>=2n的字符串，均可对应一个稍小的acceptable的字符串。进而得出任何>=2n的字符串均可被对应到[n, 2n-1]这个范围。这是[Stanford CS154](https://www.bilibili.com/video/BV1LK4y1s7Ch)中提到的简单思路。 

另一种方法则是direct proof：

2. 考虑到无限language与环路是等价的，一个n状态的状态机无重复环路路径的最大长度即acceptable string的最小长度上限。在环路内部（指不考虑转入转出），转换路径数始终与组成环路的状态数相等。所以环路大小本身并不影响总路径长度，起决定作用的是环路外的转换路径数。很明显，环路越大，环路的个数越少，连接环路的外部转换路径越少（=环路数-1）。当所有状态都拥有环路，即环路长度为1时环路数最大，为n个；环路内总转换数为1*n个；环路外连接为n-1个。所以最小acceptable路径的最大长度为n+n-1=2n-1。

第二个有意思的点是构建最小DFA，将indistinguishable的状态合并。[Stanford CS154](https://www.bilibili.com/video/BV1LK4y1s7Ch)中没有显式声明indistinguishable的定义，但不难自行概括：若状态a和b是distinguishable的，那么它们不存在转换到terminal state的路径，或者所有抵达terminal state的路径均相同。但验证indistinguishable在实践中并不现实，更好的方式是判断一个状态对是否distinguishable。对于一个状态对，若存在输入可使其中一个且仅有一个状态转换到terminal state，则这个状态对是distinguishable的。以此为基础，从terminal state出发标记所有distinguishable的状态对，剩余的即为冗余状态，可合并为同一状态。同时注意该属性的传递性以及对简化时产生的unreachable状态的删除。

[Stanford CS154](https://www.bilibili.com/video/BV1LK4y1s7Ch)中化简DFA时使用的是状态机的表格表示。闭包表示下的状态机也可使用同一方式进行化简，但这种算法需要多次对非初末状态进行遍历和更新，在闭包表示上这些操作有些复杂。1.更好的化简 2.更好的构造。

token由RE给出定义，将RE转换为DFA/NFA是实现scanner generator的必要步骤。由于RE的基本操作（catenation，Klenee closure，union）可嵌套，可以用递归很自然地直接处理RE。当然也可以将RE转换为lisp s-expr的形式再处理——不过这种做法本质上是构建RE的AST。鉴于RE比较简单，直接处理也无妨。

每一种token都对应一个DFA。那么scanner如何处理所有的token？最直接的思路是对输入应用所有的DFA，根据某些标准从所有接受该字符串的DFA中选择出最合适的一个DFA。当然这种做法相当无聊，并且本质上是对一个NFA的拙劣实现。好一点的做法是将所有token的DFA构建为一个NFA然后转换为DFA并实现。

