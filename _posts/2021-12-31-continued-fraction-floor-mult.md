---
title: 'Continued fractions and their application into fast computation of \\(\lfloor nx\rfloor\\)'
date: 2021-01-03
permalink: /posts/2021/12/continued-fraction-floor-mult/
tags:
  - continued fractions
  - programming
  - C/C++
---

When I was working on [Dragonbox](https://github.com/jk-jeon/dragonbox) and [Grisu-Exact](https://github.com/jk-jeon/Grisu-Exact) (which are float-to-string conversion algorithms with some nice properties) I had to come up with a fast method for computing things like $\lfloor n\log_{10}2 \rfloor$ or $\lfloor n\log_{2}10 \rfloor$, or more generally $\lfloor nx\rfloor$ for some integer $n$ and a fixed positive real number $x$. Actually, the sign of $x$ isn't extremely important, but let us just assume $x>0$ for simplicity.

At that time I just took a fairly straightforward approach, which is to compute the multiplication of $n$ with a truncated binary expansion of $x$. That is, select an appropriate nonnegative integer $k$ and then compute $\lfloor \frac{nm}{2^{k}}\rfloor$ where $m$ is $2^{k}$ multiplied by the binary expansion of $x$ up to $k$ th fractional digits, which is a nonnegative integer. Hence, the whole computation can be done with one integer multiplication followed by one arithmetic shift, which is indeed quite fast.

Here is an example. Consider $x=\log_{10}2$. According to [Wolfram Alpha](https://www.wolframalpha.com/input/?i=hex%28log10%282%29%29), it's hexadecimal expansion is `0x0.4d104d427de7fbcc47c4acd605be48...`. If we choose, for example, $k=20$, then $m=$`0x4d104`$=315652$. Hence, we try to compute $\lfloor nx \rfloor$ by computing

$$
  \left\lfloor \frac{315652 \cdot n}{2^{20}}\right\rfloor
$$

instead, or in C code:
```
  (315652 * n) >> 20
```
With a manual verification, it can be seen that this computation is valid for all $\vert n\vert\leq 1650$. However, when $n=1651$, the above formula gives $496$ while the correct value is $497$, and when $n=-1651$, the above gives $-497$ while the correct value is $-498$. Obviously, one would expect that the range of valid inputs will increase if we use a larger value for $k$ (thus using more digits of $x$). That is certainly true in general, but it will at the same time make the computation more prone to overflow.

Note that in order to make this computation truly fast, we want to keep things inside a machine word size. For example, we may want the result of multiplication $nm$ to fit inside 32-bits. Actually, even on 64-bit platforms, 32-bit arithmetic is often faster, so we still want things to be in 32-bits. For example, on platforms like x86_64, (if I recall correctly) there is no instruction for integer multiplication of a register and a 64-bit immediate constant, so if the constant $m$ is of 64-bits, we need to first load it into a register. If $m$ fits inside 32-bits, we don't need to do that because there is a multiplication instruction for 32-bit constants and this results in a faster code.

(Please don't say "you need a benchmark for such a claim". I mean, yeah, that's true, but firstly, once upon a time I witnessed a noticeable performance difference between these two so even though I've thrown out that code into the trash bin long time ago I still believe in this claim to some extent, and secondly and more importantly, difference of 32-bit vs 64-bit is not the point of this post.)

Hence, if for example we choose $k=30$, then $m=323228496$, so we must have $\vert n\vert\leq \lfloor (2^{31}-1)/m \rfloor = 6$ in order to ensure $nm$ to fit in 32-bits.

(In fact, at least on x86_64, it seems that there is no difference in performace of multiplying a 64-bit register and a 32-bit constant to produce a 64-bit number, versus multiplying a 32-bit register and a 32-bit constant to produce a 32-bit number, so we might only want to ensure $m$ to be in 32-bits and $mn$ to be in 64-bits, but let us just assume that we want $nm$ to be also in 32-bits for simplicity. What matters here is that we want to keep things inside a fixed number of bits.)

As a result, these competing requirements will admit a sweet spot that gives the maximum range of $n$. In this specific case, the sweet spot turns out to be $k=21$ and $m=631305$, which allows $n$ up to $\vert n \vert\leq 2620$. 

So far, everything is nice, we obtained a nice formula for computing $\lfloor nx \rfloor$ and we have found a range of $n$ that makes the formula valid. Except that we have zero theoretical estimate on how the size of the range of $n$, $k$, and $x$ are related. I just claimed that the range can be verified manually, which is indeed what I did when working on Dragonbox and Grisu-Exact.

To be precise, I did have some estimate at the time of developing Grisu-Exact, which I briefly explained [here](https://github.com/fmtlib/fmt/issues/1097) (it has little to do with this post so readers don't need to look at it really), but this estimate is too rough and gives a poor bound on the range of valid $n$'s.

Quite recently, I realized that the concept of so called [continued fractions](https://en.wikipedia.org/wiki/Continued_fraction) is a very useful tool for this kind of stuffs.
In a nutshell, the reason why this concept is so useful is that it gives us a way to enumerate all the best rational approximations of any given real number, and from there we can derive a very detailed analysis of the mentioned method of computing $\lfloor nx \rfloor$.

# Continued fractions

A *continued fraction* is a way of writing a number $x$ in a form

$$
  x = a_{0} + \dfrac{b_{1}}{a_{1} + \dfrac{b_{2}}{a_{2} + \dfrac{b_{3}}{a_{3} + \cdots}}},
$$

either as a finite or infinite sequence. We will only consider the case of *simple continued fractions*, which means that all of $b_{1},b_{2},\ \cdots\ $ are 1. From now on, by *a continued fraction* we will always refer to a simple continued fraction. Also, $a_{0}$ is assumed to be an integer while all other $a_{i}$'s are assumed to be positive integers. We denote the continued fraction with the coefficients $a_{0},a_{1},a_{2},\ \cdots\ $ as $[a_{0};a_{1},a_{2},\ \cdots\ ]$.

With these conventions, there is a fairly simple algorithm for obtaining a continued fraction expansion of $x$. First, let $a_{0}:=\lfloor x \rfloor$. Then $x-a_{0}$ is always in the interval $[0,1)$. If it is $0$ that means $x$ is an integer so stop there. Otherwise, compute the reciprocal $x_{1}:= \frac{1}{x-a_{0}}\in(1,\infty)$. Then let $a_{1}:=\lfloor x_{1} \rfloor$, then again $x_{1}-a_{1}$ lies in $[0,1)$. If it is zero, stop, and if not, compute the reciprocal $x_{2}:=\frac{1}{x_{1}-a_{1}}$ and continue.

Here is an example from Wikipedia; consider $x=\frac{415}{93}$. As $415=4\cdot 93 + 43$, we obtain $a_{0}=4$ and $x_{1}=\frac{93}{43}$. Then from $93=2\cdot43+7$ we get $a_{1}=2$ and $x_{2}=\frac{43}{7}$, and similarly $a_{2}=6$, $x_{3}=7$. Since $x_{3}$ is an integer, we get $a_{3}=7$ and the process terminates. As a result, we obtain the continued fraction expansion

$$
  \frac{415}{93} = 4 + \dfrac{1}{2+\dfrac{1}{6+\dfrac{1}{7}}}.
$$

It can be easily shown that this procedure terminates if and only if $x$ is a rational number. In fact, when $x=\frac{p}{q}$ is a rational number (whenever we say $\frac{p}{q}$ is a rational number, we implicitly assumes that it is in its reduced form; that is, $q$ is a positive integer and $p$ is an integer coprime to $q$), then the coefficients $a_{i}$'s are nothing but those appearing in the Euclid algorithm of computing GCD of $p$ and $q$ (which is a priori assumed to be $1$ of course).

When $x$ is rational and $x=[a_{0};a_{1},\ \cdots\ ,a_{i}]$ is the continued fraction expansion of $x$ obtained from the above algorithm, then either $x$ is an integer (so $i=0$) or $a_{i}>1$. Then $[a_{0};a_{1},\ \cdots\ ,a_{i-1},a_{i}-1,1]$ is another continued fraction expansion of $x$. Then it can be shown that these two are the only continued fraction expansions for $x$. For example, we have the following alternative continued fraction expansion

$$
  \frac{415}{93} = 4 + \dfrac{1}{2+\dfrac{1}{6+\dfrac{1}{6+\dfrac{1}{1}}}}
$$

of $\frac{415}{93}$, and these two expansions are the only continued fraction expansions of $\frac{415}{93}$.

When $x$ is an irrational number, we can run the same algorithm to get an infinite sequence, and the obtained continued fraction expansion of $x$ is the unique one. Here are some list of examples from Wikipedia:
- $\sqrt{19} = [4;2,1,3,1,2,8,2,1,3,1,2,8,\ \cdots\ ]$
- $e = [2;1,2,1,1,4,1,1,6,1,1,8,\ \cdots\ ]$
- $\pi = [3;7,15,1,292,1,1,1,2,1,3,1,\ \cdots\ ]$

Also, according to [Wolfram Alpha](https://www.wolframalpha.com/input/?i=log10%282%29), the continued fraction expansion of $\log_{10}2$ is

$$
  \log_{10}2 = [0; 3, 3, 9, 2, 2, 4, 6, 2, 1, 1, 3, 1, 18,\ \cdots\ ].
$$

Given a continued fraction $[a_{0};a_{1},\ \cdots\ ]$, the rational number $[a_{0};a_{1},\ \cdots\ ,a_{i}]$ obtained by truncating the sequence at the $i$ th position is called *the $i$ th convergent* of the continued fraction. As it is a rational number we can write it uniquely as

$$
  [a_{0};a_{1},\ \cdots\, a_{i}] = \frac{p_{i}}{q_{i}}
$$

for a positive integer $q_{i}$ and an integer $p_{i}$ coprime to $q_{i}$. Then one can derive a recurrence relation

$$
  \begin{cases}
    p_{i} = p_{i-2} + a_{i}p_{i-1}, &\\
    q_{i} = q_{i-2} + a_{i}q_{i-1}
  \end{cases}
$$

with the initial conditions $(p_{-2},q_{-2}) = (0,1)$ and $(p_{-1},q_{-1})=(1,0)$.

For example, for $\log_{10}2 = [0; 3, 3, 9, 2, 2, 4, 6, 2, 1, 1, 3, 1, 18,\ \cdots\ ]$, using the recurrence relation we can compute several initial convergents:

$$
  \frac{p_{0}}{q_{0}}=\frac{0}{1},\quad
  \frac{p_{1}}{q_{1}}=\frac{1+3\cdot 0}{0+3\cdot 1}=\frac{1}{3},\quad
  \frac{p_{2}}{q_{2}}=\frac{0+3\cdot 1}{1+3\cdot 3}=\frac{3}{10}, \\
  \frac{p_{3}}{q_{3}}=\frac{1+9\cdot 3}{3+9\cdot 10}=\frac{28}{93},\quad
  \frac{p_{4}}{q_{4}}=\frac{3+2\cdot 28}{10+2\cdot 93}=\frac{59}{196}, \\
  \frac{p_{5}}{q_{5}}=\frac{28+2\cdot 59}{93+2\cdot 196}=\frac{146}{485},\quad
  \frac{p_{6}}{q_{6}}=\frac{59+4\cdot 146}{196+4\cdot 485}=\frac{643}{2136}, \\
  \frac{p_{7}}{q_{7}}=\frac{146+6\cdot 643}{485+6\cdot 2136}=\frac{4004}{13301},
  \frac{p_{8}}{q_{8}}=\frac{643+2\cdot 4004}{2136+2\cdot 13301}=\frac{8651}{28738},
$$

and so on.

Note that the sequence of convergents above is converging to $\log_{10}2$ rapidly. Indeed, the error $\left\vert\frac{p_{2}}{q_{2}} - x\right\vert$ of the second convergent is about $1.03\times 10^{-3}$, and $\left\vert\frac{p_{4}}{q_{4}} - x\right\vert$ is about $9.59\times 10^{-6}$. For the sixth convergent, the error $\left\vert\frac{p_{6}}{q_{6}} - x\right\vert$ is about $3.31\times 10^{-8}$. This is way better than other types of rational approximations, let's say by truncated decimal expansions of $\log_{10}2$, because in that case the denominator must grow approximately as large as $10^{8}$ in order to achieve the error of order $10^{-8}$, but that order of error was achievable by $\frac{p_{6}}{q_{6}}$ whose denomintaor is only $2136$.

## Best rational approximations

A cool thing about continued fractions is that, in fact convergents are ones of the best possible rational approximations in the following sense. Given a real number $x$, a rational number $\frac{p}{q}$ is called a *best rational approximation* of $x$, if for every rational number $\frac{a}{b}$ with $b\leq q$, we always have

$$
  \left\vert x - \frac{p}{q}\right\vert \leq \left\vert x - \frac{a}{b}\right\vert.
$$

So, this means that there is no better approximation of $x$ by rational numbers of denominator no more than $q$.

In fact, whenever $\frac{p}{q}$ is a best rational approximation of $x$ with $q>1$, the equality in the inequality above is only achieved when $\frac{a}{b}=\frac{p}{q}$. Note that $q=1$ corresponds to the uninteresting case of approximating $x$ with integers, and obviously in this case there is the unique best approximation of $x$ for all real numbers $x$ except only when $x=n+\frac{1}{2}$ for some integer $n$, and for those exceptional cases there are precisely two best approximations, namely $n$ and $n+1$.

As hinted, given a continued fraction expansion of $x$, any convergent $\frac{p_{i}}{q_{i}}$ is a best rational approximation of $x$, except possibly for $i=0$ (which *is* also a best rational approximation if and only if $a_{1}>1$). It must be noted that, however, not every best rational approximation of $x$ is obtained as a convergent of its continued fraction expansion.

There is a nice description of all best rational approximations of $x$ in terms of convergents, but in this post that is kind of irrelevant. Rather, what matters for us is a method of enumerating all best rational approximations *from below* and *from above*. (In fact, the aforementioned description of all best rational approximations can be derived from there.)

Formally, we say a rational number $\frac{p}{q}$ is a *best rational approximation from below* of $x$ if $\frac{p}{q}\leq x$ and for any rational number $\frac{a}{b}$ with $\frac{a}{b}\leq x$ and $b\leq q$, we have

$$
  \frac{a}{b}\leq \frac{p}{q} \leq x.
$$

Similarly, we say a rational number $\frac{p}{q}$ is a *best rational approximation from above* of $x$ if $\frac{p}{q}\geq x$ and for any rational number $\frac{a}{b}$ with $\frac{a}{b}\geq x$ and $b\leq q$, we have

$$
  \frac{a}{b}\geq \frac{p}{q} \geq x.
$$

To describe how those best rational approximations from below/above look like, let $x=[a_{0};a_{1},\ \cdots\ ]$ be a continued fraction expansion of a real number $x$ and $\left(\frac{p_{i}}{q_{i}}\right)_{i}$ be the corresponding sequence of convergents. When $x$ is a non-integer rational number so that the expansion terminates after a nonzero number of steps, then we always assume that the expansion is of the second form, that is, the last coefficient is assumed to be $1$.

It can be shown that the sequence $$\left(\frac{p_{2i}}{q_{2i}}\right)_{i}$$ of even convergents strictly increases to $x$ from below, while the sequence $$\left(\frac{p_{2i+1}}{q_{2i+1}}\right)_{i}$$ of odd convergents strictly decreases to $x$ from above. In other words, the convergence of convergents happens in a "zig-zag" manner, alternating between below and above of $x$.

As the approximation errors of the even convergents decrease to zero, any sufficiently good rational approximation $\frac{p}{q}\leq x$ from below must lie in between some $\frac{p_{2i}}{q_{2i}}$ and $\frac{p_{2i+2}}{q_{2i+2}}$. Similarly, any sufficiently good rational approximation $\frac{p}{q}\geq x$ from above must lie in between some $\frac{p_{2i+1}}{q_{2i+1}}$ and $\frac{p_{2i-1}}{q_{2i-1}}$.

Based on these facts, one can show the following: a rational number is a best rational approximation from below of $x$ if and only if it can be written as

$$
  \frac{p_{2i} + sp_{2i+1}}{q_{2i} + sq_{2i+1}}
$$

for some integer $0\leq s\leq a_{2i+2}$, and it is a best approximation from above of $x$ if and only if it can be written as

$$
  \frac{p_{2i-1} + sp_{2i}}{q_{2i-1} + sq_{2i}}
$$

for some integer $0\leq s\leq a_{2i+1}$.

In general, a rational number of the form

$$
  \frac{p_{i-1} + sp_{i}}{q_{i-1} + sq_{i}}
$$

for an integer $0\leq s\leq a_{i+1}$ is called a *semiconvergent*. So in other words, semiconvergents are precisely the best rational approximations from below/above of $x$.

It can be shown that the sequence

$$
  \left(\frac{p_{2i} + sp_{2i+1}}{q_{2i} + sq_{2i+1}}\right)_{s=0}^{a_{2i+2}}
$$

strictly monotonically increases from $\frac{p_{2i}}{q_{2i}}$ to $\frac{p_{2i+2}}{q_{2i+2}}$, and similarly the sequence

$$
  \left(\frac{p_{2i-1} + sp_{2i}}{q_{2i-1} + sq_{2i}}\right)_{s=0}^{a_{2i+1}}
$$

strictly monotonically decreases from $\frac{p_{2i-1}}{q_{2i-1}}$ to $\frac{p_{2i+1}}{q_{2i+1}}$. Therefore, as $s$ increases, we get successively better approximations of $x$.

For the case of $x=\log_{10}2$, here are the lists of all best rational approximations from below:

$$
  \mathbf{0} < \frac{1}{4}< \frac{2}{7} < \mathbf{\frac{3}{10}}
  < \frac{31}{103} < \mathbf{\frac{59}{196}} < \frac{205}{681}
  < \frac{351}{1166} < \frac{497}{1651} < \mathbf{\frac{643}{2136}}
  <\ \cdots\ <\log_{10}2,
$$

and from above:

$$
  1 > \frac{1}{2} > \mathbf{\frac{1}{3}} > \frac{4}{13} > \frac{7}{23}
  > \frac{10}{33} > \frac{13}{43} > \frac{16}{53} > \frac{19}{63}
  > \frac{22}{73} > \frac{25}{83} > \mathbf{\frac{28}{93}}
  >\ \cdots\ >\log_{10}2
$$

with convergents highlighted in bold.

Clearly, if $\frac{p}{q}$ is a best rational approximation of $x$ from below, then we must have $p=\lfloor qx \rfloor$. Indeed, if $p$ is strictly less than $\lfloor qx \rfloor$, then $\frac{p+1}{q}$ must be a strictly better approximation of $x$ which is still below $x$, and if $p$ is strictly greater than $\lfloor qx \rfloor$, then $\frac{p}{q}$ must be strictly greater than $x$.

Similarly, if $\frac{p}{q}$ is a best rational approximation of $x$ from above, then we must have $p=\lceil qx \rceil$.

# Application into computation of $\lfloor nx \rfloor$

Alright, now let's get back to where we started: when do we have the eqaulity

$$\tag{$*$}
  \left\lfloor \frac{nm}{2^{k}} \right\rfloor = \lfloor nx \rfloor
$$

for all $\vert n\vert \leq n_{\max}$? First, note that this equality can be equivalently written as the inequality

$$
  \lfloor nx \rfloor \leq \frac{nm}{2^{k}} < \lfloor nx \rfloor + 1.
$$

## The case $n>0$

We will first consider the case $n>0$. In this case, the inequality above can be rewritten as

$$
  \frac{\lfloor nx \rfloor}{n}
  \leq \frac{m}{2^{k}}
  < \frac{\lfloor nx \rfloor + 1}{n}.
$$

Obviously, thus $(*)$ holds for all $n=1,\ \cdots\ ,n_{\max}$ if and only if

$$\tag{$**$}
  \max_{n=1,\ \cdots\ ,n_{\max}}\frac{\lfloor nx \rfloor}{n}
  \leq \frac{m}{2^{k}}
  <\min_{n=1,\ \cdots\ ,n_{\max}}\frac{\lfloor nx \rfloor + 1}{n}.
$$

Now, note that $\frac{\lfloor nx \rfloor}{n}$ and $\frac{\lfloor nx \rfloor + 1}{n}$ are rational approximations of $x$, where the former is smaller than or equal to $x$ and the latter is strictly greater than $x$.

Therefore, the left-hand side of $$(**)$$ is nothing but $$\frac{p_{*}}{q_{*}}$$, where $$\frac{p_{*}}{q_{*}}$$ is the best rational approximation from below of $x$ with largest $$q_{*}\leq n_{\max}$$.

Similarly, the right-hand side of $$(**)$$ is nothing but $$\frac{p^{*}}{q^{*}}$$, where $$\frac{p^{*}}{q^{*}}$$ is the best rational approximation from above of $x$ with the largest $$q^{*}\leq n_{\max}$$. **Except when $$x=\frac{p^{*}}{q^{*}}$$** (which means that $x$ is rational and its denominator is at most $n_{\max}$), in which case the situation is a bit dirtier and is analyzed as follows.

Note that the case $$x=\frac{p^{*}}{q^{*}}$$ is the case considered in [the classical paper by Granlund-Montgomery](https://gmplib.org/~tege/divcnst-pldi94.pdf) (Theorem 4.2), but here we will do a sharper analysis that gives us a condition that is not only sufficient but also necessary to have $$(*)$$ for all $n=1,\ \cdots\ ,n_{\max}$. Since the case we are considering here is when $x$ is rational and its denominator is at most $n_{\max}$, which means $$\frac{p^{*}}{q^{*}}=\frac{p_{*}}{q_{*}}=x$$, so let us drop those stars from our notation and just write $x=\frac{p}{q}$ for brevity. Then we want to find the minimizer of

$$
  \frac{\left\lfloor np/q \right\rfloor + 1}{n}
$$

for $n=1,\ \cdots\ ,n_{\max}$. Let $r$ be the remainder of $np$ divided by $q$, then the above can be written as

$$
  \frac{(np-r)/q + 1}{n} = \frac{p}{q} + \frac{q - r}{qn},
$$

so the task is to find the minimizer of $\frac{q-r}{n}$. One can expect that the minimum will be achieved when $r=q-1$, so let $u$ be the largest integer such that $u\leq n_{\max}$ and $up\equiv -1\ (\operatorname{mod}\,q)$. We claim that $u$ is an optimizer of $\frac{q-r}{n}$. Indeed, by definition $u$, we must have $n_{\max}<u+q$. Note that for any $n$,

$$
  \frac{q-r}{n} = \frac{1}{u}\cdot \frac{(q-r)u}{n},
$$

so it suffices to show that $n$ must be at most $(q-r)u$, so suppose $n>(q-r)u$ on the contrary. Since $up\equiv -1\ (\operatorname{mod}\,q)$, we have $(q-r)up\equiv r\equiv np\ (\operatorname{mod}\,q)$. As we have $np>(q-r)up$, we must have

$$
  np = (q-r)up + eq
$$

for some $e\geq 1$. However, since $np$ and $(q-r)up$ are both multiples of $p$, $eq$ must be a multiple of $p$ as well, but since $p$ and $q$ are coprime, it follows that $e$ is a multiple of $p$. Therefore,

$$
  n = (q-r)u + \frac{eq}{p} \geq (q-r)u + q \geq u + q,
$$

but this contradicts to $n_{\max} < u + q$, thus proving the claim.

As a result, we obtain

$$
  \min_{n=1,\ \cdots\ ,n_{\max}}\frac{\lfloor nx \rfloor + 1}{n}
  = x + \frac{1}{qu}
$$

where $u$ is the largest integer such that $u\leq n_{\max}$ and $up\equiv -1\ (\operatorname{mod}\,q)$.

In summary, we get the following conclusions:
- If $x$ is irrational or rational with denominator strictly greater than $n_{\max}$, then $(*)$ holds for all $n=1,\ \cdots\ ,n_{\max}$ if and only if

  $$
    \frac{2^{k}p_{*}}{q_{*}}
    \leq m < \frac{2^{k}p^{*}}{q^{*}}.
  $$

- If $x=\frac{p}{q}$ is rational with $q\leq n_{\max}$, then $(*)$ holds for all $n=1,\ \cdots\ ,n_{\max}$ if and only if

  $$
    \frac{2^{k}p}{q}
    \leq m
    < \frac{2^{k}p}{q} + \frac{2^{k}}{qu}
  $$

  where $u$ is the largest integer such that $u\leq n_{\max}$ and $up\equiv -1\ (\operatorname{mod}\,q)$.

## The case $n<0$

Next, let us consider the case $n<0$. In this case, $(*)$ is equivalent to

$$
  \frac{\lceil \vert n\vert x \rceil - 1}{\vert n\vert}
  = \frac{\lfloor nx \rfloor + 1}{n}
  < \frac{m}{2^{k}}
  \leq \frac{\lfloor nx \rfloor}{n}
  = \frac{\lceil \vert n\vert x \rceil}{\vert n\vert}.
$$

Similarly to the case $n>0$, the minimum of the right-hand side is precisely $$\frac{p^{*}}{q^{*}}$$, where again $$\frac{p^{*}}{q^{*}}$$ is the best rational approximation from above of $x$ with the largest $$q^{*}\leq n_{\max}$$.

The maximum of the left-hand side is, as one can expect, a bit more involved. It is precisely $$\frac{p_{*}}{q_{*}}$$ (with the same definition above) except when $$x=\frac{p_{*}}{q_{*}}$$, in which case we do some more analysis.

As in the case $n>0$, assume $x=\frac{p}{q}$ with $q\leq n_{\max}$, and let us find the maximizer of

$$
  \frac{\left\lceil np/q \right\rceil - 1}{n}
$$

for $n=1,\ \cdots\ ,n_{\max}$. This time, let $r$ be the unique integer such that $0\leq r\leq q-1$ and

$$
  np = \left\lceil \frac{np}{q} \right\rceil q - r,
$$

then we can rewrite our objective function as

$$
  \frac{(np+r)/q - 1}{n} = \frac{p}{q} - \frac{q-r}{qn}.
$$

Therefore, a maximizer of the above is a minimizer of $\frac{q-r}{qn}$, which is, as we obtained in the previous case, the largest integer $u$ such that $u\leq n_{\max}$ and $up\equiv -1\ (\operatorname{mod}\,q)$.

Hence, we get the following conclusions:
- If $x$ is irrational or rational with denominator strictly greater than $n_{\max}$, then $(*)$ holds for all $n=-1,\ \cdots\ ,-n_{\max}$ if and only if
  
  $$
    \frac{2^{k}p_{*}}{q_{*}} < m
    \leq \frac{2^{k}p^{*}}{q^{*}}.
  $$

- If $x=\frac{p}{q}$ is rational with $q\leq n_{\max}$, then $(*)$ holds for all $n=-1,\ \cdots\ ,-n_{\max}$ if and only if
  
  $$
    \frac{2^{k}p}{q} - \frac{2^{k}}{qu}
    < m
    \leq \frac{2^{k}p}{q}
  $$

  where $u$ is the largest integer such that $u\leq n_{\max}$ and $up\equiv -1\ (\operatorname{mod}\,q)$.


## Conclusion and an example

Putting two cases together, we get the following conclusion: if $x$ is irrational or rational with denominator strictly greater than $n_{\max}$, then $(*)$ holds for all $\vert n\vert \leq n_{\max}$ if and only if

$$\tag{$\square$}
  \frac{2^{k}p_{*}}{q_{*}} < m < \frac{2^{k}p^{*}}{q^{*}}.
$$

For the case $x=\frac{p}{q}$ with $q\leq n_{\max}$, we cannot really conclude anything useful if we consider positive $n$'s and negative $n$'s altogether, because the inequality we get is

$$
  \frac{2^{k}p}{q}
  \leq m
  \leq \frac{2^{k}p}{q},
$$

which admits a solution if and only if $\frac{2^{k}p}{q}$ is an integer, which can happen only when $q$ is a power of $2$. But for that case the problem is already trivial.

The number $x$ in main motivating examples are irrational numbers, so $(\square)$ is indeed a useful conclusion. Now, let us apply it to the case $x=\log_{10}2$ to see what we can get out of it. First, we need to see how $$\frac{p_{*}}{q_{*}}$$ and $$\frac{p^{*}}{q^{*}}$$ are determined for given $n_{\max}$. This can be done in a systematic way, as shown below:

- Find the last convergent $\frac{p_{i}}{q_{i}}$ whose denominator $q_{i}$ is at most $n_{\max}$.

- If $i$ is odd, we conclude
  
  $$
    \frac{p^{*}}{q^{*}} = \frac{p_{i}}{q_{i}}
    \quad\textrm{and}\quad
    \frac{p_{*}}{q_{*}} = \frac{p_{i-1} + sp_{i}}{q_{i-1} + sq_{i}}
  $$

  where $s$ is the largest integer with $q_{i-1}+sq_{i}\leq n_{\max}$.

- If $i$ is even, we conclude
  
  $$
    \frac{p_{*}}{q_{*}} = \frac{p_{i}}{q_{i}}
    \quad\textrm{and}\quad
    \frac{p^{*}}{q^{*}} = \frac{p_{i-1} + sp_{i}}{q_{i-1} + sq_{i}}
  $$

  where $s$ is the largest integer with $q_{i-1}+sq_{i}\leq n_{\max}$.

So, for example, if $n_{\max}=1000$, then $i=5$ with $\frac{p_{i}}{q_{i}}=\frac{146}{485}$ and $\frac{p_{i-1}}{q_{i-1}}=\frac{59}{196}$ so the maximum $s$ with $q_{i-1}+sq_{i}\leq n_{\max}$ is $s=1$, concluding

$$
  \frac{p^{*}}{q^{*}} = \frac{146}{485}
  \quad\textrm{and}\quad
  \frac{p_{*}}{q_{*}} = \frac{59 + 1\cdot 146}{196 + 1\cdot 485}
  = \frac{205}{681}.
$$

Then, the minimum $k$ that allows the inequality

$$\tag{$\square$}
  \frac{2^{k}p_{*}}{q_{*}} < m < \frac{2^{k}p^{*}}{q^{*}}
$$

to have a solution is $k=18$, and in that case $m=78913$ is the unique solution. (One can verify that $78913$ is precisely $\lfloor 2^{18}\log_{10}2 \rfloor$.)

In fact, note that $$\frac{p^{*}}{q^{*}}$$ and $$\frac{p_{*}}{q_{*}}$$ stay the same for any $681\leq n_{\max}\leq 1165$; here, $1165$ is one less of $196 + 2\cdot 485=1166$, which is the moment when $$\frac{p_{*}}{q_{*}}$$ changes into the next semiconvergent

$$
  \frac{59+2\cdot 146}{196+2\cdot 485}=\frac{351}{1166}.
$$

Therefore, with the choice $k=18$ and $m=78913$, we should have

$$
  \lfloor nx \rfloor = \left\lfloor \frac{78913\cdot n}{2^{18}} \right\rfloor
$$

for all $\vert n\vert \leq 1165$. In fact, even with $$\frac{p_{*}}{q_{*}}=\frac{351}{1166}$$ we still have $(\square)$, so the above should hold until the next moment when $$\frac{p_{*}}{q_{*}}$$ changes, which is when $n_{\max} = 196 + 3\cdot 485=1651$. If $n_{\max}=1651$, then

$$
  \frac{p_{*}}{q_{*}} = \frac{59 + 3\cdot 146}{196 + 3\cdot 485} = \frac{497}{1651},
$$

and in this case now the left-hand side of $(\square)$ is violated. Thus,

$$
  \lfloor nx \rfloor = \left\lfloor \frac{78913\cdot n}{2^{18}} \right\rfloor
$$

holds precisely up to $\vert n\vert\leq 1650$ and the first counterexample is $n=\pm 1651$.

In a similar manner, one can see that $$\frac{p_{*}}{q_{*}}=\frac{497}{1651}$$ holds up to $n_{\max} = 196 + 4\cdot 485 - 1 = 2135$, and the minimum $k$ allowing $(\square)$ to have a solution is $k=20$, with $m=315653$ as the unique solution. (One can also verify that $315653$ is precisely $\lceil 2^{20}\log_{10}2 \rceil$, so this time it is not the truncated binary expansion of $\log_{10}2$, rather that plus $1$.) Hence,

$$
  \lfloor nx \rfloor = \left\lfloor \frac{315653\cdot n}{2^{20}} \right\rfloor
$$

must hold at least up to $\vert n\vert\leq 2135$. When $n_{\max}=2136$, $$\frac{p_{*}}{q_{*}}$$ changes into $\frac{643}{2136}$, but $(\square)$ is still true with the same choice of $k$ and $m$, so the above formula must be valid at least for $\vert n\vert\leq 2620$; here $2621$ is the moment when $$\frac{p^{*}}{q^{*}}$$ changes from $\frac{146}{485}$ into $\frac{789}{2621}$. If $n_{\max}=2621$ so $$\frac{p^{*}}{q^{*}}$$ changes into $\frac{789}{2621}$, then the right-hand side of the inequality $(\square)$ is now violated, so $\vert n\vert\leq 2620$ is indeed the optimal range and $n=\pm 2621$ is the first counterexample.

In general, the transition can only occur at the denominators of semiconvergents. With this method, we can figure out what is the minimum $k$ that allows the computation up to the chosen transition point and what $m$ should be used for that choice of minimum $k$. We want $m$ to be as small as possible, so $$m=\left\lceil\frac{2^{k}p_{*}}{q_{*}} \right\rceil$$ is the best choice. This $m$ will be probably either floor or ceil of $2^{k}x$, but we cannot determine which one is better by simply looking at the binary expansion of $x$. This indicates the flaw of the naive method I tried before.


# Another application: minmax Euclid algorithm

At the very heart of [Ryū](https://dl.acm.org/doi/pdf/10.1145/3192366.3192369), [Ryū-printf](https://dl.acm.org/doi/pdf/10.1145/3360595), [Grisu-Exact](https://github.com/jk-jeon/Grisu-Exact) and [Dragonbox](https://github.com/jk-jeon/dragonbox) (which are all float-to-string conversion algorithms), there is so-called *minmax Euclid algorithm*.

Minmax Euclid algorithm is invented by Ulf Adams who is the author of Ryū and Ryū-printf mentioned above, and it appears on his paper on Ryū. It is at the core of showing the sufficiency of the number of bits of each item in the precomputed lookup table. What it does is this: given positive integers $a,b$ and $N$, compute the minimum and maximum of $ag\,\operatorname{mod}\,b$ where $g$ is any integer ranging from $1$ to $N$. It sounds simple, but naive exhaustive search doesn't scale if $a,b,N$ are very large numbers. Indeed, in its application into float-to-string conversion algorithms, $a,b$ are extremely large integers and $N$ can be as large as $2^{54}$. For that wide range of $g$, it is simply impossible to run an exhaustive search.

A rough idea of minmax Euclid algorithm is that the coefficients appearing in the Euclid algorithm for computing GCD of $a$ and $b$ tell us what the minimum and the maximum are. If you try to write down and see how $ag\,\operatorname{mod}\,b$ varies as $g$ increases, you will find out why this is the case. (Precise mathematical proof can be cumbersome, though.) Formalizing this idea will lead to the minmax Euclid algorithm.

In fact, the exact minmax Euclid algorithm described in the [Ryū paper](https://dl.acm.org/doi/pdf/10.1145/3192366.3192369) is not entirely correct because it can produce wrong outputs for some inputs. Thus, I had to give some more thought on it after I realized the flaw in 2018 when I was trying to apply the algorithm to Grisu-Exact. Eventually, I came up with an improved and corrected version of it with a hopefully correct proof, which I wrote on the [paper on Grisu-Exact](https://github.com/jk-jeon/Grisu-Exact/blob/master/other_files/Grisu-Exact.pdf) (Figure 3 and Theorem 4.3).

But more than 2 years after writing the long-winded and messy proof of the algorithm, I finally realized that in fact the algorithm is just an easy application of continued fractions. Well, I did not know anything about continued fractions back then, unfortunately!

The story is as follows. Obtaining the minimum and the maximum of $ag\,\operatorname{mod}\,b$ is equivalent to obtaining the minimum and the maximum of

$$
  \frac{ag\,\operatorname{mod}\,b}{b}
  = \frac{ag}{b} - \left\lfloor\frac{ag}{b}\right\rfloor
  = g\left(\frac{a}{b} - \frac{\lfloor ag/b \rfloor}{g}\right)
  = 1 - g\left(\frac{\lfloor ag/b \rfloor + 1}{g} - \frac{a}{b}\right),
$$

which reminds us of approximating $\frac{a}{b}$ by $\frac{\lfloor ag/b \rfloor}{g}$ or by $\frac{\lfloor ag/b \rfloor + 1}{g}$, except that what we are minimizing is not the error itself, rather the error multiplied by $g$. Hence, we define the following concepts.

We say a rational number $\frac{p}{q}$ is a *best rational approximation from below in the strong sense* of $x$ if $\frac{p}{q}\leq x$ and for any rational number $\frac{a}{b}$ with $\frac{a}{b}\leq x$ and $b\leq q$, we have

$$
  \vert qx - p\vert \leq \vert bx - a\vert.
$$

Similarly, we say a rational number $\frac{p}{q}$ is a *best rational approximation from above in the strong sense* of $x$ if $\frac{p}{q}\geq x$ and for any rational number $\frac{a}{b}$ with $\frac{a}{b}\geq x$ and $b\leq q$, we have

$$
  \vert qx - p\vert \leq \vert bx - a\vert.
$$

Let us call best rational approximations from below/above defined before as "in the weak sense" to distinguish from the new definitions.

The reason why these are called "in the strong sense" is obvious; if we have

$$
  \vert qx - p\vert \leq \vert bx - a\vert,
$$

then dividing both sides by $q$ gives

$$
  \left\vert x - \frac{p}{q}\right\vert
  \leq \frac{b}{q}\left\vert x - \frac{a}{b}\right\vert,
$$

so with $b\leq q$ the right-hand side is at most $\left\vert x-\frac{a}{b}\right\vert$, so this implies that $\frac{p}{q}$ is a better approximation than $\frac{a}{b}$.

It is a well-known fact that, if we remove the directional conditions $\frac{a}{b}\leq x$ and $\frac{a}{b}\geq x$, so asking for *best rational approximations (in both directions) in the strong sense*, then these are precisely the convergents. However, it turns out that with the directional conditions, the best rational approximations from below/above in the weak sense or in the strong sense are just same. Hence, the best rational approximations from below/above in the strong sense are also just precisely the semiconvergents. This fact is probably well-known as well, but I do not know of any reference at this point other than my own proof of it (which I do not write here). I guess this fact on semiconvergents is probably one of the necessary pieces for proving the other fact that convergents are precisely the best rational approximations in the strong sense, but I haven't checked any proof of it so I do not know.

Anyway, because of this, the problem of finding the minimum and the maximum of $ag\,\operatorname{mod}\,b$ reduces to the problem of finding the semiconvergents that are below and above $\frac{a}{b}$ with the largest denominators among all semiconvergents of denominators at most $N$. This is essentially what the improved minmax Euclid algorithm is doing as described in my [paper](https://github.com/jk-jeon/Grisu-Exact/blob/master/other_files/Grisu-Exact.pdf). I will not discuss further details here.


# Computation of $\lfloor nx - y \rfloor$

When developing [Dragonbox](https://github.com/jk-jeon/dragonbox), I also had to come up with a method of computing

$$
  \left\lfloor n\log_{10}2 - \log_{10}\frac{4}{3} \right\rfloor.
$$

So I did the same thing, I approximated both $\log_{10}2$ and $\log_{10}\frac{4}{3}$ by their respective truncated binary expansions, and computed something like

$$
  \left\lfloor \frac{nm - s}{2^{k}} \right\rfloor,
$$

where $m,s$ are both positive integers, and manually found out the range of $n$ making the above formula valid.

More generally, we can think of computing $\lfloor nx - y \rfloor$ for fixed positive real numbers $x,y$. Again, signs of $x,y$ are not terribly important, but let us just assume $x,y>0$ to make our life a bit easier. Also, let us further assume $0<y<1$ because taking the integer part of $y$ out of the floor function is not a big deal. So, how far we can go with a similar analysis in this case? Right now, it seems to me that this problem is far subtler than the case $y=0$ we have done before, but anyway here I am going to give a brief overview of what I got.

The rough idea is as follows. If we approximate $x$ by a rational number $\frac{p}{q}$ sufficiently well, then each $nx$ must be very close to $\frac{np}{q}$. Note that

$$
  \lfloor nx - y \rfloor = \begin{cases}
    \lfloor nx \rfloor & \textrm{if $nx - \lfloor nx \rfloor \geq y$},\\
    \lfloor nx \rfloor - 1 & \textrm{if $nx - \lfloor nx \rfloor < y$},
  \end{cases}
$$

hence, what matters here is whether the fractional part of $nx$ is above or below $y$. Note that the fractional part of $nx$ must be approximately equal to $\frac{np\,\operatorname{mod}\,q}{q}$. Thus, find the unique integer $0\leq u < q-1$ such that $\frac{u}{q}\leq y < \frac{u+1}{q}$, then probably we can compare the fractional part of $nx$ with $y$ by just comparing $(np\,\operatorname{mod}\,q)$ with $u$. This is indeed the case if $y$ is sufficiently far from both $\frac{u}{q}$ and $\frac{u+1}{q}$, specifically, more than the error of $\frac{np}{q}$ from $nx$.

So, in some sense the situation is better if $x$ and $y$ are "far apart" in the sense that the denominators showing up in approximating rationals of $x$ and $y$ are very  different, and the situation is worse if there are a lot of common divisors between those denominators. Maybe someone better than me at number theory can formalize this into a precise language?

Anyway, with a high probability the distance from $y$ to $\frac{u}{q}$ and $\frac{u+1}{q}$ will be of $O\left(\frac{1}{q}\right)$, but it is well-known that the distance from $x$ to $\frac{p}{q}$ can be made to be of $O\left(\frac{1}{q^{2}}\right)$, which will allow the equality

$$
  \lfloor nx - y \rfloor = \left\lfloor \frac{np - u}{q} \right\rfloor
$$

to hold for $O(q)$-many $n$'s. After getting this equality, the next step is to convert it into

$$
  \left\lfloor \frac{np - u}{q} \right\rfloor =
  \left\lfloor \frac{nm - s}{2^{k}} \right\rfloor
$$

using a Grandlund-Montgomery style analysis we did for $\lfloor nx \rfloor$ with rational $x$.

Unfortunately, the bound I got by applying this analysis to the case $x=\log_{10}2$ and $y=\log_{10}\frac{4}{3}$ was not that great, particularly because $y$ is too close to $\frac{u+1}{q}$ for the choice $\frac{p}{q}=\frac{643}{2136}$, which is otherwise a very efficient and effective approximation of $x$. Well, but I might be overlooking some things at this point, so probably I have to give some more tries on this later.

**(Edit on 02-10-2022)** I included a better analysis of this in my new paper on [Dragonbox](https://github.com/jk-jeon/dragonbox/blob/master/other_files/Dragonbox.pdf), Section 4.4. But still I consider it incomplete, because it seems that the actual range of $n$ is much wider than what the analysis method described in the paper expects.


# Acknolwedgements

Thanks Seung uk Jang for teaching me many things about continued fractions (including useful references) and other  discussions relevant to this post.
