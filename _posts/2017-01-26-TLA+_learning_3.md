---
layout: single
title:  "学习TLA+（三）"
categories: 
  - Written in Chinese
tags:
  - TLA+
  - distributed systems
toc: true
mathjax: true
excerpt_separator: <!--more-->
---

这本书的第四章，讲述了如何描述一个 FIFO。

<!--more-->

## FIFO

如下图所示，一个 Sender 和 Receiver 通过一个 Buffer 和两个 channel 来连接。这个 Buffer 和两个 channel 组成了 FIFO Buffer 这个系统，Sender 和 Receiver 是这个系统的环境。

<figure>
  <img src="/assets/images/TLA+_2_fifo.png">
<figcaption>A FIFO.</figcaption>
</figure>

这个系统有4个 nonstuttering step: in 和 out channel 的 Send 和 Rcv 操作。

## Inner Specification

这一章同时介绍了在 TLA+ 中如何隐藏变量，以下这个 spec 被放在 InnerFIFO 这个模块中，其中表示 queue 的变量将在一个封装模块中被隐藏掉。

上图中的 buffer 就是一个 queue，用 TLA+ 中的 Sequences 来表示。FIFO 的 spec 中声明了一个常量 Message 来表示所有可能被发送的信息。然后声明了3个变量，分别是 in，out，和 q。in 和 out 表示两个 channel，q 表示这个 buffer。

在这个系统中，我们需要两个 channel (上述的 in，out) 来连接，我们可以通过实例化来复用之前定义的 Channel 模块，分别命名为 InChan 和 OutChan。InChan 定义为：

$$
InChan \triangleq\ INSTANCE\ Channel\ WITH\ Data \leftarrow Message,\ chan \leftarrow in
$$

实例化就相当于用 Message 替换 Channel 中的 Data，用 in 替换 Channel 中的 chan，就是：

$$
in \in [val:Message, rdy:\{0,1\}, ack:\{0,1\}]
$$

相应的，OutChan 定义为：

$$
OutChan \triangleq\ INSTANCE\ Channel\ WITH\ Data \leftarrow Message,\ chan \leftarrow out
$$

### 定义初始状态 Init

对于变量的初始状态，in 和 out 由 InChan!Init 和 OutChan!Init 定义，q 初始状态是一个空的Sequence。这个系统的初始状态定义为：

$$
\begin{align*}
Init \triangleq &\land InChan!Init \\ &\land OutChan!Init \\ &\land q = \langle \rangle
\end{align*}
$$

### 定义类型不变式 TypeInvariant

对于类型不变式，in 和 out 可以从 InChan 和 OutChan 中继承过来。这个系统的类型不变式定义为：

$$
\begin{align*}
TypeInvariant \triangleq &\land InChan!TypeInvariant \\ &\land OutChan!TypeInvariant \\ &\land q \in Seq(Message)
\end{align*}
$$

### 定义后继状态 Next

四种 non-stutterring 步由 4 个动作来描述，分别是：

- SSend(msg) The sender sends message `msg` on the `in` channel.
- BufRcv     The buffer receives the message from the `in` channel and appends it to the tail of `q`.
- BufSend    The buffer removes the message from the head of `q` and sends it on channel `out`.
- RRcv       The receiver receives the message from the `out` channel.

它们的定义如下：

$$
\begin{align*}
SSend(msg) \triangleq &\land InChan!Send(msg) \\ &\land UNCHANGED \langle out, q \rangle \\
\\
BufRcv \triangleq &\land InChan!Rcv \\ &\land q^{'} = Append(q, in.val) \\ &\land UNCHANGED \langle out \rangle \\
\\
BufSend \triangleq &\land q \neq \langle \rangle \\ &\land OutChan!Send(Head(q)) \\ &\land q^{'} = Tail(q) \\ &\land UNCHANGED\ in \\
\\
RRcv \triangleq &\land OutChan!Rcv \\ &\land UNCHANGED \langle in, q \rangle
\end{align*}
$$

Next 则定义为：

$$
\begin{align*}
Next \triangleq &\lor \exists msg \in Message\ : SSend(msg) \\ &\lor BufRcv \\ &\lor BufSend \\ &\lor RRcv
\end{align*}
$$

### 定义 Spec

Spec 定义为：

$$
Spec \triangleq Init \land \Box{[Next]_{\langle in,out,q \rangle}}
$$

完整的 Spec 如下图所示。

<figure>
  <img src="/assets/images/TLA+_3_fifo_q_visible.png">
<figcaption>The specification of a FIFO with the internal variable q visible.</figcaption>
</figure>

### 隐藏变量 q

接下来，书中介绍了在 TLA+ 中隐藏变量的方法，仍然以 FIFO 为例，因为 q 属于内部使用的变量，并不需要对外暴露出来。具体的做法是使用 $$ \exists $$ 这个量词来隐藏某个变量。下图是 FIFO 的例子。

<figure>
  <img src="/assets/images/TLA+_3_fifo.png">
</figure>

## Bounded FIFO

在 FIFO 的基础上，这一章又介绍了如何描述一个容量有限制的 FIFO，引入了一个新的常量 N，表示容量上限。FIFO Rcv 动作需要添加新的条件，即 q 当前存储的消息还没有超过容量 N 时，才可以接受新的消息，如下图所示。

<figure>
  <img src="/assets/images/TLA+_3_bounded_fifo.png">
</figure>

## 系统和环境

这章的最后，介绍了封闭系统Spec(closed-system specification)和开放系统Spec(open-system specification)。对于 FIFO 这个系统，Sender 和 Receiver 属于它的环境，Sender 执行发送的动作，Receiver 执行接收的动作。开放系统Spec只描述系统本身的正确行为。

## 后记

通过这一章的学习，熟悉了 TLA+ 中如何描述小的可复用的模块。
