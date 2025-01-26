---
title: The Arzelà–Ascoli theorem from the duality perspective
date: 2025-01-22
permalink: /posts/2025/01/arzela-ascoli/
tags:
  - general topology
  - analysis
---

## Introduction

The [Arzelà–Ascoli theorem](https://en.wikipedia.org/wiki/Arzel%C3%A0%E2%80%93Ascoli_theorem) is an indispensable tool in mathematical analysis which characterizes compact sets of continuous functions in terms of *equicontinuity*. A "hard analysis way" of understanding this theorem is that, the only ways to "escape to infinity" for a collection of continuous functions are either:

1. values of the functions at a point blow up, or
2. regularities of the functions blow up.

In other words, if values at any given point are confined in a compact set and also the functions are "uniformly regular" (i.e., *equicontinuous*), then the collection must be relatively compact.

On the other hand, we can also view this statement in a more "soft analysis way": it is a natural consequence of how compactness interplays with joint continuity of the pairing of functions and points. As an illustration of this viewpoint, I provided below a proof of the Arzelà–Ascoli theorem based on such an idea.

## Setup

Let me describe the precise setup first. Let $X,Z$ be any sets and $\left\langle\,\cdot\,,\,\cdot\,\right\rangle\colon Z\times X \to Y$ be any function into a bounded metric space $(Y,d)$. Here are some remarks:

- The set $X$ is meant to be the "space", i.e., the set of points.
- The set $Z$ is meant to be the set of $Y$-valued functions on $X$.
- The pairing $\left\langle z,x\right\rangle$ is meant to be the evaluation of $z$ at $x$.
- However, $X$ being the space and $Z$ being the functions is just the matter of interpretation, and we can freely reverse their roles at any time if we want. In fact, when we write $f(x)$, we can either regard $x$ as an object and $f$ as an operator acting on $x$, or $f$ as an object and $x$ as an operator acting on $f$. There really is no fundamental difference between the two, which can be considered the essence of [many](https://en.wikipedia.org/wiki/Dual_space) [of the](https://en.wikipedia.org/wiki/Gelfand_representation) [so-called](https://en.wikipedia.org/wiki/Pontryagin_duality) ["dualities"](https://en.wikipedia.org/wiki/Stone%27s_representation_theorem_for_Boolean_algebras) in mathematics. This is one of the key ingredients of the proof.
- $Y$ can (should) be in fact any [uniform space](https://en.wikipedia.org/wiki/Uniform_space). We chose it to be a bounded metric space solely because of possible readers' convenience, and all proofs given below can be generalized to uniform spaces without any nontrivial issue. (In particular, the boundedness condition on $d$ is not really a restriction as we can always replace any metric by its truncation without changing the induced uniformity and topology.)

Given a set $A$, the set $Y^{A}$ of *all* functions $A\to Y$ can be either given with the *pointwise convergence topology* which is nothing but the product topology, or the *uniform convergence topology* which is the topology induced by the metric

$$
    d_{\infty}\colon (f,g) \mapsto \sup_{a\in A}d(f(a),g(a)).
$$

Let us denote the pointwise convergence topology on $Y^{A}$ as $\mathscr{P}(A)$ and the uniform convergence topology on $Y^{A}$ as $\mathscr{U}(A)$.

When $A$ is a topological space, there is another topology we often consider, the *compact convergence topology* (a.k.a. the *topology of uniform convergence on compacta*), which is the [coarsest (i.e., smallest) topology](https://en.wikipedia.org/wiki/Initial_topology) on $Y^{A}$ which makes all the restriction maps

$$
  Y^{A} \to (Y^{K},\mathscr{U}(K))
$$

continuous where $K$ runs over all compact subsets of $A$. In other words, this is the topology induced by the family $$\left\{(f,g)\mapsto\sup_{a\in K}d(f(a),g(a))\right\}_{K}$$ of pseudometrics on $Y^{A}$. We denote this topology as $\mathscr{K}(A)$.

The theorem we want to prove is the following:

><b id="arzela-ascoli">Theorem 1</b> **(Arzelà–Ascoli theorem).**
>
>Suppose $X$ is a topological space and $\mathscr{F}\subseteq Y^{X}$ a collection of functions $X\to Y$ whose restrictions onto each compact subset $K\subseteq X$ are continuous. Then $\mathscr{F}$ is contained in a $\mathscr{K}(X)$-compact subset of $Y^{X}$ if and only if the following conditions hold:
>1. for each $x\in X$, the set $$\left\{f(x)\in Y\colon f\in\mathscr{F}\right\}$$ is contained in a compact subset of $Y$, and
>2. for each compact subset $K\subseteq X$, $$\left\{f\vert_{K}\in Y^{K}\colon f\in\mathscr{F}\right\}$$ is an equicontinuous collection of functions $K\to Y$.
>

(The reason why we say "contained in a compact subset" rather than "relatively compact" is because, in general, the latter is strictly stronger than the former when the ambient space is not [preregular](https://topospaces.subwiki.org/wiki/Preregular_space).)

Here, *equicontinuous* collection of functions means the following:

>**Definition 2 (Equicontinuity).**
>
>Suppose $X$ is a topological space. Then a collection $\mathscr{F}\subseteq Y^{X}$ of functions $X\to Y$ is said to be **equicontinuous** at $x\in X$ if for each $\epsilon>0$, there exists a neighborhood $U$ of $x$ such that $d(f(x'),f(x))\leq\epsilon$ holds for all $x'\in U$ and all $f\in\mathscr{F}$. Also, we call the collection $\mathscr{F}$ **equicontinuous** if it is equicontinuous at every point in $X$.
>

Applying the space-function duality, we can view $\mathscr{F}$ as a space where each $x\in X$ acts as a function, which allows us to interpret equicontinuity as follows: $\mathscr{F}$ is equicontinuous at $x$ if and only if the *evaluation map*

$$\begin{aligned}
  \operatorname{eval}\colon X &\to (Y^{\mathscr{F}},\mathscr{U}(\mathscr{F})) \\
  x' &\mapsto (f\mapsto f(x'))
\end{aligned}$$

is continuous at $x$. Note that this is really just a paraphrase of the definition, and there is nothing deep here. In any case, inspired from this interpretation, we define the following:

>**Definition 3 (Equicontinuity with respect to the pairing).**
>
>Suppose $X$ is a topological space. Then the set $Z$ is said to be **equicontinuous (with respect to the pairing $\left\langle\,\cdot\,,\,\cdot\,\right\rangle$)** at $x\in X$ if the map
>
>$$\begin{aligned}
>  X &\to (Y^{Z},\mathscr{U}(Z)) \\
>  x' &\mapsto (z\mapsto \left\langle z,x'\right\rangle)
>\end{aligned}$$
>
>is continuous at $x$, and $Z$ is said to be **equicontinuous (with respect to the pairing $\left\langle\,\cdot\,,\,\cdot\,\right\rangle$)** if it is equicontinuous at every $x\in X$. Similarly, suppose $Z$ is a topological space, then the set $X$ is said to be **equicontinuous (with respect to the pairing $\left\langle\,\cdot\,,\,\cdot\,\right\rangle$)** at $z\in Z$ if the map
>
>$$\begin{aligned}
>  Z &\to (Y^{X},\mathscr{U}(X)) \\
>  z' &\mapsto (x\mapsto \left\langle z',x\right\rangle)
>\end{aligned}$$
>
>is continuous at $z$, and $X$ is said to be **equicontinuous (with respect to the pairing $\left\langle\,\cdot\,,\,\cdot\,\right\rangle$)** if it is equicontinuous at every $z\in Z$.
>

With these terminologies, [**Theorem 1**](#arzela-ascoli) follows quite automatically by combining three simple lemmas. We state and prove them one by one.

## Lemmas and proofs

>**Lemma 4.**
>
>Suppose $X$ is a topological space and $A\subseteq X$ a dense subspace. Then for any subset $\mathscr{F}\subseteq Y^{X}$ consisting of continuous functions, the restriction map $(\mathscr{F},\mathscr{U}(X))\to (Y^{A},\mathscr{U}(A))$ is a homeomorphism onto its image.
>

>**Proof.** The map is clearly continuous, and injectivity follows immediately by continuity of each element in $\mathscr{F}$. Hence, it is enough to show that a [net](https://en.wikipedia.org/wiki/Net_(mathematics)) $$\left(f_{\alpha}\right)_{\alpha\in D}$$ in $\mathscr{F}$ converges uniformly to a function $f\in\mathscr{F}$ whenever the corresponding net $$\left(f_{\alpha}\vert_{A}\right)_{\alpha\in D}$$ in $Y^{A}$ converges uniformly to $$f\vert_{A}$$. Indeed, for such a case, for any given $\epsilon>0$, $x\in X$ and $\alpha\in D$, continuity of $f$ and $f_{\alpha}$ ensures the existence of $a\in A$ such that
>
>$$\begin{aligned}
>  d(f_{\alpha}(x),f(x)) &\leq d(f_{\alpha}(x),f_{\alpha}(a))
>  + d(f_{\alpha}(a),f(a))
>  + d(f(a),f(x)) \\
>  &\leq \epsilon + d_{\infty}(f_{\alpha}\vert_{A}, f\vert_{A}) + \epsilon.
>\end{aligned}$$
>
>Since $\epsilon>0$, $x\in X$ and $\alpha\in D$ are all arbitrary, the uniform convergence $$f_{\alpha}\to f$$ follows.$\quad\blacksquare$

An immediate consequence of this lemma is that, when $X,Z$ are topological spaces and each $\left\langle \,\cdot\,,x\right\rangle\colon Z\to Y$ is continuous, then a subset $S\subseteq Z$ is equicontinuous if and only if $\overline{S}\subseteq Z$ is equicontinuous. Note that continuity of every $\left\langle \,\cdot\,,x\right\rangle\colon Z\to Y$ means nothing but that the topology on $Z$ is finer than (i.e. contains) the pointwise convergence topology inherited from $Y^{X}$, which shows:

><b id="stability-of-equicontinuity-under-closure"></b>**Corollary 5.**
>Suppose $X,Z$ are topological spaces and the map $Z\mapsto (Y^{X},\mathscr{P}(X))$ is continuous. Then a subset $S\subseteq Z$ is equicontinuous if and only if $\overline{S}\subseteq Z$ is equicontinuous.
>

The other two lemmas relate joint continuity of the pairing $\left\langle\,\cdot\,,\,\cdot\,\right\rangle$ to how the topology on $Z$ compares with topologies we can give on $Y^{X}$.

><b id="local-uniform-to-joint-continuity"></b>**Lemma 6.**
>Suppose $X,Z$ are topological spaces such that $\left\langle z,\,\cdot\,\right\rangle\colon X\to Y$ is continuous for each $z\in Z$. If for each $x\in X$, there exists a neighborhood $U\subseteq X$ of $x$ such that the map $Z\to (Y^{U},\mathscr{U}(U))$ is continuous, then the pairing $\left\langle\,\cdot\,,\,\cdot\,\right\rangle\colon Z\times X\to Y$ is jointly continuous.
>

That is, if $Z$ is consisting of continuous functions $X\to Y$ and if the topology on $Z$ is finer than the *local uniform convergence topology* inherited from $Y^{X}$, then the pairing must be jointly continuous. Note that this local uniform convergence topology coincides with the usual uniform convergence topology $\mathscr{U}(X)$ when $X$ is compact.

>**Proof.** Take any net $$\left((z_{\alpha},x_{\alpha})\right)_{\alpha\in D}$$ in $Z\times X$ convergent to some $(z,x)\in Z\times X$. Then we can find a neighborhood $U$ of $x$ such that the map $Z\to (Y^{U},\mathscr{U}(U))$ is continuous, so in particular $$\left(\left\langle z_{\alpha},\,\cdot\,\right\rangle\right)_{\alpha\in D}$$ converges to $\left\langle z,\,\cdot\,\right\rangle$ uniformly on $U$. Without loss of generality, we can assume $x_{\alpha}\in U$ for all $\alpha\in D$. Then since $\left\langle z,\,\cdot\,\right\rangle\colon X\to Y$ is continuous, we obtain
>
>$$\begin{aligned}
>  d(\left\langle z_{\alpha},x_{\alpha}\right\rangle,
>  \left\langle z,x\right\rangle)
>  \leq d(\left\langle z_{\alpha},x_{\alpha}\right\rangle,
>  \left\langle z,x_{\alpha}\right\rangle)
>  + d(\left\langle z,x_{\alpha}\right\rangle,
>  \left\langle z,x\right\rangle) \to 0
>\end{aligned}$$
>
>as $\alpha\to\infty$, which establishes the joint continuity of $\left\langle \,\cdot\,,\,\cdot\,\right\rangle$.$\quad\blacksquare$
>

The last lemma is an implication in the other direction:

><b id="joint-continuity-to-compact-convergence"></b>**Lemma 7.**
>Suppose $X,Z$ are topological spaces. If the pairing $\left\langle\,\cdot\,,\,\cdot\,\right\rangle\colon Z\times X\to Y$ is jointly continuous, then the map $Z\to(Y^{X},\mathscr{K}(X))$ is continuous.
>

That is, if the pairing is jointly continuous, then the topology on $Z$ must be finer than the *compact convergence topology* inherited from $Y^{X}$. This topology is in general coarser than the *local uniform convergence topology*, but of course they coincide if $X$ is locally compact. Hence, for that case, the continuity of the map $Z\to(Y^{X},\mathscr{K}(X))$ together with continuity of each $\left\langle z,\,\cdot\,\right\rangle$ is equivalent to the joint continuity of the pairing.

>**Proof.** Take any net $$\left(z_{\alpha}\right)_{\alpha\in D}$$ in $Z$ convergent to some $z\in Z$ and fix a compact subset $K\subseteq X$. Suppose for sake of contradiction that the net $$\left(\left\langle z_{\alpha},\,\cdot\,\right\rangle\right)_{\alpha\in D}$$ does not converge uniformly on $K$ to $\left\langle z,\,\cdot\,\right\rangle$. This means that, there exists $\epsilon>0$ such that for any $\alpha\in D$, there exists $h(\alpha)\geq \alpha$ and $$x_{\alpha}\in K$$ satisfying
>
>$$
>  d\left(\left\langle z_{h(\alpha)},x_{\alpha}\right\rangle,
>  \left\langle z,x_{\alpha}\right\rangle\right) \geq \epsilon.
>$$
>
>Since $K$ is compact, passing to a [subnet](https://en.wikipedia.org/wiki/Subnet_(mathematics)) if necessary, we can assume that $$\left(x_{\alpha}\right)_{\alpha\in D}$$ converges to some $x\in K$. However, since the nets $$\left((z_{h(\alpha)},x_{\alpha})\right)_{\alpha\in D}$$ and $$\left((z,x_{\alpha})\right)_{\alpha\in D}$$ in $Z\times X$ both converge to $(z,x)$ and the pairing $\left\langle\,\cdot\,,\,\cdot\,\right\rangle$ is jointly continuous, we have reached a contradiction, establishing the continuity of the map $Z\mapsto (Y^{X},\mathscr{K}(X))$.$\quad\blacksquare$
>

As noted earlier, we obtain the following corollary:

><b id="joint-continuity-characterization">Corollary 8</b>**.**
>Suppose $X,Z$ are topological spaces and $X$ is locally compact. Then the pairing $\left\langle\,\cdot\,,\,\cdot\,\right\rangle\colon Z\times X\to Y$ is jointly continuous if and only if the following conditions hold:
>1. the map $Z\to (Y^{X},\mathscr{K}(X))$ is continuous, and
>2. $\left\langle z,\,\cdot\,\right\rangle\colon X\to Y$ is continuous for each $z\in Z$.
>

Another corollary, which is a very useful statement in its own, is that, for an equicontinuous family of functions, the pointwise convergence topology and the compact convergence topology coincide:

><b id="coincidence-of-pointwise-and-compact-convergence">Corollary 9</b>**.**
>Suppose $X$ is a topological space and $Z$ is equicontinuous. Then the pointwise convergence topology (i.e., the topology inherited from $(Y^{X},\mathscr{P}(X))$) and the compact convergence topology (i.e., the topology inherited from $(Y^{X},\mathscr{K}(X))$) coincide on $Z$.
>

(By "inherited from" we mean the [initial topology](https://en.wikipedia.org/wiki/Initial_topology) on $Z$ induced by the map $Z\to Y^{X}$.)

>**Proof.** Let us denote the topologies on $Z$ inherited from $(Y^{X},\mathscr{P}(X))$ and $(Y^{X},\mathscr{K}(X))$ as $\mathscr{P}(X)$ and $\mathscr{K}(X)$, respectively. Clearly, $\mathscr{K}(X)$ is finer than (i.e. contains) $\mathscr{P}(X)$, so we only need to show that $(Z,\mathscr{P}(X)) \to (Z,\mathscr{K}(X))$ is continuous. Recall that $Z$ being equicontinuous means that the map $X\to (Y^{Z},\mathscr{U}(Z))$ is continuous. Then since $\left\langle\,\cdot\,,x\right\rangle\colon Z\to Y$ is continuous with respect to $\mathscr{P}(X)$ for each $x\in X$, applying [**Lemma 6**](#local-uniform-to-joint-continuity) with $X\leftarrow (Z,\mathscr{P}(X))$ and $Z\leftarrow X$ shows that the pairing $\left\langle\,\cdot\,,\,\cdot\,\right\rangle$ is jointly continuous on $(Z,\mathscr{P}(X))\times X$, so that now applying [**Lemma 7**](#joint-continuity-to-compact-convergence) with $X\leftarrow X$ and $Z\leftarrow (Z,\mathscr{P}(X))$ shows that the map $(Z,\mathscr{P}(X))\to (Y^{X},\mathscr{K}(X))$ is continuous. Hence, continuity of $(Z,\mathscr{P}(X))\to (Z,\mathscr{K}(X))$ follows.$\quad\blacksquare$
>

Now, we prove our main theorem.

>**Proof of** [**Theorem 1**](#arzela-ascoli)**.** $(\Rightarrow)$ Suppose that $\mathscr{F}$ is contained in a $\mathscr{K}(X)$-compact subset $\mathscr{B}\subseteq Y^{X}$. Define $\mathscr{L} \mathrel{\unicode{x2254}} \mathscr{B}\cap\overline{\mathscr{F}}$ where the closure is taken in $Y^{X}$ with respect to $\mathscr{K}(X)$, then $\mathscr{L}$ is $\mathscr{K}(X)$-compact and contains $\mathscr{F}$. Also, since uniform limits of continuous functions are continuous, we know that each $f\in \mathscr{L}$ is continuous when restricted to each compact subset $K\subseteq X$. Therefore, fix such $K$ and endow $\mathscr{L}$ with the uniform convergence topology on $K$, then we can apply [**Corollary 8**](#joint-continuity-characterization) with $X\leftarrow K$ and $Z\leftarrow \mathscr{L}$ to conclude that the pairing $(f,x)\mapsto f(x)$ is jointly continuous on $\mathscr{L}\times K$. Then, since the evaluation map $f\mapsto f(x)$ is clearly continuous on $\mathscr{L}$ for each $x\in K$, we can again apply [**Corollary 8**](#joint-continuity-characterization) with $X\leftarrow \mathscr{L}$ and $Z\leftarrow K$ to conclude that the map $K\to(Y^{\mathscr{L}},\mathscr{U}(K))$ is continuous, which means nothing but that the collection $$\left\{f\vert_{K}\in Y^{K}\colon f\in \mathscr{L}\right\}$$ of functions $K\to Y$ is equicontinuous. Clearly, $$\left\{f\vert_{K}\in Y^{K}\colon f\in \mathscr{F}\right\}$$ is equicontinuous as well.
>
>On the other hand, the continuity of the evaluation map $f\mapsto f(x)$ on $(\mathscr{L},\mathscr{K}(X))$ for given $x\in X$ shows that the image $$\left\{f(x)\in Y\colon f\in \mathscr{L}\right\}$$ of $\mathscr{L}$ under this map is compact. Therefore, $$\left\{f(x)\in Y\colon f\in \mathscr{F}\right\}$$ is contained in a compact subset of $Y$, as claimed.
>
>$(\Leftarrow)$ For each $x\in X$, let $B_{x}$ be a compact subset of $Y$ containing $$\left\{f(x)\in Y\colon f\in\mathscr{F}\right\}$$. Define $\mathscr{B} \mathrel{\unicode{x2254}} \prod_{x\in X}B_{x}\subseteq Y^{X}$, then $\mathscr{B}$ is a $\mathscr{P}(X)$-compact subset of $Y^{X}$ by the [Tychonoff's theorem](https://en.wikipedia.org/wiki/Tychonoff%27s_theorem) and it contains $\mathscr{F}$. Let $\mathscr{L} \mathrel{\unicode{x2254}} \mathscr{B}\cap \overline{\mathscr{F}}$ where the closure is taken in $Y^{X}$ with respect to $\mathscr{P}(X)$, then $\mathscr{L}$ is $\mathscr{P}(X)$-compact and contains $\mathscr{F}$.
>
>Fix a compact subset $K\subseteq X$. Then the equicontinuity of $$\left\{f\vert_{K}\in Y^{K}\colon f\in \mathscr{F}\right\}$$ implies the equicontinuity of $$\left\{f\vert_{K}\in Y^{K}\colon f\in \mathscr{L}\right\}$$. Indeed, $\mathscr{L}$ is contained in the $\mathscr{P}(X)$-closure of $\mathscr{F}$, so we can apply [**Corollary 5**](#stability-of-equicontinuity-under-closure) with $Z\leftarrow (\mathscr{L},\mathscr{P}(X))$ and $S\leftarrow \mathscr{F}$. Therefore, [**Corollary 9**](#coincidence-of-pointwise-and-compact-convergence) shows that $\mathscr{P}(K)$ and $\mathscr{U}(K)$ coincide on $\mathscr{L}$, but since $K$ is arbitrary, we conclude that $\mathscr{P}(X)$ and $\mathscr{K}(X)$ coincide on $\mathscr{L}$, which in turn shows that $\mathscr{L}$ is $\mathscr{K}(X)$-compact.$\quad\blacksquare$
>

As you can see, proofs of both directions leverage the aforementioned duality of $X$ and $Z$, or more precisely, that they can switch their roles, using joint continuity of the pairing $(f,x)\mapsto f(x)$ as a pivot of the switch.

A final remark is that, in fact, the same proof still applies to a slightly more general setup which I find occasionally useful. The observation is that, even though we are heavily relying on compactness of the domain, we do not necessarily consider *all* compact subsets, maybe just *some* compact subsets. This means that we can in fact replace the topology $\mathscr{K}(X)$ by a coarser topology obtained by a smaller collection of compact sets. In such a case, we also have to modify the equicontinuity condition accordingly; that is, the functions in the collection needs to be equicontinuous only when restricted to those compact sets in consideration, but not necessarily when restricted to any compact sets. Such a generalization yields, as a simple corollary, a slight variation of [**Theorem 1**](#arzela-ascoli) in terms of uniformly continuous functions and totally bounded sets instead of continuous functions and compact sets. We omit details in this post.