---
title: "A generalization of the Lax-Milgram Theorem"
date: 2024-09-04
permalink: /posts/2024/09/lax-milgram/
tags:
  - analysis
  - functional-analysis
  - pde
---

This is a short note on a generalization of the [Lax-Milgram theorem](https://en.wikipedia.org/wiki/Weak_formulation#The_Lax%E2%80%93Milgram_theorem), which is a common tool for proving existence of weak solutions to PDE.

## Introduction

Let $\Omega\subseteq\mathbb{R}^{n}$ be a bounded and smooth enough domain and $f\in L^{2}(\Omega)$. Consider the PDE

$$\label{eq:Poisson equation}
\begin{aligned}
  -\Delta u &= f & \textrm{on}\quad & \Omega \\
  u &= 0 & \textrm{on}\quad & \partial\Omega.
\end{aligned}
$$

Suppose that $u\in C^{2}(\overline{\Omega})$ is a solution to this equation. Then for each $\phi\in\mathcal{C}_{c}^{\infty}(\Omega)$, we have

$$
  \int_{\Omega}f\phi = -\int_{\Omega} \phi\Delta u
  = -\int_{\Omega}\nabla\cdot(\phi\nabla u)
  + \int_{\Omega}\nabla\phi\cdot\nabla u
  = \int_{\Omega}\nabla\phi\cdot \nabla u,
$$

where the last equality follows from the [divergence theorem](https://en.wikipedia.org/wiki/Divergence_theorem) and that $\phi\vert_{\partial\Omega} = 0$. Since having

$$
  \int_{\Omega}f\phi = -\int_{\Omega}\phi\Delta u
$$

for all $$\phi\in\mathcal{C}_{c}^{\infty}(\Omega)$$ is equivalent to $-\Delta u = f$ (which follows from the [Lebesgue differentiation theorem](https://en.wikipedia.org/wiki/Lebesgue_differentiation_theorem)), we conclude that $u$ is a solution if and only if

$$
  \int_{\Omega}f\phi = \int_{\Omega}\nabla\phi\cdot\nabla u
$$

holds for all $$\phi\in\mathcal{C}_{c}^{\infty}(\Omega)$$. Since both sides define a continuous linear functional on the Hilbert space $$H_{0}^{1}(\Omega)$$ (the norm-closure of $$\mathcal{C}_{c}^{\infty}(\Omega)$$ inside the [Sobolev space](https://en.wikipedia.org/wiki/Sobolev_space) $H^{1}(\Omega)$), we have converted the problem of solving $\eqref{eq:Poisson equation}$ into the problem of finding an element $$u\in H_{0}^{1}(\Omega)$$ such that the continuous linear functional $$\phi\mapsto\int_{\Omega}\nabla\phi\cdot\nabla u$$ coincides with $$\phi\mapsto\int_{\Omega}f\phi$$. Thus, basically what we are asking here is the surjectivity of the linear map

$$
\begin{align*}
  L\colon H_{0}^{1}(\Omega) &\to H_{0}^{1}(\Omega)^{*} \\
  u &\mapsto \left(\phi\mapsto \int_{\Omega}\nabla\phi\cdot\nabla u\right).
\end{align*}
$$

This is called the *weak formulation* of $\eqref{eq:Poisson equation}$. An element $$u\in H_{0}^{1}(\Omega)$$ satisfying $$Lu = \left(\phi\mapsto\int_{\Omega}f\phi\right)$$ is then called a *weak solution* of $\eqref{eq:Poisson equation}$. Once we find a weak solution, then some other machineries can be applied to show uniqueness and regularity of the found weak solution.

Therefore, we can reformulate our PDE into the following functional analysis question: given a bilinear form $B\colon E\times F\to \mathbb{K}$ where $\mathbb{K}=\mathbb{R}$ or $\mathbb{C}$ is the scalar field, when the induced map

$$
\begin{align*}
  L\colon E &\to F^{*} \\
  u &\mapsto \left(v\mapsto B(u,v)\right)
\end{align*}
$$

is surjective?

The [Lax-Milgram theorem](https://en.wikipedia.org/wiki/Weak_formulation#The_Lax%E2%80%93Milgram_theorem) is an answer to this question: $L$ is surjective when $B$ is *coercive*. I will define the term *coercive bilinear form* later, but here let me point out that this concept is somewhat quantitatively defined. I wanted to see what is "the most purely qualitative" characterization of the condition that ensures surjectivity, and got a somewhat simple answer.

## Duality pairings and Mackey-Arens theorem

A lesson I learned from functional analysis is that every attempt for building a duality theory that encompasses, but goes beyond, the usual duality theory of Banach spaces will necessarily suffer, suffer quite a lot. This is not because the concept of duality is broken beyond Banach spaces, rather because it is already broken in the Banach space level. More precisely, the notion of "the correct dual space" is not always the usual one, the space of all continuous linear functionals with the operator norm. Rather, the correct dual space very much depends on the given situation, and there is no one-size-fit-all answer.

This is why it is a good idea to temporarily forget about the concept of continuous dual, and start with a so-called *duality pairing* given on an arbitrary pair $(E,F)$ of spaces, which a priori has nothing to with any kinds of topologies we can give on those spaces.

>**Definition 1 (Duality pairing).**
>
>Let $E,F$ be vector spaces over $\mathbb{K}=\mathbb{R}$ or $\mathbb{C}$. A bilinear map $\left\langle\,\cdot\,,\,\cdot\,\right\rangle\colon F\times E\to \mathbb{K}$ is called a *duality pairing* on $(E,F)$ if:
>1. for any $v\in E$, $\left\langle v,u\right\rangle = 0$ for all $u\in F$ implies $v=0$, and
>2. for any $u\in F$, $\left\langle v,u\right\rangle = 0$ for all $v\in E$ implies $u=0$.
>

For a vector space $E$, let $E^{\star}$ denotes the *algebraic dual* of $E$, that is, the vector space of all linear functionals on $E$ with no further constraints. Then the conditions given in the definition above simply means that the induced maps

$$
\begin{align*}
  \iota\colon F &\to E^{\star} \\
  v &\mapsto \left(u\mapsto\left\langle v,u\right\rangle\right)
\end{align*}
$$

and

$$
\begin{align*}
  \iota\colon E &\to F^{\star} \\
  u &\mapsto \left(v\mapsto\left\langle v,u\right\rangle\right),
\end{align*}
$$

both denoted as $\iota$, are injective, that is, are linear embeddings.

Since we obviously do not want to get our hands dirty with dull pure algebra😗, let us put some topologies on $E,F$ and see what happens. Specifically, a natural question to ask is, *when $F$ is precisely the continuous dual space of $E$*? That is, *which topologies on $E$ makes $F$ the continuous dual space of $E$*? The [Mackey-Arens theorem](https://en.wikipedia.org/wiki/Mackey_topology) answers this question.

Given a topological vector space $(E,\mathscr{T})$, let $(E,\mathscr{T})'$ denotes the *continuous dual space* of $(E,\mathscr{T})$, that is, the linear space of all linear functionals on $E$ that are continuous with respect to the topology $\mathscr{T}$. If $\mathscr{T}$ is obvious, we may omit it and just write $E'$.

>**Definition 2 (Dual topologies).**
>
>Let $E,F$ be vector spaces over $\mathbb{K}=\mathbb{R}$ or $\mathbb{C}$ and $\left\langle\,\cdot\,,\,\cdot\,\right\rangle\colon F\times E\to \mathbb{K}$ a duality pairing. Then a linear topology $\mathscr{T}$ on $E$ is said to be a *dual topology* (with respect to the pairing $\left\langle\,\cdot\,,\,\cdot\,\right\rangle$) if $\iota[F] = (E,\mathscr{T})'$. Dual topologies on $F$ with respect to the pairing are similarly defined.

><b id="mackey-arens">Theorem 3</b> **(Mackey-Arens).**
>
>Let $E,F$ be vector spaces over $\mathbb{K}=\mathbb{R}$ or $\mathbb{C}$ and $\left\langle\,\cdot\,,\,\cdot\,\right\rangle\colon F\times E\to \mathbb{K}$ a duality pairing. Then:
>1. $\iota[F]\subseteq (E,\mathscr{T})'$ if and only if $\mathscr{T}\supseteq\sigma(E,F)$, and
>2. $\iota[F]\supseteq (E,\mathscr{T})'$ if and only if $\mathscr{T}\subseteq\tau(E,F)$.
>
>In particular, $\mathscr{T}$ is a dual topology if and only if
>
>$$
>  \sigma(E,F) \subseteq \mathscr{T} \subseteq \tau(E,F).
>$$
>

The definitions of the topologies $\sigma(E,F)$ and $\tau(E,F)$ are given below.

This theorem is basically a consequence of the [bipolar theorem](https://en.wikipedia.org/wiki/Bipolar_theorem) (which in turn is a consequence of the [Hahn-Banach theorem](https://en.wikipedia.org/wiki/Hahn%E2%80%93Banach_theorem)) and the [Banach-Alaoglu theorem](https://en.wikipedia.org/wiki/Banach%E2%80%93Alaoglu_theorem) (which in turn is a consequence of the [Arzelà–Ascoli theorem](https://en.wikipedia.org/wiki/Arzel%C3%A0%E2%80%93Ascoli_theorem)). Detailed proof of this theorem is not given in this post.

>**Definition 4 (Weak topology).**
>
>Let $E,F$ be vector spaces over $\mathbb{K}=\mathbb{R}$ or $\mathbb{C}$ and $\left\langle\,\cdot\,,\,\cdot\,\right\rangle\colon F\times E\to \mathbb{K}$ a duality pairing. Then the *weak topology* on $E$ (with respect to the pairing $\left\langle\,\cdot\,,\,\cdot\,\right\rangle$) is the [initial topology](https://en.wikipedia.org/wiki/Initial_topology) generated by $\iota[F]$, and is denoted as $\sigma(E,F)$. Equivalently, $\sigma(E,F)$ is the topology generated by the collection of all seminorms on $E$ of the form
>
>$$
>\begin{align*}
>  E&\to [0,\infty) \\
>  u&\mapsto\vert \left\langle v,u\right\rangle\vert
>\end{align*}
>$$
>
>for any $v\in F$.

>**Definition 5 (Mackey topology).**
>
>Let $E,F$ be vector spaces over $\mathbb{K}=\mathbb{R}$ or $\mathbb{C}$ and $\left\langle\,\cdot\,,\,\cdot\,\right\rangle\colon F\times E\to \mathbb{K}$ a duality pairing. Then the *Mackey topology* on $E$ (with respect to the pairing $\left\langle\,\cdot\,,\,\cdot\,\right\rangle$) is the topology generated by the collection of all seminorms on $E$ of the form
>
>$$
>\begin{align*}
>  E&\to [0,\infty) \\
>  u&\mapsto \sup_{v\in K}\vert \left\langle v,u\right\rangle\vert
>\end{align*}
>$$
>
>for any [absolutely convex](https://en.wikipedia.org/wiki/Absolutely_convex_set) $\sigma(F,E)$-compact subsets of $K$.

## A generalization of the Lax-Milgram theorem

The theorem we want to prove is the following:

>**Theorem 6 (Lax-Milgram).**
>
>Let $E,F$ be [locally convex spaces](https://en.wikipedia.org/wiki/Locally_convex_topological_vector_space) over $\mathbb{K}=\mathbb{R}$ or $\mathbb{C}$ and $B\colon F\times E\to \mathbb{K}$ a [separately continuous](https://en.wikipedia.org/wiki/Locally_convex_topological_vector_space) bilinear map. Define
>
>$$
>\begin{align*}
>  L\colon E&\to F' \\
>  u&\mapsto \left(v\mapsto B(v,u)\right)
>\end{align*}
>$$
>
>and
>
>$$
>\begin{align*}
>  R\colon F&\to E' \\
>  v&\mapsto \left(u\mapsto B(v,u)\right).
>\end{align*}
>$$
>
>Then the followings are equivalent:
>1. $L$ is surjective.
>2. $R$ is injective and $R^{-1}\colon \left(R[F],\sigma(E',E)\right)\to \left(F,\sigma(F,F')\right)$ is continuous.
>3. $R$ is injective and $R^{-1}\colon \left(R[F],\tau(E',E)\right)\to \left(F,\sigma(F,F')\right)$ is continuous.

Note that $B$ being separately continuous precisely means that $L,R$ are well-defined.

>**Proof.** $(1\Rightarrow 2)$ Suppose $v\in F\setminus\{0\}$. By [Hahn-Banach theorem](https://en.wikipedia.org/wiki/Hahn%E2%80%93Banach_theorem), we can find $$v^{*}\in F'$$ such that $$v^{*}(v) \neq 0$$. Then by the assumption, there exists $u\in E$ such that $$Lu = v^{*}$$. Then
>
>$$
>  0\neq v^{*}(v) = (Lu)(v) = B(v,u) = (Rv)(u),
>$$
>
>thus $Rv\neq 0$. This shows that $R$ is injective.
>
>To show continuity of $R^{-1}$, let $$\left(v_{\alpha}\right)_{\alpha\in D}$$ be a [net](https://en.wikipedia.org/wiki/Net_(mathematics)) in $F$ such that $$\left(Rv_{\alpha}\right)_{\alpha\in D}$$ is convergent to $Rv$ with respect to $\sigma(E',E)$ for some $v\in F$. We want to show that $$\left(v_{\alpha}\right)_{\alpha\in D}$$ is convergent to $v$ with respect to $\sigma(F,F')$, which means nothing but that $$\left(v^{*}(v_{\alpha})\right)_{\alpha\in D}$$ converges to $$v^{*}(v)$$ for all $$v^{*}\in F'$$. Given $$v^{*}\in F'$$, by the assumption there exists $u\in E$ such that $$Lu=v^{*}$$. Then $$v^{*}(v_{\alpha}) = (Lu)(v_{\alpha}) = B(v_{\alpha},u) = (Rv_{\alpha})(u)$$, so $$\left(v^{*}(v_{\alpha})\right)_{\alpha\in D}$$ converges to $$(Rv)(u) = v^{*}(v)$$ as desired.
>
>$(2\Rightarrow 3)$ Trivial, since $\tau(E',E)$ is finer than or equal to $\sigma(E',E)$.
>
>$(3\Rightarrow 1)$ Take an arbitrary $$v^{*}\in F'$$. Consider a linear functional
>
>$$
>\begin{align*}
>  \lambda\colon R[F] &\to \mathbb{K} \\
>  Rv &\mapsto v^{*}(v).
>\end{align*}
>$$
>
>By the assumption, $\lambda$ is continuous with respect to $\tau(E',E)$. Hence, by [Hahn-Banach theorem](https://en.wikipedia.org/wiki/Hahn%E2%80%93Banach_theorem), there exists a linear extension $\tilde{\lambda}\colon E'\to \mathbb{K}$ of $\lambda$ which is continuous with respect to $\tau(E',E)$. Therefore, by [**Theorem 3**](#mackey-arens), there exists $u\in E$ such that $\iota(u) = \tilde{\lambda}$, that is, $$u^{*}(u) = \tilde{\lambda}(u^{*})$$ holds for all $$u^{*}\in E'$$. Then, for any $v\in F$,
>
>$$
>  (Lu)(v) = B(v,u) = (Rv)(u) = \tilde{\lambda}(Rv) = \lambda(Rv) = v^{*}(v),
>$$
>
>concluding $$Lu = v^{*}$$. Therefore, $L$ is surjective.$\quad\blacksquare$

