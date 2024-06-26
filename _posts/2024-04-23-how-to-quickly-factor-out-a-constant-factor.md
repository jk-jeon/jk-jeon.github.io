---
title: "How to quickly factor out a constant factor from integers"
date: 2024-04-23
permalink: /posts/2024/04/how-to-quickly-factor-out-a-constant-factor/
tags:
  - programming
  - C/C++
---

This post revisits the topic of integer division, building upon the discussion in [the previous post](https://jk-jeon.github.io/posts/2023/08/optimal-bounds-integer-division/). Specifically, I'll delve into removing trailing zeros in the decimal representation of an input integer, or more broadly, factoring out the highest power of a given constant that divides the input. This exploration stems from the problem of converting floating-point numbers into strings, where certain contemporary algorithms, such as [Schubfach](https://drive.google.com/file/d/1luHhyQF9zKlM8yJ1nebU0OgVYhfC6CBN/edit) and [Dragonbox](https://github.com/jk-jeon/dragonbox), may yield outputs containing trailing zeros.

Here is the precise statement of our problem:

> Given positive integer constants $n_{\max}$ and $q\leq n_{\max}$, how to find the largest integer $k$ such that $q^{k}$ divides $n$, together with the corresponding $\frac{n}{q^{k}}$, for any $n=1,\ \cdots\ ,n_{\max}$?

As one can expect, this more or less boils down to an efficient divisibility test algorithm. However, merely testing for divisibility is not enough, and we need to actually divide the input by the divisor when we know the input is indeed divisible.

# Naïve algorithm

Here is the most straightforward implementation I can think of:

```cpp
std::size_t s = 0;
while (true) {
    auto const r = n % q;
    if (r == 0) {
        n /= q;
        s += 1;
    }
    else {
        break;
    }
}
return {n, s};
```

Of course, this works (as long as $n\neq 0$ which we do assume throughout), but obviously our objective is to explore avenues for greater performance. Here, we are assuming that the divisor $q$ is a given constant, so any sane modern compiler knows that the dreaded generic integer division is not necessary. Rather, they would replace the division into a multiply-and-shift, or some slight variation of it, as explained in [the previous post](https://jk-jeon.github.io/posts/2023/08/optimal-bounds-integer-division/). That is great, but note that we need to compute *both* the quotient and the remainder. As far as I know, there is no known algorithm capable of computing both quotient and remainder with *only one* multiplication, which means that the above code will perform *two* multiplications per iteration. However, if we consider cases where the input is not divisible by the divisor, we realize that we don't actually require the precise values of the quotient or the remainder. Our sole concern is whether the remainder is *zero*, and *only if* that is the case, we do want to know the quotient of the division. Therefore, it's conceivable that we could accomplish this with just one multiplication per iteration, which presumably will improve the performance.

Actually, [the classical paper](https://gmplib.org/~tege/divcnst-pldi94.pdf) by Granlund-Montgomery already presented such an algorithm, which is the topic of the next section.

# Granlund-Montgomery modular inverse algorithm

First, let us assume that $q$ is an odd number for a while. Assume further that our input is of $b$-bit unsigned integer type, e.g. of type `std::uint32_t`, with $b=32$. Hence, we are assuming that $n_{\max}$ is at most $2^{b}-1$. Now, since $q$ is *coprime* to $2^{b}$, there uniquely exists the *modular inverse* of $q$ with respect to $2^{b}$. Let us call it $m$, which in general can be found using the extended Euclid's algorithm. But how is it useful to us?

The key observation here is that, on the group $\mathbb{Z}/2^{b}$ of integer residue classes modulo $2^{b}$, the multiplication by any integer coprime to $2^{b}$ is an automorphism, i.e., a *bijective* group homomorphism onto itself. In particular, multiplication by such an integer induces a *bijection* from the set $$\left\{0,1,\ \cdots\ ,2^{b}-1\right\}$$ onto itself.

Now, what does this bijection, defined in particular by the modular inverse $m$ (which must be coprime to $2^{b}$), do to multiples of $q$, i.e., $0, q,\ \cdots\ ,\left\lfloor\frac{2^{b} - 1}{q}\right\rfloor q$? Note that for any integer $a$,

$$
  (aq)m \equiv a(qm) \equiv a\ (\operatorname{mod}\ 2^{b}),
$$

thus, the bijection $$\left\{0,1,\ \cdots\ ,2^{b}-1\right\}\to\left\{0,1,\ \cdots\ ,2^{b}-1\right\}$$ defined by $m$ maps $aq$ into $a$. Therefore, *anything that gets mapped into $$\left\{0,1,\ \cdots\ ,\left\lfloor\frac{2^{b}-1}{q}\right\rfloor\right\}$$ must be a multiple of $q$*, because the map is a bijection, and *vice versa*.

Furthermore, the image of this bijection, in the case when the input $n$ was actually a multiple of $q$, is precisely $n/q$. Since multiplication modulo $2^{b}$ is what C/C++ abstract machines are supposed to do whenever they see an integer multiplication, we find that the following code does the job we want, but with *only one* multiplication per an iteration:

```cpp
// m and threshold_value are precalculated from q.
// m is the modular inverse of q, and
// threshold_value is (0xff....ff / q) + 1.
std::size_t s = 0;
while (true) {
    auto const r = n * m;
    if (r < threshold_value) {
        n = r;
        s += 1;
    }
    else {
        break;
    }
}
return {n, s};
```

All is good so far, but recall that this works *only when $q$ is odd*. Unfortunately, our main application, the trailing zero removal, requires $q$ to be $10$, which is even.

Actually, [the aforementioned paper](https://gmplib.org/~tege/divcnst-pldi94.pdf) by Granlund-Montgomery proposes a clever solution for the case of possibly non-odd divisors, based on *bit-rotation*. Let us write $q = 2^{t}q_{0}$ for an integer $t$ and a positive odd number $q_{0}$. Obviously, there is no modular inverse of $q$ with respect to $2^{b}$, but we can try the next best thing, that is, the modular inverse of $q_{0}$ with respect to $2^{b-t}$. Let us call it $m$.

Now, for given $n=0,1,\ \cdots\ ,n_{\max} \leq 2^{b}-1$, let us consider the multiplication $nm$ modulo $2^{b}$ and then perform bit-rotate-to-right on it by $t$-bits. Let us call the result $r$. Clearly, the highest $t$-bits of $r$ is all-zero if and only if the lowest $t$-bits of $(nm\ \operatorname{mod}\ 2^{b})$ is all-zero, and since $m$ must be an odd number, this is the case if and only if the lowest $t$-bits of $n$ is all-zero, that is, $n$ is a multiple of $2^{t}$.

We claim that $n$ is a multiple of $q$ if and only if $r \leq \left\lfloor\frac{2^{b}-1}{q}\right\rfloor = \left\lfloor\frac{2^{b-t}-1}{q_{0}}\right\rfloor$ holds. By the argument from the previous paragraph, $r$ must be strictly larger than $\left\lfloor\frac{2^{b-t}-1}{q_{0}}\right\rfloor$ if $n$ is not a multiple of $2^{t}$, since $\left\lfloor\frac{2^{b-t}-1}{q_{0}}\right\rfloor$ is at most of $(b-t)$-bits. Thus, we only need to care about the case when $n$ is a multiple of $2^{t}$. In that case, write $n=2^{t}n_{0}$, then $r$ is in fact equal to

$$
  \frac{(nm\ \operatorname{mod}\ 2^{b})}{2^{t}}
  = \frac{nm - \lfloor nm/2^{b}\rfloor 2^{b}}{2^{t}}
  = n_{0}m - \left\lfloor \frac{n_{0}m}{2^{b-t}}\right\rfloor 2^{b-t}
  = (n_{0}m\ \operatorname{mod}\ 2^{b-t}),
$$

since the lowest $t$-bits of $(nm\ \operatorname{mod}\ 2^{b})$ are all zero. By the discussion of the case when $q$ is odd, we know that $n_{0}$ is a multiple of $q_{0}$ if and only if $(n_{0}m\ \operatorname{mod}\ 2^{b-t})$ is at most $\left\lfloor\frac{2^{b-t}-1}{q_{0}}\right\rfloor$, thus finishing the proof of the claim.

Of course, when $n$ was really a multiple of $q$, the resulting $r$ is precisely $n/q$, by the same reason as in the case of odd $q$. Consequently, we obtain the following version which works for all $q$:

```cpp
// t, m and threshold_value are precalculated from q.
// t is the number of trailing zero bits of q,
// m is the modular inverse of (q >> t) with respect to 2^(b-t), and
// threshold_value is (0xff....ff / q) + 1.
std::size_t s = 0;
while (true) {
    auto const r = std::rotr(n * m, t); // C++20 <bit>
    if (r < threshold_value) {
        n = r;
        s += 1;
    }
    else {
        break;
    }
}
return {n, s};
```

# Lemire's algorithm

In a [paper published in 2019](https://doi.org/10.1002/spe.2689), Lemire et al proposed an alternative way of checking divisibility which has some advantages over Granlund-Montgomery algorithm. The theorem behind this algorithm was not optimal when first presented, but later they showed the optimal result in [another paper](https://doi.org/10.1016/j.heliyon.2021.e07442). Here, I present a more general result which contains the result by Lemire et al as a special case. This is one of the theorems I proved in my [paper on Dragonbox](https://github.com/jk-jeon/dragonbox/tree/master/other_files/Dragonbox.pdf) (Theorem 4.6), which I copied below:

><b id="integer-check">Theorem 1</b>
>Let $x,\xi$ be real numbers and $n_{\max}$ be a positive integer
>such that
>
>$$
>    \left\lfloor nx \right\rfloor = \left\lfloor n\xi \right\rfloor
>$$
>
>holds for all $n=1,\ \cdots\ ,n_{\max}$. Then, for a positive real number $\eta$,
>we have the followings.
>
>1. If $x=\frac{p}{q}$ is a rational number with $2\leq q\leq n_{\max}$,
>    then we have
>    
>    $$
>        \left\{n\in \{1,\ \cdots\ ,n_{\max}\}\colon nx\in \mathbb{Z}\right\}
>        = \left\{n\in \{1,\ \cdots\ ,n_{\max}\}\colon n\xi - \left\lfloor n\xi\right\rfloor < \eta\right\}
>    $$
>    
>    if and only if
>    
>    $$
>        \left\lfloor \frac{n_{\max}}{q} \right\rfloor q(\xi - x)
>        < \eta \leq u(\xi - x) + \frac{1}{q}
>    $$
>    
>    holds, where $u$ is the smallest positive integer such that
>    $up\equiv 1\ (\operatorname{mod}\ q)$.
>
>2. If $x$ is either an irrational number or a rational number
>    with the denominator strictly greater than $n_{\max}$, then we have
>    
>    $$
>        \left\{n\in \{1,\ \cdots\ ,n_{\max}\} \colon nx\in \mathbb{Z}\right\} = \emptyset
>        = \left\{n\in \{1,\ \cdots\ ,n_{\max}\} \colon n\xi - \left\lfloor n\xi\right\rfloor < \eta\right\}
>    $$
>    
>    if and only if
>    
>    $$
>        \eta \leq q_{*}\xi - p_{*}
>    $$
>    
>    holds, where $$\frac{p_{*}}{q_{*}}$$ is the
>    best rational approximations of $x$ from below
>    with the largest denominator $$q_{*}\leq n_{\max}$$.

The theorem above shows a necessary and sufficient condition for the product $nx$ to be an integer in terms of a good enough approximation $\xi$ of $x$. The second item is irrelevant in this post, so we only focus on the first item. The point here is that we are going to let $x = \frac{1}{q}$ so that $nx$ is an integer if and only if $n$ is a multiple of $q$. In that case, it is shown in the [paper](https://github.com/jk-jeon/dragonbox/tree/master/other_files/Dragonbox.pdf) (*Remark 2* after Theorem 4.6) that we can always take $\xi = \eta$ as long as $\xi$ satisfies the condition

$$\label{eq:precondition}
  \left\lfloor \frac{n}{q} \right\rfloor = \left\lfloor n\xi \right\rfloor
$$

for all $n=1,\ \cdots\ ,n_{\max}$, so that $n$ is a multiple of $q$ if and only if

$$
  n\xi - \left\lfloor n\xi \right\rfloor < \xi.
$$

Also, another theorem from [the paper](https://github.com/jk-jeon/dragonbox/tree/master/other_files/Dragonbox.pdf) (Theorem 4.2) shows that a necessary and sufficient condition for having the above is that

$$
  \frac{1}{q} \leq \xi < \frac{1}{q} + \frac{1}{vq}
$$

holds, where $v$ is the greatest integer with $v\equiv -1\ (\operatorname{mod}\ q)$ and $v\leq n_{\max}$, i.e.,

$$
  v=\left\lfloor \frac{n_{\max} + 1}{q} \right\rfloor q - 1.
$$

In practice, we take $\xi = \frac{m}{2^{b}}$ for some positive integers $m,b$ so that

$$
  n\xi - \lfloor n\xi\rfloor < \xi
$$

holds if and only if

$$
  (nm\ \operatorname{mod}\ 2^{b}) < m.
$$

Hence, the above is a necessary and sufficient condition for $n$ to be divisible by $q$, as long as the inequality

$$
  \frac{1}{q} \leq \frac{m}{2^{b}} < \frac{1}{q} + \frac{1}{vq}
  = \frac{\left\lfloor (n_{\max}+1)/q\right\rfloor}
  {\left\lfloor (n_{\max}+1)/q\right\rfloor q - 1}
$$

holds. Furthermore, since $\left\lfloor nx\right\rfloor = \left\lfloor n\xi \right\rfloor$ holds, the quotient $\left\lfloor \frac{n}{q}\right\rfloor$ can be also computed as

$$
  \lfloor n\xi\rfloor = \left\lfloor \frac{nm}{2^{b}} \right\rfloor.
$$

In other words, we multiply $m$ to $n$, then the lowest $b$-bits can be used for inspecting divisibility, while the remaining upper bits constitutes the quotient.

Note that this is always the case, regardless of whether $n$ is divisible by $q$ or not, so this algorithm actually does more than what we are asking for. Furthermore, there is only *one* magic number, $m$, while Granlund-Montgomery requires two. This may have some positive impact on the code size, instruction decoding overhead, register usage overhead, etc..

However, one should also note that the cost of these "extra features" is that we must perform a "widening multiplication", that is, if $n_{\max}$ and $m$ are of $b$-bits, then we need all of $2b$-bits of the multiplication of $n$ and $m$. It is also worth mentioning that the magic number $m$ might actually require more than $b$-bits. For details, please refer to [the previous post]((https://jk-jeon.github.io/posts/2023/08/optimal-bounds-integer-division/)), or [this paper](https://doi.org/10.1016/j.heliyon.2021.e07442) by Lemire et al.

In any case, the resulting algorithm would be like the following:

```cpp
// m is precalculated from q.
std::size_t s = 0;
while (true) {
    auto const r = widening_mul(n, m);
    if (low_bits(r) < m) {
        n = high_bits(r);
        s += 1;
    }
    else {
        break;
    }
}
return {n, s};
```

# Generalized modular inverse algorithm

Staring at [**Theorem 1**](#integer-check), one can ask: *if all we care about is **divisibility**, not the quotient, then do we really need to take $x = \frac{1}{q}$?* That is, as long as $p$ is any integer coprime to $q$, asking if $n/q$ is an integer is exactly the same question as asking if $\frac{np}{q}$ is an integer. In fact, this observation leads to a derivation of the modular inverse divisibility check algorithm by Granlund-Montgomery explained [above](#granlund-montgomery-modular-inverse-algorithm) for the case of odd divisor, and the Dragonbox paper already did this (*Remark 3* after Theorem 4.6). A few days ago, I realized the same argument actually applies to the case of general divisor as well, which leads to yet another divisibility check algorithm explained below, which I think probably is novel.

Suppose that $p$ is a positive integer coprime to $q$, then as pointed out, $n$ is divisible by $q$ if and only if $nx$ is an integer with $x\mathrel{\unicode{x2254}}\frac{p}{q}$. Therefore, by [**Theorem 1**](#integer-check) (together with Theorem 4.2 from [the Dragonbox paper](https://github.com/jk-jeon/dragonbox/tree/master/other_files/Dragonbox.pdf)), for any $\xi,\eta$ satisfying

$$\label{eq:condition for xi}
  \frac{p}{q} \leq \xi < \frac{p}{q} + \frac{1}{vq}
$$

and

$$\label{eq:condition for eta}
  \left\lfloor \frac{n_{\max}}{q} \right\rfloor q\left(\xi - \frac{p}{q}\right)
  < \eta \leq u\left(\xi - \frac{p}{q}\right) + \frac{1}{q},
$$

a given $n=1,\ \cdots\ ,n_{\max}$ is divisible by $q$ if and only if

$$\label{eq:integer check condition}
  n\xi - \left\lfloor n\xi \right\rfloor < \eta,
$$

where $v$ is the greatest integer satisfying $vp\equiv -1\ (\operatorname{mod}\ q)$ and $u$ is the smallest positive integer satisfying $up\equiv 1\ (\operatorname{mod}\ q)$.

Again, since our goal is to make the evaluation of the inequality $\eqref{eq:integer check condition}$ as easy as possible, we may want to take $\xi=\frac{m}{2^{b}}$ and $\eta=\frac{s}{2^{b}}$ as before, so that $\eqref{eq:integer check condition}$ becomes

$$
  (nm\ \operatorname{mod}\ 2^{b}) < s.
$$

Although that is what we will do eventually, let us consider a little bit more general case that $\xi=\frac{m}{N}$ and $\eta=\frac{s}{N}$ for any positive integer $N$, not necessarily of the form $2^{b}$, for the sake of ~~pedantry~~ clarity of what is going on. Of course, in this case $\eqref{eq:integer check condition}$ is equivalent to

$$
  (nm\ \operatorname{mod}\ N) < s.
$$

With this setting, we can also rewrite $\eqref{eq:condition for xi}$ and $\eqref{eq:condition for eta}$ as

$$\label{eq:condition for xi modified}
  0 \leq qm - Np < \frac{N}{v}
$$

and

$$\label{eq:condition for eta modified}
  \left\lfloor \frac{n_{\max}}{q} \right\rfloor \left(qm - Np\right)
  < s \leq \frac{u}{q}\left(qm - Np\right) + \frac{N}{q},
$$

respectively. Note that, both sides of the above inequality have the factor $qm - Np$, and the left-hand side multiplies it to $\left\lfloor\frac{n_{\max}}{q}\right\rfloor$ and the right-hand side multiplies it to $\frac{u}{q}$. Since $u$ is at most $q-1$, usually $\left\lfloor\frac{n_{\max}}{q}\right\rfloor$ is much larger than $\frac{u}{q}$, so it is usually the case that the inequality can hold only when the factor $qm - Np$ is small enough. So, it makes sense to actually *minimize* it. Note that we are to take $N = 2^{b}$ and $b$ is a factor defined by the application, while $m$ and $p$ are something we can choose. In such a situation, the smallest possible nonnegative value of $qm - Np$ is exactly $g\mathrel{\unicode{x2254}} \operatorname{gcd}(q,N)$, the greatest common divisor of $q$ and $N$. Recall that a general solution for the equation $qm - Np = g$ is given as

$$
  m = m_{0} + \frac{Nk}{g},\quad
  p = p_{0} + \frac{qk}{g}
$$

where $m_{0}$ is the modular inverse of $\frac{q}{g}$ with respect to $\frac{N}{g}$, $p_{0}$ is the unique integer satisfying $qm_{0} - Np_{0} = g$, and $k$ is any integer.

Now, we cannot just take any $k\in\mathbb{Z}$, because this whole argument breaks down if $p$ and $q$ are not coprime. In particular, $p_{0}$ is not guaranteed to be coprime to $q$. For example, take $N=32$ and $q=30$, then we get $g=2$, $m_{0}=15$ and $p_{0}=14$ so that $qm_{0} - Np_{0} = 2$, but $\operatorname{gcd}(p_{0},q) = 2$. Nevertheless, it is always possible to find some $k$ such that $p = p_{0} + \frac{qk}{g}$ is coprime to $q$. The proof I wrote below of this fact is due to [Seung uk Jang](https://seungukj.github.io/):

><b id="coprime-existence">Proposition 2</b>
>Let $a,b$ be any integers and $g\mathrel{\unicode{x2254}}\operatorname{gcd}(a,b)$. Then there exist integers $x,y$ such that $ax - by = g$ and $\operatorname{gcd}(a,y) = 1$. Specifically, such $x,y$ can be found as
>
>$$
>  x = x_{0} + \frac{bk}{g},\quad y = y_{0} + \frac{ak}{g},
>$$
>
>where $x_{0}$ is the modular inverse of $\frac{a}{g}$ with respect to $\frac{b}{g}$, $y_{0} = \frac{ax_{0} - g}{b}$, and $k$ is the smallest nonnegative integer satisfying
>
>$$
>  \left(\frac{a}{g}\right)k \equiv 1 - y_{0}
>  \ \left(\operatorname{mod}\ \frac{g}{\operatorname{gcd}(a/g,g)}\right).
>$$

>**Proof.** Take $x,y,k$ as given above. Note that $\frac{a}{g}$ is coprime to $g/\operatorname{gcd}(a/g,g)$, so such $k$ uniquely exists. Then we have
>
>$$
>  y\equiv y_{0} + (1-y_{0}) \equiv 1
>  \ \left(\operatorname{mod}\ \frac{g}{\operatorname{gcd}(a/g,g)}\right),
>$$
>
>so $y$ is coprime to $g/\operatorname{gcd}(a/g,g)$. On the other hand, from
>
>$$
>  \left(\frac{a}{g}\right)x_{0} - \left(\frac{b}{g}\right)y_{0} = 1,
>$$
>
>we know that $y_{0}$ is coprime to $\frac{a}{g}$, thus $y$ is coprime to $\frac{a}{g}$. This means that $y$ is coprime to $a$. Indeed, let $p$ be any prime factor of $a$, then $p$ should divide either $\frac{a}{g}$ or $g$. If $p$ divides $\frac{a}{g}$, then it cannot divide $y$ since $y$ is coprime to $\frac{a}{g}$. Also, if $p$ divides $g$ but not $\frac{a}{g}$, then it divides $g/\operatorname{gcd}(a/g,g)$ which is again coprime to $y$. $\quad\blacksquare$

For instance, applying this proposition to our example $N=32$, $q=30$ yields

$$
  m = 15 + 16k,\quad p = 14 + 15k
$$

with $k$ being the smallest nonnegative integer satisfying

$$
  15k \equiv 1 - 14\ (\operatorname{mod}\ 2),
$$

which is $k=1$. Hence, we take $m = 31$ and $p = 29$.

In fact, for the specific case of $N=2^{b}$ and $q\leq N$, we can always take either $k=0$ or $1$. Indeed, by the assumption, $g$ is a power of $2$ and $q/g$ is an odd number. Then, one of $p_{0}$ and $p_{0} + \frac{q}{g}$ is an odd number, so one of them is coprime to $g$. On the other hand, since

$$
  \left(\frac{q}{g}\right)m_{0} - \left(\frac{N}{g}\right)p_{0} = 1,
$$

we know $p_{0}$ is coprime to $\frac{q}{g}$, so both $p_{0}$ and $p_{0} + \frac{q}{g}$ are coprime to $\frac{q}{g}$. Therefore, the odd one between $p_{0}$ and $p_{0}+\frac{q}{g}$ must be coprime to $q$.

In any case, let us assume from now on that we have chosen $m$ and $p$ in a way that $\operatorname{gcd}(p,q) = 1$ and $qm - Np = g$. Then $\eqref{eq:condition for xi modified}$ and $\eqref{eq:condition for eta modified}$ can be rewritten as

$$
  v < \frac{N}{g}
$$

and

$$
  \left\lfloor \frac{n_{\max}}{q} \right\rfloor g
  < s \leq \frac{ug + N}{q},
$$

respectively. Next, we claim that $\frac{ug+N}{q}$ is an integer. Indeed, we have

$$
  (ug+N)p \equiv g + Np \equiv qm \equiv 0
  \ (\operatorname{mod}\ q),
$$

and since $p$ is coprime to $q$, $ug+N$ must be a multiple of $q$. Hence, the most sensible choice of $s$ in this case is $s = \frac{ug+N}{q}$, and let us assume that from now on. Then, the left-hand side of the inequality above can be rewritten as

$$
  \left\lfloor\frac{n_{\max}}{q}\right\rfloor q - u < \frac{N}{g}.
$$

Actually, this inequality follows from $v < \frac{N}{g}$, so is redundant, because $v$ can be written in terms of $u$ as

$$
  v = \left\lfloor \frac{n_{\max} + u}{q}\right\rfloor q - u.
$$

Plugging in the above equation into $v<N/g$, we obtain that the only condition we need to ensure is

$$
  \left\lfloor\frac{n_{\max} + u}{q}\right\rfloor < \frac{(N/g) + u}{q},
$$

or equivalently,

$$\label{eq:nmax condition}
  n_{\max} \leq \left\lfloor \frac{(N/g)+u}{q}\right\rfloor q + q - 1 - u.
$$

Therefore, as long as the above is true, we can check divisibility of $n=1,\ \cdots\ ,n_{\max}$ by inspecting the inequality

$$
  (nm\ \operatorname{mod}\ N) < \frac{ug+N}{q}.
$$

Furthermore, if $n$ turned out to be a multiple of $q$, then we can compute $n/q$ from $(nm\ \operatorname{mod}\ N)$ as in the classical Granlund-Montgomery case. More precisely, assume that $n=aq$ for some integer $a\geq 0$, then

$$\begin{align*}
  (nm\ \operatorname{mod}\ N)
  &= aqm - \left\lfloor \frac{aqm}{N}\right\rfloor N
  = aqm - \left(\left\lfloor \frac{a(qm - Np)}{N}\right\rfloor + ap\right)N \\
  &= a(qm - Np) - \left\lfloor \frac{ag}{N}\right\rfloor
  = ag - \left\lfloor \frac{ag}{N}\right\rfloor.
\end{align*}$$

We show that $ag < N$ always holds as long as $1<q<N$, $aq \leq n_{\max}$ hold and $n_{\max}$ satisfies $\eqref{eq:nmax condition}$. Indeed, $ag < N$ is equivalent to $aq < \frac{qN}{g}$, so it is enough to show the right-hand side of $\eqref{eq:nmax condition}$ is strictly less than $\frac{qN}{g}$. Note that

$$\begin{align*}
  \left\lfloor \frac{(N/g)+u}{q} \right\rfloor q + q-1-u
  &\leq \frac{(N/g)+u}{q} \cdot q + q - 1 - u \\
  &= \frac{N}{g} + q - 1,
\end{align*}$$

and

$$
  \frac{qN}{g} - \left(\frac{N}{g} + q - 1\right)
  = \left(\frac{N}{g} - 1\right)\left(q - 1\right) > 0
$$

by the condition $1<q<N$. Therefore, in this case we always have

$$
  (nm\ \operatorname{mod}\ N) = ag,
$$

thus $a$ can be recovered by computing $\frac{(nm\ \operatorname{mod}\ N)}{g}$. Note that in particular if $N=2^{b}$, division by $g$ is just a bit-shift.

Compared to the method proposed by Granlund-Montgomery, bit-rotation is never needed, but at the expense of requires an additional shift for computing $n/q$. Note that this shifting is only needed if $n$ turned out to be a multiple of $q$. The divisibility check alone can be done with one multiplication and one comparison.

So far it sounds like this new method is better than the classical Granlund-Montgomery algorithm based on bit-rotation, but note that the maximum possible value of $n_{\max}$ (i.e., the right-hand side of $\eqref{eq:nmax condition}$) is roughly of size $N/g$, so in particular, if $N=2^{b}$ and $q = 2^{t}q_{0}$ for some odd integer $q_{0}$, then $n_{\max}$ should be of at most about $(b-t)$-bits. Depending on the specific parameters, it is possible to improve the right-hand side of $\eqref{eq:nmax condition}$ a little bit by choosing a different $p$ (since $u$ is determined from $p$), but this cannot have any substantial impact on how large $n_{\max}$ can be. Also, at this moment I do not know of an elegant way of choosing $p$ that maximizes the bound on $n_{\max}$.

In summary, the algorithm works as follows, assuming $1<q<N$ and $N=2^{b}$.

1. Write $q = 2^{t}q_{0}$ for an odd integer $q_{0}$.

2. Let $m_{0}$ be the modular inverse of $q_{0}$ with respect to $2^{b-t}$, and let $p_{0}\mathrel{\unicode{x2254}} (q_{0}m_{0}-1)/2^{b-t}$.

3. If $p_{0}$ is odd, let $p\mathrel{\unicode{x2254}} p_{0}$, otherwise, let $p\mathrel{\unicode{x2254}} p_{0} + q_{0}$.

4. Let $m\mathrel{\unicode{x2254}} (2^{b-t}p + 1)/q_{0}$.

5. Let $u$ be the modular inverse of $p$ with respect to $q$.

6. Then for any $n=0,1,\ \cdots\ ,\left\lfloor (2^{b-t}+u)/q\right\rfloor q + q - 1 - u$, $n$ is a multiple of $q$ if and only if
    
    $$
      (nm\ \operatorname{mod}\ 2^{b}) < \frac{2^{b-t}+u}{q_{0}}.
    $$

7. In case the above inequality holds, we also have
    
    $$
      \frac{n}{q} = \frac{(nm\ \operatorname{mod}\ 2^{b})}{2^{t}}.
    $$

The corresponding code for factoring out the highest power of $q$ would look like the following:

```cpp
// m, threshold_value and t are precalculated from q.
std::size_t s = 0;
while (true) {
    auto const r = n * m;
    if (r < threshold_value) {
        n = r >> t;
        s += 1;
    }
    else {
        break;
    }
}
return {n, s};
```

One should keep in mind that the above only works provided $n$ is at most $\left\lfloor (2^{b-t}+u)/q\right\rfloor q + q - 1 - u$, which is roughly equal to $2^{b-t}$.

# Benchmark and conclusion

Determining the most efficient approach inherently depends on a multitude of factors. To gain insight into the relative performances of various algorithms, I conducted a benchmark. Given that my primary application involves removing trailing zeros, I set the divisor $q$ to $10$ for this benchmark. Additionally, considering the requirements of Dragonbox, where trailing zero removal may only be necessary for numbers up to $8$ digits for IEEE-754 binary32 and up to $16$ digits for IEEE-754 binary64, I incorporated these assumptions in the benchmark to determine the optimal parameters for each algorithm.

Here is the data I collected on my laptop (Intel(R) Core(TM) i7-7700HQ CPU @ 2.80GHz, Windows 10):

- 32-bit benchmark for numbers with at most 8 digits.

| Algorithms                                 | Average time consumed per a sample |
| -------------------------------------------|------------------------------------|
| Null (baseline)                            | 1.4035ns                           |
| Naïve                                      | 12.7084ns                          |
| Granlund-Montgomery                        | 11.8153ns                          |
| Lemire                                     | 12.2671ns                          |
| Generalized Granlund-Montgomery            | 11.2075ns                          |
| Naïve 2-1                                  | 8.92781ns                          |
| Granlund-Montgomery 2-1                    | 7.85643ns                          |
| Lemire 2-1                                 | 7.60924ns                          |
| Generalized Granlund-Montgomery 2-1        | 7.85875ns                          |
| Naïve branchless                           | 3.30768ns                          |
| Granlund-Montgomery branchless             | 2.52126ns                          |
| Lemire branchless                          | 2.71366ns                          |
| Generalized Granlund-Montgomery branchless | 2.51748ns                          |

- 64-bit benchmark for numbers with at most 16 digits.

| Algorithms                                 | Average time consumed per a sample |
| -------------------------------------------|------------------------------------|
| Null (baseline)                            | 1.68744ns                          |
| Naïve                                      | 16.5861ns                          |
| Granlund-Montgomery                        | 14.1657ns                          |
| Lemire                                     | 14.3427ns                          |
| Generalized Granlund-Montgomery            | 15.0626ns                          |
| Naïve 2-1                                  | 13.2377ns                          |
| Granlund-Montgomery 2-1                    | 11.3316ns                          |
| Lemire 2-1                                 | 11.6016ns                          |
| Generalized Granlund-Montgomery 2-1        | 11.8173ns                          |
| Naïve 8-2-1                                | 12.5984ns                          |
| Granlund-Montgomery 8-2-1                  | 11.0704ns                          |
| Lemire 8-2-1                               | 13.3804ns                          |
| Generalized Granlund-Montgomery 8-2-1      | 11.1482ns                          |
| Naïve branchless                           | 5.68382ns                          |
| Granlund-Montgomery branchless             | 4.0157ns                           |
| Lemire branchless                          | 4.92971ns                          |
| Generalized Granlund-Montgomery branchless | 4.64833ns                          |

(The code is available [here](https://github.com/jk-jeon/rtz_benchmark).)

And here are some detailed information on how the benchmark is done:
- Samples were generated randomly using the following procedure:
  - Uniformly randomly generate the total number of digits, ranging from 1 to the specified maximum number of digits.
  - Given the total number of digits, uniformly randomly generate the number of trailing zeros, ranging from 0 to the total number of digits minus 1.
  - Uniformly randomly generate an unsigned integer with given total number of digits and the number of trailing zeros.
- *Generalized Granlund-Montgomery* refers to the generalized modular inverse algorithm explained in the [last section](#generalized-modular-inverse-algorithm).
- Algorithms without any suffix iteratively remove trailing zeros one by one as demonstrated as code snippets in previous sections.
- Algorithms suffixed with "2-1" initially attempt to iteratively remove two consecutive trailing zeros at once (by running the loop with $q=100$), and then remove one more zero if necessary.
- Algorithms suffixed with "8-2-1" first check if the input contains at least eight trailing zeros (using the corresponding divisibility check algorithm with $q=10^{8}$), and if that is the case, then remove eight zeros and invoke the 32-bit "2-1" variants of themselves. If there are fewer than eight trailing zeros, then they proceed like their "2-1" variants.
- Algorithms suffixed with "branchless" do branchless binary search, as suggested by reddit users [r/pigeon768](https://www.reddit.com/user/pigeon768/) and [r/TheoreticalDumbass](https://www.reddit.com/user/TheoreticalDumbass/). (See [this reddit post](https://www.reddit.com/r/cpp/comments/1cbsobb/how_to_quickly_factor_out_a_constant_factor_from/).)

It seems there is not so much difference between all three algorithms overall, and even the naïve one is not so bad. There were notable fluctuation with repeated runs and the top performer varied run to run, but all three consistently outperformed the naïve approach, and I could observe a certain pattern.

Firstly, Lemire's algorithm seems to suffer for large divisors, for instance $q=10^{8}$ or maybe even $q=10^{4}$. This is probably because for large divisors the number of bits needed for a correct divisibility test often exceeds the word size. This means that comparing the low bits of the result of multiplication with the threshold value is not a simple single comparison in reality.

For instance, "Lemire 8-2-1" and "Lemire branchless" algorithms from the benchmark use $80$-bits for checking divisibility by $q=10^{8}$ for inputs of up to $16$ decimal digits. This means that, given the input is passed as a $64$-bit unsigned integer, we perform widening $128$-bit multiplication with a $64$-bit magic number $m$ (whose value is $12089258196146292$ in this case), and we check two things to decide divisibility: that the lowest $16$-bits from the upper half of the $128$-bit multiplication result is all zero, and that the lower half of this $128$-bit multiplication result is strictly less than $m$. Actually, the minimum required number of bits for a correct divisibility test is $78$ assuming inputs are limited to $16$ decimal digits, and I opted for $2$ more bits to facilitate the extraction of $16$-bits from the upper $64$-bits rather than $14$-bits.

(Note also that having to have more bits than the word size means that, even if all we care is divisibility without any need to determine the quotient, Lemire's algorithm may still need widening multiplication!)

Secondly, it seems on x86-64 the classical Granlund-Montgomery algorithm is just better than the proposed generalization of it, especially for the branchless case. Note that `ror` and `shr` have identical performance ([reference](https://www.agner.org/optimize/instruction_tables.pdf)), so for the branchless case, having to bit-shift *only when* the input is divisible turns out to be a pessimization, because we need to evaluate the bit-shift regardless of the divisibility, and as a result it just ends up requiring an additional `mov` compared to the unconditional bit-rotation of classical Granlund-Montgomery algorithm. Even for the branchful case, it seems the compiler still tries to evaluate the bit-shift regardless of the divisibility and as a consequence generates one more `mov`.

My conclusion is that, the classical Granlund-Montgomery is probably the best for trailing zero removal, at least on x86-64. Yet, the proposed generalized modular inverse algorithm may be a better choice on machines without `ror` instruction, or machines with smaller word size. Lemire's algorithm does not seem to offer any big advantage over the other two on x86-64, and I expect that the cost of widening multiplication may overwhelm the advantage of having only one magic number on machines with smaller word size.

On the other hand, Lemire's algorithm is still very useful if I have to know the quotient regardless of divisibility. There indeed is such an occasion in Dragonbox, where I am already leveraging Lemire's algorithm for this purpose.

Special thanks to reddit users [r/pigeon768](https://www.reddit.com/user/pigeon768/) and [r/TheoreticalDumbass](https://www.reddit.com/user/TheoreticalDumbass/) who proposed the branchless idea!