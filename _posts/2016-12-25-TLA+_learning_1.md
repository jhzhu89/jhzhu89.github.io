---
layout: single
title:  "学习TLA+（一）"
categories: 
  - Written in Chinese
tags:
  - TLA+
  - distributed systems
toc: true
mathjax: true
excerpt_separator: <!--more-->
---

最近在读Raft这篇关于分布式系统一致性算法的论文。作者采用了TLA+语言来描述Raft的规范。像英语、汉语这样的自然语言，很难很精确的描述一个系统得行为。大牛`Leslie Lamport`发明了TLA，通过时序逻辑方程来描述系统。TLA为描述系统提供了数学基础，而TLA+是构建在TLA上的一个完备的描述语言。

<!--more-->

## TLA 和 TLA+ 的定义

以下是摘自[TLA官网](http://research.microsoft.com/en-us/um/people/lamport/tla/tla-intro.html)上的对于TLA和TLA+的定义：

> TLA stands for the Temporal Logic of Actions, but it has become a shorthand for referring to the TLA+ specification language and the PlusCal algorithm language, together with their associated tools.

> TLA+ is based on the idea that the best way to describe things formally is with simple mathematics, and that a specification language should contain as little as possible beyond what is needed to write simple mathematics precisely. TLA+ is especially well suited for writing high-level specifications of concurrent and distributed systems.

TLA+ 很适合用来描述分布式系统的行为。借学习 Raft 论文的机会，我将学习<<Specifying Systems - The TLA+ Language and Tools for Hardware and Software Engineers>>这本书，希望以后有机会将这项技术应用到具体的开发过程中。

## 行为（Behaviors）的定义

该书旨在教大家用规范语言来描述系统的行为，以下是书中对`行为`的定义：

> A computer system differs from the systems traditionally studied by scientists because we can pretend that its state changes in discrete steps. So, we represent the execution of a system as a sequence of states. Formally, we define a `behavior` to be a sequence of states, where a state is an assignment of values to variables.  We specify a system by specifying a set of possible behaviors - the ones representing a correct execution of the system.

大意为：计算机系统和科学家们传统研究的系统的区别在于计算机可以用离散的步骤来模仿状态的改变。所以，我们用一系列状态来表示系统的运行。我们把`behavior`定义为一系列状态，而状态是对一些变量赋的值。我们通过描述一组呈现系统正确运状态行的行为来描述一个系统。

## 描述一个只显示小时的时钟

本书第二章通过一个描述时钟的例子，让我们感受以下描述一个系统的过程。首先是描述它的行为。对于这个时钟，它的一个典型行为是：

$$
[hr = 11] -> [hr = 12] -> [hr = 1] -> [hr = 2] -> ... 
$$

一对连续的状态，比如*[hr = 1] -> [hr = 2]*, 叫做`一步(Step)`。

要描述这个时钟，我们需要描述它所有可能的行为。我们首先要描述它的所有可能的初始值，和一个描述这个时钟值如何在后续步骤中改变的关系。我们可以非正式的将初始值和后继状态关系定义为:

$$
HC_{ini} \triangleq hr \subset \{1, ..., 12\} \\
HC_{nxt} \triangleq hr^{'} = IF\ hr \neq 12\ THEN\ hr + 1\ ELSE\ 1
$$

这种包含有`撇号'`修饰的变量的方程，叫做一个`动作(action)`。一个满足$$ HC_{nxt} $$的step，叫做$$HC_{nxt} $$ step。

怎么用一个方程来描述这个系统？这个方程要对一个`behavior`做出两个断言：

1. 它的初始状态满足$$ HC_{ini} $$
2. 它的每一步满足$$ HC_{nxt} $$

文中使用了时序逻辑运算符$$ \Box $$来表达第二条断言。$$ \Box HC_{nxt} $$ 断言对于这个行为的每一步，$$ HC_{nxt} $$都成立。$$ HC_{ini} \land \Box HC_{nxt} $$满足了我们的要求。

如果我们只考虑这个时钟系统，并且不把它关联到其他系统中，那么用这个方程描述它已经足够了。但是，如果它是某一个大系统的一部分，则还有一些可能的状态需要考虑。比如一个天气系统，需要显示小时和温度。它的一个可能的行为如下：

$$
\begin{bmatrix} hr = 11 \\ tmp = 23.5 \end{bmatrix} \rightarrow  \begin{bmatrix} hr = 11 \\ tmp = 23.5 \end{bmatrix} \rightarrow \begin{bmatrix} hr = 12 \\ tmp = 23.4 \end{bmatrix} \rightarrow \begin{bmatrix} hr = 12 \\ tmp = 23.3 \end{bmatrix} \rightarrow \begin{bmatrix} hr = 1 \\ tmp = 23.3 \end{bmatrix} \rightarrow \ ...
$$

其中有些步，温度变了而时间没有改变。这样的步，并不满足$$ \Box HC_{nxt} $$，因为这个方程要求在每一步中，时间必须递增，$$ HC_{ini} \land \Box HC_{nxt} $$并不能描述这个天气系统中的时钟。

一个能够描述任意时钟的方程，必须允许在某些步中，hr保持不变，即：$$ hr = hr^{'} $$，它们叫做`假步(stuttering steps)`。所以，一个能够描述任意时钟的方程为：$$ HC_{ini} \land \Box ( HC_{nxt} \lor (hr = hr^{'})) $$。在TLA中，$$ [HC_{nxt}]_{hr} $$表示$$ HC_{nxt} \lor (hr = hr^{'}) $$。

结合以上，我们用方程HC来描述时钟：$$ HC \triangleq HC_{ini} \land \Box [HC_{nxt}]_{hr} $$。

如果一个时钟行为正常，那么它应该正确的显示 1 到 12，就是说，对于任何满足这个时钟描述 $$HC$$ 的行为，对于它们的每一个状态，hr 应该是一个介于 1 到 12 的整数。$$ HC_{ini} $$ 断言 hr 是一个介于 1 到 12 的整数，而 $$ \Box HC_{ini} $$ 则断言 $$ HC_{ini} $$ 永远成立。所以，对于任何满足 $$HC$$ 的行为，$$ \Box HC_{ini} $$ 都应该成立。用数学语言讲，就是 $$HC$$ 暗含 $$ \Box HC_{ini} $$，即：$$ HC \Rightarrow \Box HC_{ini} $$。

一个每一种行为都满足的时序方程叫做`定理(theorem)`，所以 $$ HC \Rightarrow \Box HC_{ini} $$ 是一个定理。

## 用 TLA＋ 描述这个时钟

TLA+ 和数学方程基本一致。

~~~
------------------------------- MODULE clock -------------------------------
EXTENDS Naturals
VARIABLE hr
HCini == hr \in (1 .. 12)
HCnxt == hr' = IF hr # 12 THEN hr + 1 ELSE 1
HC == HCini /\ [][HCnxt]_hr
----------------------------------------------------------------------------
THEOREM HC => []HCini
============================================================================
~~~

我们可以进一步用取模运算将 $$ HC_{nxt} $$ 化简为：

~~~
HCnxt2 == hr' = (hr%12) + 1
HC == HCini /\ [][HCnxt2]_hr
~~~

## 后记

到此为止，学习了这本书前两章内容，对 TLA+ 有了一些认识和感觉。通过 TLA+ 这种规范语言来描述系统，如同写出数学命题，然后证明其成立，十分严谨，且让人信服。对于我们工程师而言，证明其成立也许不是我们工作的重点，然而通过 TLA+ 来描述系统的时序逻辑，在用其仿真工具来模拟执行流程，可以很好的帮我们理解系统的行为和发现 bug。

