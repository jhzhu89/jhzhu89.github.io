---
layout: single
title:  "学习TLA+（二）"
categories: 
  - Written in Chinese
tags:
  - TLA+
  - distributed systems
toc: true
excerpt_separator: <!--more-->
---

第三章，讲述了如何描述一个异步接口。

<!--more-->

## Asynchoronous Interface

如下图所示，一个 Sender 和 Receiver 通过这个接口来连接。

<figure>
  <img src="/assets/images/TLA+_2_async_interface.png">
<figcaption>An asynchronous interface.</figcaption>
</figure>

以下是这个异步接口系统的行为示例。

$$
\begin{bmatrix} val = 26 \\ rdy = 0 \\ ack = 0 \end{bmatrix} \xrightarrow{Send 37} \begin{bmatrix} val = 37 \\ rdy = 1 \\ ack = 0 \end{bmatrix} \xrightarrow{Ack} \begin{bmatrix} val = 37 \\ rdy = 1 \\ ack = 1 \end{bmatrix} \xrightarrow{Send 4} \begin{bmatrix} val = 4 \\ rdy = 0 \\ ack = 1 \end{bmatrix} \xrightarrow{Ack} \begin{bmatrix} val = 4 \\ rdy = 0 \\ ack = 0 \end{bmatrix} \xrightarrow{Send 19} \begin{bmatrix} val = 19 \\ rdy = 1 \\ ack = 0 \end{bmatrix} \xrightarrow{Ack} ...
$$

## 该异步接口的第一版 specification 

接下来，我们来给用 TLA+ 描述这个系统，给它写 specification (这个 specification 将被放在 AsynchInterface 模块以供复用)。

对于这个系统要传递的数据的具体类型我们并不关心，声明为

$$
CONSTANT\ Data
$$

三个变量则声明为

$$
VARIABLES\ val, rdy, ack
$$

这个系统的类型不变式定义为：

$$
\begin{align*}
TypeInvariant \triangleq &\land val \in Data \\ &\land rdy \in {0,1} \\ &\land ack \in {0,1}
\end{align*}
$$

对于`type`等，书中的定义如下：

- A `state function` is an ordinary expression (one with no prime or $$ \Box $$) that can contain variables and constants.
- A `state predicate` is a Boolean-valued state function.
- An `invariant Inv` of a specification `Spec` is a state predicate such that $$ Spec \implies \Box{Inv} $$ is a theorem.
- A variable `v` has type `T` in a specification `Spec` iff $$ v \in T $$ is an invariant of `Spec`.

弄懂这些定义对理解 TLA+ specification 很重要。

`Type Invariant` 不属于 specification，这个 specification 应该暗含它(implies)。

### 初始状态断言（initial predicate）

初始状态可以定义为：

$$
\begin{align*}
Init \triangleq &\land val \in Data \\ &\land rdy \in {0,1} \\ &\land ack = rdy
\end{align*}
$$

ack 和 rdy 只有两个取值，同样可以使用 ack != rdy 作为初始状态。

### 后继状态动作`Next`（next-state action）

对于这个异步接口协议，它的一个步，是发送或者接收一个值。我们分别定义两个动作`Send`和`Rcv`，则`Next`可以定义为$$ Next \triangleq Send \lor Rcv $$。

Send 和 Rcv 的定义如下：

$$
\begin{align*}
Send \triangleq &\land rdy = ack \\ &\land val^{'} \in {0,1} \\ &\land rdy^{'} = 1 - rdy \\ &\land UNCHANGED\ ack \\
Rcv \triangleq &\land rdy \neq ack \\ &\land ack^{'} = 1 - ack \\ &\land UNCHANGED\ \langle val, rdy  \rangle
\end{align*}
$$

> 这里的`Send`和`Rcv`是用来描述这个异步接口的，属于这个异步接口的specification，但是实际系统中，是 Sender 来触发 Send 这个动作， Receiver 完成 Rcv 这个动作。

### Spec

最后需要定义这个异步接口的Spec，如下：

$$
\begin{align*}
Spec \triangleq Init \land \Box{[Next]_{\langle val, rdy, ack \rangle}}
\end{align*}
$$

### 对于 TLA+ 中 Constant ，Variable 的理解

因为 TLA+ 用时序逻辑方程来描述系统行为，在这个例子中，Data 并不随着时序而变化，它表示一个集合，所以在这里就是个 Constant；而 val， ack ，rdy 都可能随着时序而发生变化，所以才是 Variable 。书中把`Data`称为这个 specification 的一个 parameter ，在后续章节中会看到，这个 Data 可以用另一些类型来实例化。

## 使用 TLA+ Record 重构 specification

接下来，该书介绍了 TLA+ 的 Record，以及一些相关的运算符，使这个 specification 更简洁，更易于被其他系统使用，完整的定义如下：

<figure>
  <img src="/assets/images/TLA+_2_chan.png">
</figure>

图中，chan 是一个 TLA+ 的 Record。

## 后记

我主要想通过理解这些书中的例子，来学习如何对系统的关键部分进行抽象，通过它的一些行为来分解出一些它的一些 action，如何定义系统的初始状态和状态转换方程，最后写出完整的 Spec。更进一步，还要学习如何对 Spec 做出证明(就是对 Spec 暗含 Type invariant 做出证明)。

关于 Record 这部分，主要是 TLA+ 语言的细节，这里就不再赘述了。
