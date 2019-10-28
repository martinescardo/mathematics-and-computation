---
title: Runners in action
author: Andrej Bauer
layout: post
categories:
  - Programming languages
  - Software
  - Publications
---

It has been almost a decade since [Matija Pretnar](http://matija.pretnar.info)
and I posted the [first blog posts](http://math.andrej.com/category/eff/) about
programming with algebraic effects and handlers and the programming language
[Eff](http://www.eff-lang.org). Since then handlers have become a well-known
control mechanism in programming languages.

Handlers and monads excel at *simulating* effects, either in terms of other
effects or as pure computations. For example, the familiar [state
monad](https://wiki.haskell.org/State_Monad) implements mutable state with
(pure) state-passing functions, and there are many more examples. But I have
always felt that handlers and monads are not very good at explaining how a
program interacts with its external environment and how it gets to perform
*real-world* effects.

[Danel Ahman](https://danel.ahman.ee) and I have worked for a while on attacking
the question on how to better model external resources and what programming
constructs are appropriate for working with them. The time is right for us to
show what we have done so far. The theoretical side of things is explained in
our paper [**Runners in action**](http://arxiv.org/abs/1910.11629), Danel
implemented a Haskell library
[**Haskell-Coop**](https://github.com/danelahman/haskell-coop) to go with the
paper, and I implemented a programming language
[**Coop**](https://github.com/andrejbauer/coop).

<!--more-->

General-purpose programming languages, even the so-called pure ones, have to have
*some* account of interaction with the external environment. A popular choice is
to provide a foreign-function interface that connects the language with an
external library, and through it with an operating system and the universe. A
more nuanced approach would differentiate between a function that just happens
to be written in a different language, and one that actually performs an effect.
The latter kind is known as an *algebraic operation* in the
algebraic-effects-and-handlers way of doing things.

A *bad* approach to modeling the external world is to pretend that it is
internal to the language. One would think that this is obvious but it is not.
For instance, Haskell represents the interface to the external world through the
[`IO`
monad](https://www.haskell.org/onlinereport/haskell2010/haskellch41.html#x49-32100041.1).
But what is this monad *really*? How does it get to interact with the external
world? The Haskell Wiki page which answers this question has [the following
disclaimer](https://wiki.haskell.org/IO_inside#Welcome_to_the_RealWorld.2C_baby):

> *"Warning: The following story about `IO` is incorrect in that it cannot
> actually explain some important aspects of `IO` (including interaction and
> concurrency). However, some people find it useful to begin developing an
> understanding."*

The Wiki goes on to say how `IO` is a bit like a state monad with an imaginary
`RealWorld` state, except that of course `RealWorld` is not really a Haskell
type, or at least not one that actually holds the state of the real world.

The situation with Eff is not much better: it treats some operations at the
top-level in a special way. For example, if `print` percolates to the top level,
it turns into a *real* `print` that actually causes an effect. So it looks like
there is some sort of "top level handler" that models the real world, but that
cannot be the case: a handler may discard the continuation or run it twice, but
Eff hardly has the ability to discard the external world, or to make it
bifurcate into two separate realities.

If `IO` monad is not really monad and a top-level handler is not really a
handler, then what we have is a case of ingenious hackery in need of proper PL
design.

How precisely does an operation call in the program cause an effect in the
external world? As we have just seen, some sort of runtime environment or top
level needs to do relate it to the external world. From the viewpoint of the
program, the external world appears as state which is not directly accessible,
or even representable in the language. The effect of calling an operation
$\mathtt{op}(a,\kappa)$ is to change the state of the world, and to get back a
result. We can model the situation with a map $\overline{\mathtt{op}} : A \times
W \to B \times W$, where $W$ is the set of all states of the world, $A$ is the
set of parameters, and $B$ the set of results of the operation. The operation
call $\mathtt{op}(a, \kappa)$ is "executed" in the current world $w \in W$ by
computing $\overline{\mathtt{op}}(a,w) = (b, w')$ to get the next world $w'$ and
a result $b$. The program then proceeds with the continuation $\kappa\,b$ in the
world $w'$.

What I have just described is *not* a monad or a handler, but a *comodel*, also
known as a *runner*, and the map $\overline{\mathtt{op}}$ is not an operation,
but a *co-operation*. This was all observed a while ago by Gordon Plotkin and
John Power, and by [Tarmo Uustalu](https://www.ioc.ee/~tarmo/), see our paper
for references. A short introductory account of comodels is given in Section 4
of my notes [What is algebraic about algebraic effects and
handlers?](https://arxiv.org/abs/1807.05923). We should be using runners, not
"top-level" effects and "special" monads.

Danel and I worked out how *effectful* runners (a generalization of runners that
supports other effects, apart from state) provide a mathematical model of
resource management. They also give rise to a programming concept that models
top-level external resources, as well as allows programmers to modularly define
their own “virtual machines” and run code inside them. Such virtual machines can
be nested and combined in interesting ways. We capture the core ideas of
programming with runners in an equational calculus $\lambda_{\mathsf{coop}}$,
that guarantees the linear use of resources and execution of finalization code.

If you are familiar with handlers, as a first approximation you can think of
runners as handlers that use the continuation at most once in a tail-call
position. As it turns out, while runners are less general than handlers (and
monads), they have very good properties: they can be modularly composed in
several ways, they guarantee that finalization code will be executed, and they
give an account of *two* kinds of exceptions: ordinary *exceptions*, which are
special events that disrupt the flow of user code and need to be attended to,
and *signals* which are unrecoverable failures which *kill* user code and cannot
be caught, but can be finalized. Various programming languages call these two
kinds "checked" and "non-checked" exceptions, or "errors" and "exceptions", etc.

To find out more, we kindly invite you to have a look at the paper, and to try
out the implementations.

The prototype programming language [Coop](https://github.com/andrejbauer/coop)
implements and extends $\lambda_{\mathsf{coop}}$. You can start by skimming the [Coop
manual](https://github.com/andrejbauer/coop/blob/master/Manual.md) and the
[examples](https://github.com/andrejbauer/coop/tree/master/examples).

Depending on your preferred setup, you might like better the
[Haskell-Coop](https://github.com/danelahman/haskell-coop) library, as it allows
you to combine runners with everything else that Haskell has to offer.