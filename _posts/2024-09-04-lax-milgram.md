---
title: "A generalization of the Lax-Milgram Theorem"
date: 2024-09-04
permalink: /posts/2024/09/lax-milgram/
tags:
  - analysis
  - distribution-theory
  - functional-analysis
  - pde
---

This is a short note on a generalization of the [Lax-Milgram theorem](https://en.wikipedia.org/wiki/Weak_formulation#The_Lax%E2%80%93Milgram_theorem), which is a common tool for proving existence of weak solutions to PDE.

## Introduction

Let $\Omega\subseteq\mathbb{R}^{n}$ be a bounded and smooth enough domain and $f\in L^{2}(\Omega)$. Consider the PDE

$$
\begin{cases}
\begin{aligned}
  -\Delta u &= f & \textrm{on}\quad & \Omega \\
  u &= 0 & \textrm{on}\quad & \partial\Omega.
\end{aligned}
\end{cases}
$$

Suppose that $u\in C^{2}(\overline{\Omega})$ is a solution to this equation. Then for each $\phi\in\mathcal{C}_{c}^{\infty}(\Omega)$, we have

$$
\begin{equation*}
  \int_{\Omega}f\phi = -\int_{\Omega} \phi\Delta u
  = -\int_{\Omega}\nabla\cdot(\phi\nabla u)
  + \int_{\Omega}\nabla\phi\cdot\nabla u
  = \int_{\Omega}\nabla\phi\cdot \nabla u,
\end{equation*}
$$

where the last equality follows from the [divergence theorem](https://en.wikipedia.org/wiki/Divergence_theorem) and that $\phi\vert_{\partial\Omega} = 0$. Since having

$$
\begin{equation*}
  \int_{\Omega}f\phi = -\int_{\Omega}\phi\Delta u
\end{equation*}
$$

for all $$\phi\in\mathcal{C}_{c}^{\infty}(\Omega)$$ is equivalent to $-\Delta u = f$ (which follows from the [Lebesgue differentiation theorem](https://en.wikipedia.org/wiki/Lebesgue_differentiation_theorem)), we conclude that $u$ is a solution if and only if

$$
\begin{equation*}
  \int_{\Omega}f\phi = \int_{\Omega}\nabla\phi\cdot\nabla u
\end{equation*}
$$

holds for all $$\phi\in\mathcal{C}_{c}^{\infty}(\Omega)$$. Since both sides define a continuous linear functional on the Hilbert space $$H_{0}^{1}(\Omega)$$ (the norm-closure of $$\mathcal{C}_{c}^{\infty}(\Omega)$$ inside the [Sobolev space](https://en.wikipedia.org/wiki/Sobolev_space) $H^{1}(\Omega)$), we have converted the problem of solving the PDE into the problem of finding an element $$u\in H_{0}^{1}(\Omega)$$ such that the continuous linear functional $$\phi\mapsto\int_{\Omega}\nabla\phi\cdot\nabla u$$ coincides with $$\phi\mapsto\int_{\Omega}f\phi$$. Thus, basically what we are asking here is the surjectivity of the linear map

$$
\begin{align*}
  L\colon H_{0}^{1}(\Omega) &\to H_{0}^{1}(\Omega)^{*} \\
  u &\mapsto \left(\phi\mapsto \int_{\Omega}\nabla\phi\cdot\nabla u\right).
\end{align*}
$$

This is called the *weak formulation* of the PDE. An element $$u\in H_{0}^{1}(\Omega)$$ satisfying $$Lu = \left(\phi\mapsto\int_{\Omega}f\phi\right)$$ is then called a *weak solution* of the PDE. Once we find a weak solution, then some other machineries can be applied to show uniqueness and regularity of the found weak solution.

Therefore, we can reformulate our PDE into the following functional analysis question: given a bilinear form $B\colon E\times F\to \mathbb{K}$ where $\mathbb{K}=\mathbb{R}$ or $\mathbb{C}$ is the scalar field, when the induced map

$$
\begin{aligned}
  L\colon E &\to F^{*} \\
  u &\mapsto \left(v\mapsto B(u,v)\right)
\end{aligned}
$$

is surjective?

The [Lax-Milgram theorem](https://en.wikipedia.org/wiki/Weak_formulation#The_Lax%E2%80%93Milgram_theorem) is an answer to this question: $L$ is surjective when $B$ is *coercive*. I will define the term *coercive bilinear form* later, but here let me point out that this concept is somewhat quantitatively defined. I wanted to see what is "the most purely qualitative" characterization of the condition that ensures surjectivity, and got a somewhat simple answer.

## Duality pairings and Mackey-Arens theorem

A lesson I learned from functional analysis is that every attempt for building a duality theory that encompasses, but goes beyond, the usual duality theory of Banach spaces will necessarily suffer, suffer quite a lot. This is not because the concept of duality is broken beyond Banach spaces, rather because it is already broken in the Banach space level. More precisely, the notion of "the correct dual space" is not always the usual one, the space of all continuous linear functionals with the [operator norm](https://en.wikipedia.org/wiki/Operator_norm). Rather, the correct dual space very much depends on the given situation, and there is no one-size-fit-all answer.

This is why it is a good idea to temporarily forget about the concept of continuous dual, and start with a so-called *duality pairing* given on an arbitrary pair $(E,F)$ of spaces, which a priori has nothing to with any kinds of topologies we can give on those spaces.

>**Definition 1 (Duality pairing).**
>
>Let $E,F$ be vector spaces over $\mathbb{K}=\mathbb{R}$ or $\mathbb{C}$. A bilinear map $\left\langle\,\cdot\,,\,\cdot\,\right\rangle\colon F\times E\to \mathbb{K}$ is called a **duality pairing** on $(E,F)$ if:
>1. for any $v\in E$, $\left\langle v,u\right\rangle = 0$ for all $u\in F$ implies $v=0$, and
>2. for any $u\in F$, $\left\langle v,u\right\rangle = 0$ for all $v\in E$ implies $u=0$.
>

For a vector space $E$, let $E^{\star}$ denotes the *algebraic dual* of $E$, that is, the vector space of all linear functionals on $E$ with no further constraints. Then the conditions given in the definition above simply means that the induced maps

$$
\begin{aligned}
  \iota\colon F &\to E^{\star} \\
  v &\mapsto \left(u\mapsto\left\langle v,u\right\rangle\right)
\end{aligned}
$$

and

$$
\begin{aligned}
  \iota\colon E &\to F^{\star} \\
  u &\mapsto \left(v\mapsto\left\langle v,u\right\rangle\right),
\end{aligned}
$$

both denoted as $\iota$, are injective, that is, are linear embeddings.

Since we obviously do not want to get our hands dirty with dull pure algebra😗, let us put some topologies on $E,F$ and see what happens. Specifically, a natural question to ask is, *when $F$ is precisely the continuous dual space of $E$*? That is, *which topologies on $E$ makes $F$ the continuous dual space of $E$*? The [Mackey-Arens theorem](https://en.wikipedia.org/wiki/Mackey_topology) answers this question.

Given a topological vector space $(E,\mathscr{T})$, let $(E,\mathscr{T})'$ denotes the *continuous dual space* of $(E,\mathscr{T})$, that is, the linear space of all linear functionals on $E$ that are continuous with respect to the topology $\mathscr{T}$. If $\mathscr{T}$ is obvious, we may omit it and just write $E'$.

>**Definition 2 (Dual topologies).**
>
>Let $E,F$ be vector spaces over $\mathbb{K}=\mathbb{R}$ or $\mathbb{C}$ and $\left\langle\,\cdot\,,\,\cdot\,\right\rangle\colon F\times E\to \mathbb{K}$ a duality pairing. Then a linear topology $\mathscr{T}$ on $E$ is said to be a **dual topology** (with respect to the pairing $\left\langle\,\cdot\,,\,\cdot\,\right\rangle$) if $\iota[F] = (E,\mathscr{T})'$. Dual topologies on $F$ with respect to the pairing are similarly defined.

><b id="mackey-arens">Theorem 3</b> **(Mackey-Arens).**
>
>Let $E,F$ be vector spaces over $\mathbb{K}=\mathbb{R}$ or $\mathbb{C}$ and $\left\langle\,\cdot\,,\,\cdot\,\right\rangle\colon F\times E\to \mathbb{K}$ a duality pairing. Then:
>1. $\iota[F]\subseteq (E,\mathscr{T})'$ if and only if $\mathscr{T}\supseteq\sigma(E,F)$, and
>2. $\iota[F]\supseteq (E,\mathscr{T})'$ if and only if $\mathscr{T}\subseteq\tau(E,F)$.
>
>In particular, $\mathscr{T}$ is a dual topology if and only if
>
>$$
>\begin{equation*}
>  \sigma(E,F) \subseteq \mathscr{T} \subseteq \tau(E,F).
>\end{equation*}
>$$
>

The definitions of the topologies $\sigma(E,F)$ and $\tau(E,F)$ are given below.

This theorem is basically a consequence of the [bipolar theorem](https://en.wikipedia.org/wiki/Bipolar_theorem) (which in turn is a consequence of the [Hahn-Banach theorem](https://en.wikipedia.org/wiki/Hahn%E2%80%93Banach_theorem)) and the [Banach-Alaoglu theorem](https://en.wikipedia.org/wiki/Banach%E2%80%93Alaoglu_theorem) (which in turn is a consequence of the [Arzelà–Ascoli theorem](https://en.wikipedia.org/wiki/Arzel%C3%A0%E2%80%93Ascoli_theorem)). Detailed proof of this theorem is not given in this post.

>**Definition 4 (Weak topology).**
>
>Let $E,F$ be vector spaces over $\mathbb{K}=\mathbb{R}$ or $\mathbb{C}$ and $\left\langle\,\cdot\,,\,\cdot\,\right\rangle\colon F\times E\to \mathbb{K}$ a duality pairing. Then the **weak topology** on $E$ (with respect to the pairing $\left\langle\,\cdot\,,\,\cdot\,\right\rangle$) is the [initial topology](https://en.wikipedia.org/wiki/Initial_topology) generated by $\iota[F]$, and is denoted as $\sigma(E,F)$. Equivalently, $\sigma(E,F)$ is the topology generated by the collection of all seminorms on $E$ of the form
>
>$$
>\begin{aligned}
>  E&\to [0,\infty) \\
>  u&\mapsto\vert \left\langle v,u\right\rangle\vert
>\end{aligned}
>$$
>
>for any $v\in F$.
>

>**Definition 5 (Mackey topology).**
>
>Let $E,F$ be vector spaces over $\mathbb{K}=\mathbb{R}$ or $\mathbb{C}$ and $\left\langle\,\cdot\,,\,\cdot\,\right\rangle\colon F\times E\to \mathbb{K}$ a duality pairing. Then the **Mackey topology** on $E$ (with respect to the pairing $\left\langle\,\cdot\,,\,\cdot\,\right\rangle$) is the topology generated by the collection of all seminorms on $E$ of the form
>
>$$
>\begin{aligned}
>  E&\to [0,\infty) \\
>  u&\mapsto \sup_{v\in K}\vert \left\langle v,u\right\rangle\vert
>\end{aligned}
>$$
>
>for any [absolutely convex](https://en.wikipedia.org/wiki/Absolutely_convex_set) $\sigma(F,E)$-compact subsets $K$ of $F$.
>

For each $u\in E$, $v\mapsto\left\langle v,u\right\rangle$ is a $\sigma(F,E)$-continuous linear functional on $F$, thus $$\sup_{v\in K}\vert\left\langle v,u\right\rangle\vert$$ is finite (and the supremum is achieved) for any $\sigma(F,E)$-compact subset $K$ of $F$. Hence,

$$
\begin{aligned}
  E&\to [0,\infty) \\
  u&\mapsto \sup_{v\in K}\vert \left\langle v,u\right\rangle\vert
\end{aligned}
$$

is indeed a seminorm for such $K$.

When $E$ is a normed space, the Mackey topology $\tau(E,E')$ is nothing but the norm topology. Indeed, recall tha the norm topology on $E$ is the subspace topology inherited from the norm topology on $E''$, which is the topology of uniform convergence on norm-bounded subsets of $E'$. By the [Banach-Alaoglu theorem](https://en.wikipedia.org/wiki/Banach%E2%80%93Alaoglu_theorem), any norm-bounded subset of $E'$ is contained in an absolutely convex weak-$$*$$ compact subset of $E'$, and weak-$$*$$ topology is nothing but the weak topology $\sigma(E',E)$. Therefore, the norm topology on $E$ is coarser than or equal to $\tau(E,E')$. On the other hand, the [uniform boundedness principle](https://en.wikipedia.org/wiki/Uniform_boundedness_principle) shows that any $\sigma(E',E)$-[bounded subsets](https://en.wikipedia.org/wiki/Bounded_set_(topological_vector_space)) (thus $\sigma(E',E)$-compact subsets in particular) of $E$ are norm-bounded, thus the Mackey topology is coarser than or equal to the norm topology, concluding the equivalence between the two.

It is worth noting that, on the other hand, $\tau(E',E)$ is *not* the operator norm topology. Since the operator norm topology coincides with the Mackey topology $\tau(E',E'')$, we conclude that $\tau(E',E)$ coincides with the operator norm topology if $E$ is reflexive. Of course, the [Mackey-Arens theorem](#mackey-arens) shows that the converse is also true.

## A generalization of the Lax-Milgram theorem

The theorem we want to prove is the following:

><b id="lax-milgram">Theorem 6</b> **(Lax-Milgram).**
>
>Let $E,F$ be [locally convex spaces](https://en.wikipedia.org/wiki/Locally_convex_topological_vector_space) over $\mathbb{K}=\mathbb{R}$ or $\mathbb{C}$ and $B\colon F\times E\to \mathbb{K}$ a bilinear form. Define
>
>$$
>\begin{aligned}
>  L\colon E&\to F^{\star} \\
>  u&\mapsto \left(v\mapsto B(v,u)\right)
>\end{aligned}
>$$
>
>and
>
>$$
>\begin{aligned}
>  R\colon F&\to E^{\star} \\
>  v&\mapsto \left(u\mapsto B(v,u)\right),
>\end{aligned}
>$$
>
>and suppose that $R[F] \subseteq E'$. Then the followings are equivalent:
>1. $L[E] \supseteq F'$.
>2. $R$ is injective and $R^{-1}\colon \left(R[F],\mathscr{T}\right)\to \left(F,\sigma(F,F')\right)$ is continuous for all dual topologies $\mathscr{T}$ on $E'$ with respect to the canonical pairing between $E'$ and $E$.
>3. $R$ is injective and $R^{-1}\colon \left(R[F],\mathscr{T}\right)\to \left(F,\sigma(F,F')\right)$ is continuous for some dual topology $\mathscr{T}$ on $E'$ with respect to the canonical pairing between $E'$ and $E$.
>

Note that the only role that the topologies on $E,F$ play is on defining the dual spaces $E',F'$: it does not matter which dual topology we choose.

>**Proof.** $(1\Rightarrow 2)$ Suppose $$v\in F\setminus\{0\}$$. By [Hahn-Banach theorem](https://en.wikipedia.org/wiki/Hahn%E2%80%93Banach_theorem), we can find $$v^{*}\in F'$$ such that $$v^{*}(v) \neq 0$$. Then by the assumption, there exists $u\in E$ such that $$Lu = v^{*}$$. Then
>
>$$
>\begin{equation*}
>  0\neq v^{*}(v) = (Lu)(v) = B(v,u) = (Rv)(u),
>\end{equation*}
>$$
>
>thus $Rv\neq 0$. This shows that $R$ is injective.
>
>For continuity of $R^{-1}$, it is enough to show that $R^{-1}$ is continuous when $\mathscr{T}=\sigma(E',E)$ because any dual topology should contain $\sigma(E',E)$ by the [Mackey-Arens theorem](#mackey-arens). Let $$\left(v_{\alpha}\right)_{\alpha\in D}$$ be a [net](https://en.wikipedia.org/wiki/Net_(mathematics)) in $F$ such that $$\left(Rv_{\alpha}\right)_{\alpha\in D}$$ is convergent to $Rv$ with respect to $\sigma(E',E)$ for some $v\in F$. We want to show that $$\left(v_{\alpha}\right)_{\alpha\in D}$$ is convergent to $v$ with respect to $\sigma(F,F')$, which means nothing but that $$\left(v^{*}(v_{\alpha})\right)_{\alpha\in D}$$ converges to $$v^{*}(v)$$ for all $$v^{*}\in F'$$. Given $$v^{*}\in F'$$, by the assumption there exists $u\in E$ such that $$Lu=v^{*}$$. Then $$v^{*}(v_{\alpha}) = (Lu)(v_{\alpha}) = B(v_{\alpha},u) = (Rv_{\alpha})(u)$$, so $$\left(v^{*}(v_{\alpha})\right)_{\alpha\in D}$$ converges to $$(Rv)(u) = v^{*}(v)$$ as desired.
>
>$(2\Rightarrow 3)$ Trivial.
>
>$(3\Rightarrow 1)$ Take an arbitrary $$v^{*}\in F'$$. Consider a linear functional
>
>$$
>\begin{aligned}
>  \lambda\colon R[F] &\to \mathbb{K} \\
>  Rv &\mapsto v^{*}(v).
>\end{aligned}
>$$
>
>By the assumption, $\lambda$ is continuous with respect to $\mathscr{T}$. Hence, by [Hahn-Banach theorem](https://en.wikipedia.org/wiki/Hahn%E2%80%93Banach_theorem), there exists a linear extension $\tilde{\lambda}\colon E'\to \mathbb{K}$ of $\lambda$ which is continuous with respect to $\mathscr{T}$. Therefore, by the [Mackey-Arens theorem](#mackey-arens), there exists $u\in E$ such that $\iota(u) = \tilde{\lambda}$, that is, $$u^{*}(u) = \tilde{\lambda}(u^{*})$$ holds for all $$u^{*}\in E'$$. Then, for any $v\in F$,
>
>$$
>\begin{equation*}
>  (Lu)(v) = B(v,u) = (Rv)(u) = \tilde{\lambda}(Rv) = \lambda(Rv) = v^{*}(v),
>\end{equation*}
>$$
>
>concluding $$Lu = v^{*}$$. Therefore, $L[E]$ contains $F'$.$\quad\blacksquare$

In application, a usual assumption is that $E$ is a reflexive Banach space. By definition, the usual operator norm topology on $E'$ is a dual topology with respect to the pairing between $E'$ and $E$, thus in that case usually we want to verify the continuity of $R^{-1}\colon R[F]\to F$ with respect to the norm topology on $R[F]\subseteq E'$. As noted earlier, in this situation the operator norm topology coincides with the Mackey topology $\tau(E',E)$.

When $E$ is not reflexive, continuity of $R^{-1}$ with respect to the norm topology is not enough, because the norm topology on $E'$ is in general finer than $\tau(E',E)$. Hence, we have to work with $\tau(E',E)$ instead. Or it could be any other dual topology, but in principle $\tau(E',E)$ is the finest one so the continuity must be easiest to show when we endow $E'$ with $\tau(E',E)$. To work with $\tau(E',E)$, we should first characterize absolutely convex $\sigma(E,E')$-compact sets, in other words, absolutely convex *weakly compact* subsets of $E$. For instance, when $E = L^{1}(\mu)$ for some measure $\mu$, we may need to use the [Dunford-Pettis theorem](https://en.wikipedia.org/wiki/Uniform_integrability#Relevant_theorems).

It may seem that showing continuity of $R^{-1}\colon R[F]\to F$ when the codomain $F$ is endowed with the weak topology $\sigma(F,F')$ is easier than when $F$ is endowed with a finer topology, for instance a norm topology. However, this is in fact not the case when both $E$ and $F$ are normed spaces and $R\colon F\to E'$ is not only well-defined but also continuous. When $E,F$ are normed spaces, it is often the case that the given bilinear map $B$ is also [jointly continuous](https://en.wikipedia.org/wiki/Bilinear_map#Continuity_and_separate_continuity), thus this continuity condition on $R$ is easily guaranteed.

Now, suppose that $E,F$ are normed spaces and $R\colon F\to E'$ is continuous. We claim that in this case, if $R$ is injective and

$$
\begin{equation*}
  R^{-1}\colon (R[F],\|\cdot\|_{E'})\to (F,\sigma(F,F'))
\end{equation*}
$$

is continuous, then

$$
\begin{equation*}
  R^{-1}\colon (R[F],\|\cdot\|_{E'})\to (F,\|\cdot\|_{F})
\end{equation*}
$$

is continuous as well. Note that the continuity of the first map means nothing but that the topology on $F$ induced by the norm $$v\mapsto \|Rv\|_{E'}$$ is finer than or equal to $\sigma(F,F')$. On the other hand, continuity of $R\colon F\to E'$ means that the topology induced by this new norm is coarser than or equal to the original norm topology on $F$. Hence, by the [Mackey-Arens theorem](#mackey-arens), this new norm must induce exactly the same dual space $F'$, but this means that the norm topology induced by it is nothing but the Mackey topology $\tau(F,F')$, which is nothing but the original norm topology on $F$. Hence, $R^{-1}$ should be continuous when the codomain is endowed with the norm topology as well.

## Recovering the classical case

When $E$ is a reflexive Banach space, a usual way of verifying the continuity condition required in [**Theorem 6**](#lax-milgram) is to show that the bilinear form $B$ is *coercive*:

>**Definition 8 (Coercive bilinear forms).**
>
>Let $E,F$ be normed spaces over $\mathbb{K}=\mathbb{R}$ or $\mathbb{C}$. Then a bilinear form $B\colon F\times E\to\mathbb{K}$ is said to be **coercive** if there exists a constant $c>0$ such that
>
>$$
>\begin{equation*}
> \inf_{v\in F\setminus\{0\}}\sup_{u\in E\setminus\{0\}}
> \frac{|B(v,u)|}{\|u\|_{E}\|v\|_{F}} \geq c.
>\end{equation*}
>$$
>

Note that coercivity is really nothing but the norm-continuity of the map $R^{-1}\colon R[F]\to F$. Indeed, recall the map $R\colon v\mapsto \left(u\mapsto B(v,u)\right)$ from $F$ to $E^{\star}$, then

$$
\begin{equation*}
  \|Rv\|_{E'} = \sup_{u\in E\setminus\{0\}}\frac{|(Rv)(u)|}{\|u\|_{E}}
  = \sup_{u\in E\setminus\{0\}}\frac{|B(v,u)|}{\|u\|_{E}},
\end{equation*}
$$

so $B$ being coercive is equivalent to that there exists a constant $c>0$ such that

$$
\begin{equation*}
  \|v\|_{F}\leq \frac{1}{c}\|Rv\|_{E'}.
\end{equation*}
$$

Hence, we obtain the following corollary, which is a more common way the Lax-Milgram theorem is stated:

>**Corollary 9 ([Lions-Lax-Milgram](https://en.wikipedia.org/wiki/Lions%E2%80%93Lax%E2%80%93Milgram_theorem)).**
>
>Let $E$ be a reflexive Banach space over $\mathbb{K}=\mathbb{R}$ or $\mathbb{C}$, $F$ a normed space over $\mathbb{K}$, and $B\colon F\times E\to \mathbb{K}$ a bilinear form. Define $L\colon E\to F^{\star}$ and $R\colon F\to E^{\star}$ as in [**Theorem 6**](#lax-milgram) and suppose that $R[F]\subseteq E'$ and $R\colon F\to E'$ is continuous. Then $L[E] \supseteq F'$ holds if and only if $B$ is coercive.
>

As an example application, let us go back to the PDE we saw in the [Introduction](#introduction):

$$
\begin{cases}
\begin{aligned}
  -\Delta u &= f & \textrm{on}\quad & \Omega \\
  u &= 0 & \textrm{on}\quad & \partial\Omega.
\end{aligned}
\end{cases}
$$

In this case, the bilinear form we are looking at is

$$
\begin{aligned}
  B\colon H_{0}^{1}(\Omega)\times H_{0}^{1}(\Omega) &\to \mathbb{K} \\
  (\phi,u) &\mapsto \int_{\Omega}\nabla\phi \cdot \nabla u.
\end{aligned}
$$

Since $$B(\overline{\phi},\phi) = \|\phi\|_{L^{2}(\Omega)}^{2}$$, that this bilinear form is coercive follows immediately from the [Poincaré inequality](https://en.wikipedia.org/wiki/Poincar%C3%A9_inequality). Therefore, we can apply the corollary above and conclude that there exists $u\in H_{0}^{1}(\Omega)$ such that

$$
\begin{equation*}
  \int_{\Omega}\nabla\phi \cdot \nabla u = \lambda(\phi)
\end{equation*}
$$

holds for all $\phi\in H_{0}^{1}(\Omega)$, for any given $$\lambda\in H_{0}^{1}(\Omega)^{*}$$. This space $$H_{0}^{1}(\Omega)^{*}$$ in particular contains all integrations against functions in $L^{2}(\Omega)$, but it contains many more elements.

## A non-Banach application

Let's have some fun with a case involving non-Banach locally convex spaces. Let us consider the bilinear form

$$
\begin{aligned}
  B\colon \mathcal{C}_{c}^{\infty}(\Omega)\times \mathcal{C}_{c}^{\infty}(\Omega)'
  &\to \mathbb{K} \\
  (\phi,u) &\mapsto -\left\langle \Delta u,\phi\right\rangle
  \mathrel{\unicode{x2254}} -\left\langle u,\Delta\phi\right\rangle,
\end{aligned}
$$

where $$\mathcal{C}_{c}^{\infty}(\Omega)$$ is endowed with the usual [LF-topology](https://en.wikipedia.org/wiki/LF-space). Here, we are interested in the surjectivity of the induced map

$$
\begin{aligned}
  L\colon \mathcal{C}_{c}^{\infty}(\Omega)' &\to \mathcal{C}_{c}^{\infty}(\Omega)' \\
  u &\mapsto -\Delta u,
\end{aligned}
$$

or putting differently, we are asking the question: *can we always invert the Laplacian applied to an arbitrary [distribution](https://en.wikipedia.org/wiki/Distribution_(mathematics))?*

Let us try to apply [**Theorem 6**](#lax-milgram), so we consider the other induced map

$$
\begin{aligned}
  R\colon \mathcal{C}_{c}^{\infty}(\Omega)&\to
  \left(\mathcal{C}_{c}^{\infty}(\Omega)'\right)^{\star} \\
  \phi &\mapsto \left(u\mapsto -\left\langle u,\Delta\phi\right\rangle \right).
\end{aligned}
$$

First, we are interested in checking the condition

$$
\begin{equation*}
  R\left[\mathcal{C}_{c}^{\infty}(\Omega)\right] \subseteq \mathcal{C}_{c}^{\infty}(\Omega)''.
\end{equation*}
$$

Here, we need to be a bit careful about which topology we should put on the space $$\mathcal{C}_{c}^{\infty}(\Omega)'$$ because there are multiple possible choices. However, as noted earlier, the only relevance of this choice is on determining what is the desired dual space of $$\mathcal{C}_{c}^{\infty}(\Omega)'$$, and there is really just only one choice of the space which we want to be the dual space of $$\mathcal{C}_{c}^{\infty}(\Omega)'$$: the predual $$\mathcal{C}_{c}^{\infty}(\Omega)$$. Hence, let us just choose any dual topology for the pairing between the two, for instance the weak-$$*$$ topology. Then, asking if

$$
\begin{equation*}
  R\left[\mathcal{C}_{c}^{\infty}(\Omega)\right] \subseteq \mathcal{C}_{c}^{\infty}(\Omega)''
  \cong \mathcal{C}_{c}^{\infty}(\Omega)
\end{equation*}
$$

holds really just means asking if $-\Delta\phi$ belongs to $$\mathcal{C}_{c}^{\infty}(\Omega)$$ for any $$\phi\in\mathcal{C}_{c}^{\infty}(\Omega)$$ which is trivially the case.

The next question is whether or not $R$ is injective. This one is indeed easy to answer. Suppose $R\phi = 0$, then we must have $(R\phi)(\tilde{\phi}) = 0$ where $\tilde{\phi}$ is the distribution defined as

$$
\begin{equation*}
  \tilde{\phi}\colon \psi \mapsto \int_{\Omega}\overline{\phi}\psi.
\end{equation*}
$$

Hence,

$$
\begin{equation*}
  0 = -\left\langle\tilde{\phi},\Delta\phi\right\rangle
  = -\int_{\Omega}\overline{\phi}\Delta\phi
  = -\int_{\Omega}\nabla\cdot (\overline{\phi}\nabla\phi)
  - |\nabla\phi|^{2}
  = \int_{\Omega}|\nabla\phi|^{2}
\end{equation*}
$$

by the divergence theorem, so we get $\phi = 0$.

Next, we want to know the continuity of the map

$$
\begin{equation*}
  R^{-1}\colon R\left[\mathcal{C}_{c}^{\infty}(\Omega)\right]
  \to\left(\mathcal{C}_{c}^{\infty}(\Omega),
  \sigma\left(\mathcal{C}_{c}^{\infty}(\Omega),\mathcal{C}_{c}^{\infty}(\Omega)'\right)\right),
\end{equation*}
$$

which is where the "real work" should be done. Let us actually show that $R^{-1}$ is continuous even when the codomain is endowed with the usual LF-topology rather than the weak topology. That is, we want to see that the map

$$
\begin{aligned}
  R\colon \mathcal{C}_{c}^{\infty}(\Omega)&\to
  \mathcal{C}_{c}^{\infty}(\Omega)\\
  \phi &\mapsto -\Delta\phi
\end{aligned}
$$

is open onto its image. Recall that $$\mathcal{C}_{c}^{\infty}(\Omega)$$ is the locally convex direct limit of spaces $$\mathcal{C}_{0}^{\infty}(\operatorname{int}K)$$ for all compact subsets $K$ in $\Omega$, where $$\mathcal{C}_{0}^{\infty}(\operatorname{int}K)$$ is the [Fréchet space](https://en.wikipedia.org/wiki/Fr%C3%A9chet_space) consisting of smooth functions on $\operatorname{int}K$ whose every derivative belongs to the Banach space $$\mathcal{C}_{0}(\operatorname{int}K)$$. Then a basic open neighborhood of $$0\in\mathcal{C}_{c}^{\infty}(\Omega)$$ is the absolutely convex hull of the union of open neighborhoods of $$0\in\mathcal{C}_{0}^{\infty}(\operatorname{int}K)$$. Hence, it turns out, it is enough to show that each

$$
\begin{aligned}
  R_{K}\colon \mathcal{C}_{0}^{\infty}(\operatorname{int}K)&\to
  \mathcal{C}_{0}^{\infty}(\operatorname{int}K)\\
  \phi &\mapsto -\Delta\phi
\end{aligned}
$$

is an open map onto its image.

Assuming that $K$ has sufficiently smooth boundary (which we can), this is again a consequence of the [Poincaré inequality](https://en.wikipedia.org/wiki/Poincar%C3%A9_inequality). Given $$\phi\in\mathcal{C}_{0}^{\infty}(\operatorname{int}K)$$, note that

$$
\begin{equation*}
  \int_{\operatorname{int}K}|\nabla\phi|^{2}
  = -\int_{\operatorname{int}K}\overline{\phi}\Delta\phi
  \leq \|\phi\|_{L^{2}(\operatorname{int}K)}\|\Delta\phi\|_{L^{2}(\operatorname{int}K)}.
\end{equation*}
$$

By the Poincaré inequality, there exists a constant $$C_{K}\in(0,\infty)$$ such that

$$
\begin{equation*}
  \|\phi\|_{L^{2}(\operatorname{int}K)}^{2}
  \leq C_{K}\|\nabla\phi\|_{L^{2}(\operatorname{int}K)}^{2}
  \leq C_{K}\|\phi\|_{L^{2}(\operatorname{int}K)}\|\Delta\phi\|_{L^{2}(\operatorname{int}K)},
\end{equation*}
$$

thus we get

$$
\begin{equation*}
  \|\phi\|_{L^{2}(\operatorname{int}K)}
  \leq C_{K}\|\Delta\phi\|_{L^{2}(\operatorname{int}K)}
  \leq \tilde{C}_{K}\|\Delta\phi\|_{\mathcal{C}_{0}(\operatorname{int}K)}
\end{equation*}
$$

for some different constant $\tilde{C}_{K}$. Now, applying this inequality to derivatives of $\phi$ and then applying the [Sobolev inequality](https://en.wikipedia.org/wiki/Sobolev_inequality#General_Sobolev_inequalities), we get the conclusion that $$\Delta\phi_{\alpha} \to 0$$ implies $$\phi_{\alpha}\to 0$$ in $$\mathcal{C}_{0}^{\infty}(\operatorname{int}K)$$ for any net $$\left(\phi_{\alpha}\right)_{\alpha\in D}$$, which is the desired conclusion.

Therefore, we can now apply [**Theorem 6**](#lax-milgram) to conclude that the map

$$
\begin{aligned}
  L\colon \mathcal{C}_{c}^{\infty}(\Omega)' &\to \mathcal{C}_{c}^{\infty}(\Omega)' \\
  u &\mapsto -\Delta u
\end{aligned}
$$


is surjective.