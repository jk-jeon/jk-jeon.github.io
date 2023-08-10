---
title: "On the Optimal Bounds for Integer Division by Constants"
date: 2023-08-07
permalink: /posts/2022/08/optimal-bounds-integer-division/
tags:
  - programming
  - C/C++
---

It is well-known that the integer division is quite a heavy operation on modern CPU's - so slow, in fact, that it has even become a common wisdom to *avoid doing it at ALL cost* in performance-critical sections of a program. I do not know why division is particularly hard to optimize from the hardware perspective. I am just guessing, maybe (1) every general algorithm is essentially just a minor variation of the good-old long division, (2) which is almost impossible to parallelize. But whatever, that is not the topic of this post.

Rather, this post is about some of the common optimization techniques for circumventing 
integer division. To be more precise, the post can be roughly divided into two parts. The first part discusses the well-known Granlund-Montgomery style multiply-and-shift technique and some associated issues. The second part is about my recent research on a related multiply-add-and-shift technique. In this section, I establish an optimal bound, which perhaps is a novel result. It would be intriguing to see whether modern compilers can take advantage of this new bound.

Note that by *integer division*, we specifically mean rgw computation of the quotient and/or the remainder, rather than evaluating the result as a real number. More specifically, throughout this entire post, *integer division* will always mean taking the quotient unless specified otherwise. Also, I will confine myself into divisions of positive integers, even though divisions of negative integers hold practical significance as well. Finally, all assembly code provided herein are for x86-64 architecture.

# Turning an integer division into a multiply-and-shift

One of the most widely used techniques is converting division into a multiplication when the divisor is a known constant (or remains mostly unchanged). The idea behind this approach is quite simple. For instance, dividing by $4$ is equivalent to multiplying by $0.25$, which can be further represented as multiplying by $25$ and then dividing by $100$, where dividing by $100$ is simply a matter of moving the decimal dot into left by two positions. Since we are only interested in taking the quotient, this means throwing away the last two digits.

In this particular instance, $4$ is a divisor of $100$ so we indeed have a fairly concise such a representation, but in general the divisor might not divide a power of $10$. Let us take $7$ as an example. In this case, we cannot write $\frac{1}{7}$ as $\frac{m}{10^{k}}$ for some positive integers $m,k$. However, we can still come up with a good *approximation*. Note that

$$
  \frac{1}{7} = 0.142857142857\cdots,
$$

so presumably something like $\frac{142858}{1000000}$ would be a good enough approximation of $\frac{1}{7}$. Taking that as our approximation, we may want to compute $n/7$ by multiplying $142858$ to $n$ and then throwing away the last $6$ digits. (Note that we are taking $142858$ instead of $142857$ because the latter already fails when $n=7$. In general, we must take ceiling, not floor nor half-up rounding.)

This indeed gives the right answer for all $n=1,\ \cdots\ ,166668$, but it starts to produce a wrong answer at $n=166669$, where the correct answer is $23809$, whereas our method produces $23810$. And of course such a failure is expected. Given that we are using an approximation with nonzero error, it is inevitable that the error will eventually manifest as the dividend $n$ grows large enough. But, the question at hand is, *can we estimate how far it will go?* Or, *how to choose a good enough approximation guaranteed to work correctly when there is a given limit on how big our $n$ can be?*

Obviously, our intention is to implement this concept in computer program, so the denominators of the approximations will be powers of $2$, not $10$. So, for a given positive integer $d$, our goal is to find a good enough approximation of $\frac{1}{d}$ of the form $\frac{m}{2^{k}}$. Perhaps one of the most widely known formal results in this realm is the following theorem by Granlund-Montgomery:

>**Theorem 1 (Granlund-Montgomery, 1994).**
>
>Suppose $m$, $d$, $k$ are nonnegative integers such that $d\neq 0$ 
>
>$$
>  2^{N+k} \leq md \leq 2^{N+k}+2^{k}.
>$$
>
>Then $\left\lfloor n/d \right\rfloor = \left\lfloor mn/2^{N+k} \right\rfloor$ for every integer $n$ with $0\leq n< 2^{N}$.

Here, $d$ is the given divisor and we are supposed to approximate $\frac{1}{d}$ by $\frac{m}{2^{N+k}}$. An assumption here is that we want to perform the division $n/d$ for all $n$ from $0$ to $2^{N}-1$, where $N$ is supposed to be the bit-width of the integer data type under consideration. In this setting, the mentioned theorem establishes a sufficient condition under which we can calculate the quotient of $n/d$ by first multiplying $n$ by $m$ and subsequently right-shifting the result by $(N+k)$-bits. Note that we will need more than $N$-bits; since our dividend is of $N$-bits, the result of the multiplication $mn$ will need to be stored in $2N$-bits. Since we are shifting by $(N+k)$-bits, the lower half of the result is actually not needed, and we just take the upper half and shift it by $k$-bits.

We will not talk about the proof of this theorem because a much more general theorem will be presented in the next section. But let us see how results like this are being applied in the wild. For example, consider the following C++ code:
```cpp
std::uint64_t div(std::uint64_t n) noexcept {
    return n / 17;
}
```

Modern compilers are well-aware of these Granlund-Montgomery-style division tricks. Indeed, my compiler (clang) adeptly harnessed such tactics, as demonstrated by its translation of the code above into the following lines of assembly instructions:
```asm
div(unsigned long):
        mov     rax, rdi
        movabs  rcx, -1085102592571150095
        mul     rcx
        mov     rax, rdx
        shr     rax, 4
        ret
```

([Check it out!](https://godbolt.org/z/vfGjsnE8P))

Note that we have $N=64$ and $d=17$ in this case, and the magic constant $-1085102592571150095$ you can see in the second line, interpreted as an unsigned value $17361641481138401521$, is the $m$ appearing in the theorem. You can also see in the fifth line that $k$ in this case is equal to $4$. What the assembly code is doing is as follows:
- Load the $64$-bit input $n$ into the $64$-bit `rax` register.
- Load the $64$-bit constant $17361641481138401521$ into the $64$-bit `rcx` register.
- Multiply them together, then the result is of $128$-bits. The lower half of this is stored back into `rax`, and the upper half is stored in the `rdx` register. But we do not care about the lower half, so throw it away and copy the upper half back into `rax`.
- Shift the result to the right by $4$-bits. The result must be equal to $n/17$.

We can easily check that these $m$ and $k$ chosen by the compiler indeed satisfy the inequalities in the theorem:

$$\begin{aligned}
  295147905179352825856 = 2^{64+4}
  &\leq md \\
  &= 17361641481138401521\cdot 17 \\
  &= 295147905179352825857 \\
  &\leq 2^{64+4} + 2^{4} \\
  &= 295147905179352825872.
\end{aligned}$$

We have not yet seen how to actually find $m$ and $k$ using a result like the theorem above, but before asking such a question, let us instead ask: *"is this condition on $m$ and $k$ the best possible one?"*

And the answer is: **No**.

<span id="granlund-montgomery-counterexample"></span>Here is an example, take $N=32$ and $d=102807$. In this case, the smallest $k$ that allows an integer $m$ that satisfies

$$
  2^{N+k} \leq md \leq 2^{N+k} + 2^{k}
$$

to exist is $k=17$, and in that case the unique $m$ satisfying the above is $5475793997$. This is kind of unfortunate, because the magic constant $m=5475793997$ is of $33$-bits, so the computation of $nm$ cannot be done inside $64$-bits. However, it turns out that we can take $k=16$ and $m=2737896999$ and still the equality $\left\lfloor \frac{n}{d}\right\rfloor = \left\lfloor \frac{nm}{2^{N+k}}\right\rfloor$ holds for all $n=0,\ \cdots\ ,2^{N}-1$, although the above inequality is not satisfied in this case. Now, the new constant $2737896999$ is of $32$-bits, so we can do our computation inside $64$-bits. This might result a massive difference in practice!

It [seems](https://godbolt.org/z/b3jcs9vMK) that even the most recent version of GCC (13.2) is still not aware of this, while clang knows that the above $m$ and $k$ works. (In the link provided, GCC is actually trying to compute $\left\lfloor \frac{nm}{2^{N+k}} \right\rfloor$ with the $33$-bit constant $m = 5475793997$. Detailed explanation will be given in a [later section](#when-the-magic-number-is-too-big).)

Then what is the best possible condition? I am not sure who was the first for finding the optimal bound, but at least it seems that such a bound is written in the famous book *Hacker's Delight* by H. S. Warren Jr. Also, recently (in 2021), [Lemire et al.](https://doi.org/10.1016/j.heliyon.2021.e07442) showed the optimality of an equivalent bound. I will not write down the optimal bounds obtained by these authors, because I will present a more general result proved by myself in the next section.

Here is one remark before getting into the next section. The aforementioned results on the optimal bound work for $n$ from the range $$\left\{1,\ \cdots\ ,n_{\max}\right\}$$ where $n_{\max}$ is *not necessarily of the form* $2^{N}-1$. However, it seems that even recent compilers do not seem to leverage this fact. For example, let us look at the following code:
```cpp
std::uint64_t div(std::uint64_t n) {
    [[assume(n < 10000000000)]];
    return n / 10;
}
```

Here we are relying on a new language feature added in C++23: `assume`. GCC generated the following lines of assemblies:
```asm
div(unsigned long):
        movabs  rax, -3689348814741910323
        mul     rdi
        mov     rax, rdx
        shr     rax, 3
        ret
```

([Check it out!](https://godbolt.org/z/Pr48e7Kja))

My gripe with this is the generation of a superfluous `shr` instruction. GCC seems to think that $k$ must be at least $67$ (which is why it shifted by $3$-bits, after throwing away the $64$-bit lower half), but actually $k=64$ is fine with the magic number $m=1844674407370955162$ thanks to the bound on $n$, in which case we do not need this additional shifting.

How about clang? Unlike GCC, clang currently does not seem to understand this new language feature `assume`. But it has an equivalent language extension `__builtin_assume`, so I tried with that:
```cpp
std::uint64_t div(std::uint64_t n) {
    __builtin_assume(n < 10000000000);
    return n / 10;
}
```

```asm
div(unsigned long):
        mov     rax, rdi
        movabs  rcx, -3689348814741910323
        mul     rcx
        mov     rax, rdx
        shr     rax, 3
        ret
```

([Check it out!](https://godbolt.org/z/7fGzYr8Mh))

And there is no big difference😥

# Turning multiplication by a real number into a multiply-and-shift

Actually, during the development of [Dragonbox](https://github.com/jk-jeon/dragonbox), I was interested in a more general problem of multiplying a rational number to $n$ and then finding out the integer part of the resulting rational number. In other words, my problem was not just about division, rather about multiplication followed by a division. This presence of multiplier certainly makes the situation a little bit more tricky, but it is anyway possible to derive the optimal bound in a similar way, which leads to the following generalization of the results mentioned in the previous section. *(Disclaimer: I am definitely not claiming to be the first who proved this, and I am sure an equivalent result could be found elsewhere, though I am not aware of any.)*

><b id=floor-computation>Theorem 2</b> (From [this paper](https://github.com/jk-jeon/dragonbox/blob/master/other_files/Dragonbox.pdf))**.**
>
>Let $x$ be a real number and $n_{\max}$ a positive integer. Then for a real number $\xi$, we have the followings.
>
>  1. If $x=\frac{p}{q}$ is a rational number with $q\leq n_{\max}$, then we have $\left\lfloor nx \right\rfloor = \left\lfloor n\xi \right\rfloor$ for all $n=1,\ \cdots\ ,n_{\max}$ if and only if $x \leq \xi < x + \frac{1}{vq}$  holds, where $v$ is the greatest integer such that $vp\equiv -1\ (\mathrm{mod}\ q)$ and $v\leq n_{\max}$.
>  
>  2. If $x$ is either irrational or a rational number with the denominator strictly greater than $n_{\max}$, then we have $\left\lfloor nx \right\rfloor = \left\lfloor n\xi \right\rfloor$ for all $n=1,\ \cdots\ ,n_{\max}$ if and only if $$\frac{p_{*}}{q_{*}} \leq \xi < \frac{p^{*}}{q^{*}}$$ holds, where $$\frac{p_{*}}{q_{*}}$$, $$\frac{p^{*}}{q^{*}}$$ are the best rational approximations of $x$ from below and above, respectively, with the largest denominators $$q_{*},q^{*}\leq n_{\max}$$.

Note that $\left\lfloor nx \right\rfloor$ is supposed to be the one we actually want to compute, while $\xi$ is supposed to be the chosen approximation of $x$. For the special case when $x = \frac{1}{d}$, $n_{\max} = 2^{N}-1$, and $\xi = \frac{m}{2^{N+k}}$, we obtain the setting of Granlund-Montgomery.

So I said rational number in the beginning of this section, but actually our $x$ can be any real number. Note, however, that the result depends on whether $x$ is *"effectively rational"* or not over the domain $n=1,\ \cdots\ ,n_{\max}$, i.e., whether or not there exists a multiplier $n$ which makes $nx$ into an integer.

Since I was working on floating-point conversion problems when I derived this theorem, for me the more relevant case was the second case, that is, when $x$ is "effectively irrational", because the numerator and the denominator of $x$ I was considering were some high powers of $2$ and $5$. But the first case is more relevant in the main theme of this post, i.e., integer division, so let us forget about these jargons like *best rational approximations* and such. (**Spoiler**: they will show up again in the next section.)

So let us focus on the first case. First of all, note that if $p=1$, that is, when $x = \frac{1}{q}$, then $v$ has a simpler description: it is the last multiple of $q$ in the range $1,\ \cdots\ ,n_{\max}+1$ minus one. If you care, you can check the aforementioned paper by [Lemire et al.](https://doi.org/10.1016/j.heliyon.2021.e07442) to see that their **Theorem 1** exactly corresponds to the resulting bound. In fact, in this special case it is rather easy to see why the best bound should be something like that.

Indeed, note that having the equality

$$
  \left\lfloor nx \right\rfloor = \left\lfloor n\xi \right\rfloor
$$

for all $n=1,\ \cdots\ ,n_{\max}$ is equivalent to having the inequality

$$
  \frac{\left\lfloor nx\right\rfloor}{n} \leq \xi
  < \frac{\left\lfloor nx\right\rfloor + 1}{n}
$$

for all such $n$. Hence, it is enough to find the largest possible value of the left-hand side and the smallest possible value of the right-hand side. Since we are assuming that the denominator of $x$ is bounded by $n_{\max}$, obviously the maximum value of the left-hand side is just $x$. Thus, it is enough to find the minimum value of the right-hand side. Note that we can write

$$
  n = q\left\lfloor \frac{n}{q} \right\rfloor + r
  = q\left\lfloor nx\right\rfloor + r
$$

where $r$ is the remainder of the division $n/q$. Replacing $\left\lfloor nx\right\rfloor$ by $n$ and $r$ using the above identity, we get

$$
  \frac{\left\lfloor nx\right\rfloor + 1}{n}
  = \frac{(n-r)/q + 1}{n} = \frac{n + (q-r)}{qn}
  = \frac{1}{q} + \frac{q-r}{qn}.
$$

Therefore, the minimization of the left-hand side is equivalent to the minimization of $\frac{q-r}{n}$. Intuitively, it appears reasonable to believe that the minimizer $n$ must have the largest possible remainder $r=q-1$, because for example if $r$ were set to be $q-2$ instead, then the numerator gets doubled, necessitating a proportionally larger $n$ to achieve a diminished value of $\frac{q-r}{n}$. Also, among $n$'s with $r=q-1$, obviously the largest $n$ yields the smallest value of $\frac{q-r}{n}$, so it sounds rational to say that probably the greatest $n$ with $r=q-1$ is the minimizer of $\frac{q-r}{n}$. Indeed, this is quite easy to prove: suppose we call such $n$ as $v$, and suppose that there is $n$ which is even better than $v$:

$$
  \frac{q-r}{n} \leq \frac{1}{v},
  \quad (q-r)v \leq n.
$$

Now, since $v$ divided by $q$ has the remainder $q-1$, the left-hand side and the right-hand side have the same remainder when divided by $q$. Therefore, the difference between the two must be either zero or at least $q$. But since $v$ is the *largest* one with the remainder $q-1$, it must be at least $n_{\max} - q + 1$, thus $n$ cannot be larger than $v$ by more than $q - 1$. Thus, the only possibility is $n=v$.

When $x=\frac{p}{q}$ and $p\neq 1$, the remainder $r$ of $np/q$ depends pretty randomly on $n$, so it is somewhat harder to be intuitively convinced that still the minimizer of $\frac{\left\lfloor nx\right\rfloor + 1}{n}$ should have $r = q - 1$. But the same logic as above still works just with a little bit of tweaks in this case. The full proof can be found in the paper mentioned, or [one](https://jk-jeon.github.io/posts/2021/12/continued-fraction-floor-mult/) of my previous posts.

## Some applications

Here I collected some applications of the presented theorem.

### Finding the first error case

In the [beginning](#turning-an-integer-division-into-a-multiply-and-shift) of the previous section, I claimed that

$$
  \left\lfloor \frac{n}{7} \right\rfloor
  = \left\lfloor \frac{n\cdot 142858}{1000000} \right\rfloor
$$

holds for all $n=1,\ \cdots\ ,166668$ but not for $n=166669$. Now we can see how did I get this. Note that $166669\equiv 6\ (\mathrm{mod}\ 7)$, so the range $$\left\{1,\ \cdots\ ,166668\right\}$$ and the range $$\left\{1,\ \cdots\ ,166669\right\}$$ have different $v$'s: it is $166662$ for the former, while it is $166669$ for the latter. And this makes the difference, because the inequality

$$
  \frac{142858}{1000000} < \frac{1}{7} + \frac{1}{7v}
$$

holds if and only if $v < 166667$. Thus, $n=166669$ is the first counterexample.

### Coming up with a better magic number than Granlund-Montgomery

We can also see now why a better magic number worked in the [example](#granlund-montgomery-counterexample) from the previous section. Let me repeat it here with the notation of [**Theorem 2**](#floor-computation): we have $x=1/102807$ and $n_{\max}=2^{32}-1$, so we have

$$
  \left\lfloor \frac{n}{102807} \right\rfloor
  = \left\lfloor \frac{nm}{2^{k}} \right\rfloor
$$

for all $n=1,\ \cdots\ ,2^{32}-1$ if and only if $\xi=\frac{m}{2^{k}}$ satisfies

$$
  \frac{1}{102807} \leq \frac{m}{2^{k}}
  < \frac{1}{102807} + \frac{1}{v\cdot 102807}.
$$

In this case, $v$ is the largest integer in the range $1,\ \cdots\ ,2^{32}-1$ which has the remainder $102806$ when divided by $102807$. Then it can be easily seen that $v=4294865231$, so the inequality above becomes

$$
  \frac{2^{k}}{102807} \leq m
  < \frac{41776\cdot 2^{k}}{4294865231}.
$$

The smallest $k$ that allows an integer solution to the above inequality is $k=48$, in which case the unique solution is $m=2737896999$.

### A textbook example for the case $p\neq 1$

Let me demonstrate how [**Theorem 2**](#floor-computation) can be used for the case $p\neq 1$. Suppose that we want to convert the temperature from Fahrenheit to Celsius. Obviously, representing such values only using integers is a funny idea, but let us pretend that we are completely serious (and hey, in 2023, *Fahrenheit itself* is a funny joke from the first place😂... except that I am currently living in an interesting country🤮). Or, if one *desperately* wants to make a really serious example, we can maybe think about doing the same thing with fixed-point fractional numbers. But whatever.

So the formula is, we first subtract $32$ and then multiply $5/9$. For the sake of simplicity, let us arbitrarily assume that our Fahrenheit temperature is ranging from $32^{\circ}\mathrm{F}$ to $580^{\circ}\mathrm{F}$ so in particular we do not run into negative numbers. After subtracting $32$, the range is from $0$ to $548$, so we take $n_{\max}=548$. With $x = \frac{5}{9}$, our $v$ is so the largest integer such that $v\leq 548$ and $5v\equiv 8\ (\mathrm{mod}\ 9)$, or equivalently, $v\equiv 7\ (\mathrm{mod}\ 9)$. The largest multiple of $9$ in the range is $540$, so we see $v=547$. Hence, the inequality we are given with is

$$
  \frac{5}{9} \leq \frac{m}{2^{k}} < \frac{5}{9} + \frac{1}{547\cdot 9}
  = \frac{304}{547}.
$$

The smallest $k$ that allows an integer solution is $k=10$, and the unique solution for that case is $m=569$. Therefore, the final formula would be:

$$
  \left\lfloor \frac{5(n-32)}{9} \right\rfloor =
  \left\lfloor \frac{569(n-32)}{2^{10}} \right\rfloor.
$$

### Will the case $p\neq 1$ be potentially relevant for compiler-writers?

Might be, but there is a caveat: in general, the compiler is not allowed to optimize an expression `n * p / q` into `(n * m) >> k`, because `n * p` can overflow and in that case dividing by `q` will give a weird answer. To be more specific, for unsigned integer types, C/C++ standards say that any overflows should wrap around, so the expression `n * p / q` is not really supposed to compute $\left\lfloor \frac{np}{q}\right\rfloor$, rather it is supposed to compute $\left\lfloor \frac{(np\ \mathrm{mod}\ 2^{N})}{q}\right\rfloor$ where $N$ is the bit-width, even though it is quite likely that the one who wrote the code actually wanted the former. On the other hand, for signed integer types, (a signed-equivalent of) [**Theorem 2**](#floor-computation) might be applicable, because signed overflows are specifically [defined to be undefined](https://en.cppreference.com/w/cpp/language/operator_arithmetic#Overflows). But presumably there are lots of code out there relying on the wrong assumption that signed overflows will wrap around, so maybe compiler-writers do not want to do this kind of optimizations.

Nevertheless, there are situations where doing this is perfectly legal. For example, suppose `n`, `p`, `q` are all of type `std::uint32_t` and the code is written like `static_cast<std::uint32_t>((static_cast<std::uint64_t>(n) * p) / q)` to intentionally avoid this overflow issue. Then the compiler might recognize such a pattern and do this kind of optimizations. Or more generally, with the new `assume` attribute (or some other equivalent compiler-specific mechanisms), the user might give some assumptions that ensure no overflow.

It seems that currently both clang and GCC [do not do this](https://godbolt.org/z/KhM14oqbE), so if they want to do so in a future, then [**Theorem 2**](#floor-computation) might be useful. But how many code can benefit from such an optimization? Will it be really worth implementing? I do not know, but maybe not.

# When the magic number is too big

Even with the optimal bound, there exist situations where the smallest possible magic number does not fit into the word size. For example, consider the case $n_{\max} = 2^{64}-1$ and $x = 1/10961$. In this case, our $v$ is

$$
  v = \left\lfloor \frac{2^{64} - 10961}{10961} \right\rfloor \cdot 10961 + 10960
  = 18446744073709550681.
$$

Hence, the inequality we need to inspect is

$$\begin{aligned}
  \frac{1}{10961} \leq \frac{m}{2^{k}}
  & < \frac{1}{10961} + \frac{1}{10961\cdot 18446744073709550681} \\
  &= \frac{1682943533775162}{18446744073709550681},
\end{aligned}$$

and the smallest $k$ allowing an integer solution is $78$, in which case the unique solution is $m = 27573346857372255605$. And unfortunately, this is a $65$-bit number!

Let us see how to deal with this case by looking at what a compiler actually does. Consider the code:
```cpp
std::uint64_t div(std::uint64_t n) {
    return n / 10961;
}
```
My compiler (clang, again) generated the following lines of assemblies:
```asm
div(unsigned long):
        movabs  rcx, 9126602783662703989
        mov     rax, rdi
        mul     rcx
        sub     rdi, rdx
        shr     rdi
        lea     rax, [rdi + rdx]
        shr     rax, 13
        ret
```
([Check it out!](https://godbolt.org/z/fa3bM8TMx))

This is definitely a little bit more complicated than the happy case, but if we think carefully, we can realize that this is still just computing $\left\lfloor \frac{nm}{2^{k}}\right \rfloor$:
- The magic number $m'=9126602783662703989$ is precisely $m - 2^{64}$.
- In the third line, we multiply this magic number with the input. Let us call the upper half of the result as $u$ (which is stored in `rdx`), and the lower half as $\ell$ (which is stored in `rax`, and we do not care about the lower half anyway).
- We subtract the upper half $u$ from the input $n$ (the `sub` line), divide the result by 2 (the `shr` line), add the result back to $u$ (the `lea` line), and then store the end result into `rax`. Now this looks a bit puzzling, but what it really does is nothing but to compute $\left\lfloor (n + u)/2 \right\rfloor = \left\lfloor nm / 2^{65}\right\rfloor$. The reason why we first subtract $u$ from $n$ is to avoid overflow. On the other hand, the subtraction is totally fine as there can be no underflow, because $u = \left\lfloor nm'/2^{64}\right\rfloor$ is at most $n$, as $m'$ is less than $2^{64}$.
- Recall $k = 78$, so we want to compute $\left\lfloor nm/2^{78} \right\rfloor$. Since we got $\left\lfloor nm / 2^{65}\right\rfloor$ from the previous step, we just need to shift this further by $13$-bits.

This is not so bad, we just have two more trivial instructions compared to the happy case. But the reason why the above works is largely due to that the magic number is just one bit larger than $64$-bits. Well, can it be even larger than that?

The answer is: **No**, fortunately, the magic number being just one bit larger than the word size is the worst case.

To see why, note that the size of the interval where $\xi=\frac{m}{2^{k}}$ can possibly live is precisely $1/vq$. Therefore, if $k$ is large enough so that $2^{k}\geq vq$, then the difference between the endpoints of the inequality

$$
  \frac{2^{k}}{q} \leq m < 2^{k}\left(\frac{1}{q} + \frac{1}{vq}\right)
$$

is at least $1$, so the inequality must admit an integer solution. Now, we are interested in the bit-width of $\left\lceil\frac{2^{k}}{q}\right\rceil$, which must be the smallest possible magic number if $k$ admits at least one solution. Since the smallest admissible $k$ is at most the smallest $k$ satisfying $2^{k}\geq vq$, thus we must have $vq>2^{k-1}$, so $\frac{2^{k}}{q} < 2v$. Clearly, the right-hand side is of at most one bit larger than the word size.

Actually, there is an alternative way of dealing with the case of too large magic number, which is to consider a slightly different formula: instead of just doing a multiplication and then a shift, perform an addition by another magic number between those two operations. Using the notations from [**Theorem 2**](#floor-computation), what this means is that we have some $\zeta$ satisfying the equality

$$
  \left\lfloor nx \right\rfloor = \left\lfloor n\xi + \zeta \right\rfloor
$$

instead of $\left\lfloor nx \right\rfloor = \left\lfloor n\xi \right\rfloor$, where $\xi=\frac{m}{2^{k}}$ and $\zeta=\frac{s}{2^{k}}$ so that

$$
  \left\lfloor n\xi + \zeta \right\rfloor
  = \left\lfloor \frac{nm + s}{2^{k}}\right\rfloor
$$

can be computed by performing a multiplication, an addition, and a shift.

The inclusion of this $\zeta$ might allow us to use a smaller magic number, thereby enabling its accommodation within a single word. The next section is dedicated to the condition for having the above equality.

Here is a small remark before we start the (extensive) discussion of the optimal bound for this. Note that this trick of including the $\zeta$ term is probably not so useful for $64$-bit divisions, because addition is not really a trivial operation in this case. This is because the result of the multiplication $mn$ spans two $64$-bit blocks. Hence, we need an `adc` (add-with-carry) instruction or an equivalent, which is not particularly well-optimized in typical x86-64 CPU's. While I have not conducted a benchmark, I speculate that this approach probably result in worse performance. However, this trick might give a better performance for the $32$-bit case than the method explained in this section, as every operation can be done inside $64$-bits. Interestingly, it [seems](https://godbolt.org/z/zqKW3WhEn) modern compilers do not use this trick anyway. In the link provided, the compiler is trying to compute multiplication by $4999244749$ followed by the shift by $49$-bits. However, it turns out, by multiplying $1249811187$, adding $1249811187$, and then shifting to right by $47$-bits, we can do this computation completely within $64$-bits. I do not know whether the reason why the compilers do not leverage this trick is because it does not perform better, or just because they did not bother to implement it.

# Multiply-add-and-shift rather than multiply-shift

>**WARNING**: The contents of this section is **substantially** more complicated and math-heavy than previous sections.

As said in the last section, we will explore the condition for having

$$
  \left\lfloor nx \right\rfloor = \left\lfloor n\xi + \zeta \right\rfloor
$$

for all $n=1,\ \cdots\ ,n_{\max}$, where $x$, $\xi$ are real numbers and $\zeta$ is a nonnegative real number. We will derive the optimal bound, i.e., an "if and only if" condition. We remark that an optimal bound has been obtained in the paper by [Lemire et al.](https://doi.org/10.1016/j.heliyon.2021.e07442) mentioned above for the special case when $x=\frac{1}{q}$ for some $q\leq n_{\max}$ and $\xi=\zeta$ and is effectively rational. According to their paper, the proof of the optimality of their bound is almost identical to the case of having no $\zeta$, so they even did not bother to write down the proof. I provided a proof of this special case in a later [subsection](#the-result-by-lemire-et-al). The proof I wrote seem to rely on a heavy result proved below, but it can be done without it pretty easily as well.

However, in the general case I am dealing here, i.e., the only restriction I have is $\zeta\geq 0$, the situation is quite more complicated. Nevertheless, even in this generality, it is possible to give a very concrete description of how exactly the presence of $\zeta$ distorts the optimal bound.

Just like the case $\zeta=0$ (i.e., [**Theorem 2**](#floor-computation)), having the equality for all $n=1,\ \cdots\ ,n_{\max}$ is equivalent to having the inequality

$$
  \max_{n=1,\ \cdots\ ,n_{\max}}\frac{\left\lfloor nx\right\rfloor - \zeta}{n}
  \leq \xi
  <\min_{n=1,\ \cdots\ ,n_{\max}}\frac{\left\lfloor nx\right\rfloor + 1 - \zeta}{n},
$$

so the question is how to evaluate the maximum and the minimum in the above. By the reason that will become apparent as we proceed, we will in fact try to find the *largest* maximizer of the left-hand side and the *smallest* minimizer of the right-hand side.

## The lower bound

Since $\zeta\geq 0$ is supposed to be just a small constant, it sounds reasonable to believe that the maximizer of $\frac{\left\lfloor nx\right\rfloor}{n}$ is probably quite close to be the maximizer of $\frac{\left\lfloor nx\right\rfloor - \zeta}{n}$ as well. So let us start there: let $n_{0}$ be the largest among all maximizers of $\frac{\left\lfloor nx\right\rfloor}{n}$. Now, what should happen if $n_{0}$ were not the maximizer of $\frac{\left\lfloor nx\right\rfloor - \zeta}{n}$? Say, what can we say about $n$'s such that

$$
  \frac{\left\lfloor nx\right\rfloor - \zeta}{n}
  \geq \frac{\left\lfloor n_{0}x\right\rfloor - \zeta}{n_{0}}
$$

holds? First of all, since this inequality implies

$$
  -\frac{\zeta}{n} \geq -\frac{\zeta}{n_{0}},
$$

we must have $n\geq n_{0}$ unless $\zeta=0$, which is an uninteresting case anyway.

As a result, we can equivalently reformulate our optimization problem as follows: given $n=0,\ \cdots\ ,n_{\max}-n_{0}$, find the largest maximizer of

$$
  \frac{\left\lfloor (n_{0}+n)x \right\rfloor - \zeta}{n_{0}+n}.
$$

Now, we claim that

$$\label{eq:floor splits; lower bound}
  \left\lfloor (n_{0}+n)x \right\rfloor
  = \left\lfloor nx \right\rfloor + \left\lfloor n_{0}x \right\rfloor
$$

holds for all such $n$. (Note that in general $\left\lfloor x+y\right\rfloor$ is equal to either $\left\lfloor x\right\rfloor + \left\lfloor y\right\rfloor$ or $\left\lfloor x\right\rfloor + \left\lfloor y\right\rfloor + 1$.) This follows from the fact that $\frac{\left\lfloor n_{0}x\right\rfloor}{n_{0}}$ is not only the best rational approximation from below *in the weak sense*, but also *in the strong sense*. Okay, so at this point there is no way to get around these jargons anymore, so let us define them formally.

>**Definition 3** (Best rational approximations from below/above)**.**
>
>Let $x$ be a real number. We say a rational number $\frac{p}{q}$ (in its reduced form, which is always assumed if not specified otherwise) is a **best rational approximation from below (above, resp.)** if $\frac{p}{q}\leq x$ ($\frac{p}{q}\geq x$, resp.) and for any rational number $\frac{a}{b}$ with $\frac{p}{q}\leq\frac{a}{b}\leq x$ ($\frac{p}{q}\geq\frac{a}{b}\geq x$, resp.), we always have $q\leq b$.

In other words, $\frac{p}{q}$ is a best rational approximation of $x$ if any better rational approximation must have a larger denominator.

>**Remark.** Note that the terminology **best rational approximation** is pretty standard in pure mathematics, but it usually disregards the direction of approximation, from below or above. For example, $\frac{1}{3}$ is a best rational approximation from below of $\frac{3}{7}$, but it is **not** a best rational approximation in the usual, **non-directional** sense, because $\frac{1}{2}$ is a better approximation with a strictly less denominator. But this concept of non-directional best rational approximation is quite irrelevant to our application, so any usage of the term **best rational approximation** in this post always means the directional ones, either from below or above.

The definition provided above is the one *in the weak sense*, although it is not explicitly written so. The corresponding one in the strong sense is given below:

>**Definition 4** (Best rational approximations from below/above in the strong sense)**.**
>
>Let $x$ be a real number. We say a rational number $\frac{p}{q}$ (again, in its reduced form) is a **best rational approximation from below (above, resp.) in the strong sense**, if $\frac{p}{q}\leq x$ ($\frac{p}{q}\geq x$, resp.) and for any rational number $\frac{a}{b}$ with $qx - p\geq bx - a\geq 0$ ($p - qx\geq a - bx \geq 0$, resp.), we always have $q\leq b$.

As the name suggests, if $\frac{p}{q}$ is a best rational approximation from below (above, resp.) in the strong sense, then it is a best rational approximation from below (above, resp.) in the weak sense. To see why, take any rational number $\frac{a}{b}$ such that $\frac{p}{q}\leq\frac{a}{b}\leq x$ holds, then it is enough to show that $q\leq b$ must hold. (We are only considering the "from below" case, and the "from above" case can be done in the same way.) This is indeed quite easy to show: since $x - \frac{p}{q} \geq x - \frac{a}{b}$ holds, multiplying $q$ on both sides yields

$$
  qx - p \geq \frac{q}{b}(bx - a).
$$

If $b<q$, then the right-hand side is at least $bx - a$, but then by the assumption that $\frac{p}{q}$ is a best rational approximation in the strong sense, we must have $q\leq b$, which is a contradiction. Thus, we get $q\leq b$, so $\frac{p}{q}$ is indeed a best rational approximation from below in the weak sense.

Remarkably, using the theory of [continued fractions](https://en.wikipedia.org/wiki/Continued_fraction), it can be shown that the converse is also true. Note that this fact is entirely not trivial only by looking at their definitions. You can find a proof of this fact in my [paper](https://github.com/jk-jeon/dragonbox/blob/master/other_files/Dragonbox.pdf) on Dragonbox; see the remark after **Algorithm C.13** (I shamelessly give my own writing as a reference because I do not know of any other).

Alright, but what is the point of these nonsenses?

First, as suggested before, our $\frac{\left\lfloor n_{0}x\right\rfloor}{n_{0}}$ is a best rational approximation from below of $x$. This is obvious from its definition: $n_{0}$ is the maximizer of $\frac{\left\lfloor nx\right\rfloor}{n}$ for $n=1,\ \cdots\ ,n_{\max}$, which means that whenever we have

$$
  \frac{\left\lfloor n_{0}x \right\rfloor}{n_{0}}
  < \frac{a}{b} \leq x,
$$

then since $a \leq \left\lfloor bx\right\rfloor$ holds (because otherwise the inequality $\frac{a}{b}\leq x$ fails to hold), we must have $b>n_{\max}$, so in particular $b>n_{0}$.

Therefore, by the aforementioned equivalence of the two concepts, we immediately know that $\frac{\left\lfloor n_{0}x\right\rfloor}{n_{0}}$ is a best rational approximation from below of $x$ in the strong sense. Note that this does *not* imply that $n_{0}$ is a minimizer of $nx - \left\lfloor nx\right\rfloor$ for $n=1,\ \cdots\ ,n_{\max}$ because $\frac{\left\lfloor n_{0}x\right\rfloor}{n_{0}}$ is not necessarily in its reduced form (since we chose $n_{0}$ to be the largest maximizer). Still, if we denote its reduced form as $\frac{p}{q}$, then certainly $q$ is the minimizer of it (and $p$ must be equal to $\left\lfloor qx\right\rfloor$). Indeed, by the definition of best rational approximations from below in the strong sense, if $n$ is the smallest integer such that $n>q$ and

$$
  nx - \left\lfloor nx\right\rfloor < qx - p,
$$

then $\frac{\left\lfloor nx\right\rfloor}{n}$ itself must be a best rational approximation from below in the strong sense, thus also in the weak sense. This in particular means $\frac{\left\lfloor nx\right\rfloor}{n}\geq\frac{p}{q} = \frac{\left\lfloor n_{0}x\right\rfloor}{n_{0}}$ holds since $n>q$, and in fact the equality cannot hold because otherwise we get

$$
  qx - p
  = qx - \frac{q}{n}\left\lfloor nx\right\rfloor
  = \frac{q}{n}\left(nx - \left\lfloor nx\right\rfloor\right)
  < \frac{q}{n}(qx - p)
$$

which contradicts to $n>q$. Therefore, by the definition of $n_{0}$, we must have $n>n_{\max}$, so in particular $n>n_{0}$.

Using this fact, now we can easily prove $\eqref{eq:floor splits; lower bound}$. Again let $\frac{p}{q}$ be the reduced form of $\frac{\left\lfloor n_{0}x\right\rfloor}{n_{0}}$, then for any $n = 0,\ \cdots\ ,n_{\max} - q$, we must have

$$
  (q+n)x - \left\lfloor (q+n)x\right\rfloor \geq qx - p,
$$

so we get

$$
  \left\lfloor (q+n)x\right\rfloor
  \leq \left\lfloor qx\right\rfloor+ nx
  < \left\lfloor qx\right\rfloor + \left\lfloor nx\right\rfloor + 1,
$$

and this rules out the possibility $\left\lfloor (q+n)x\right\rfloor = \left\lfloor qx\right\rfloor + \left\lfloor nx\right\rfloor + 1$, thus we must have $\left\lfloor (q+n)x\right\rfloor = \left\lfloor qx\right\rfloor + \left\lfloor nx\right\rfloor$. Then inductively, we get that for any positive integer $k$ such that $kq\leq n_{\max}$ and $n=0,\ \cdots\ ,n_{\max} - kq$,

$$
  \left\lfloor (kq+n)x\right\rfloor
  = \left\lfloor qx\right\rfloor + \left\lfloor ((k-1)q + n)x\right\rfloor
  =\ \cdots\ = \left\lfloor kqx\right\rfloor + \left\lfloor nx\right\rfloor.
$$

Hence, we must have $\left\lfloor (n_{0}+n)x\right\rfloor = \left\lfloor n_{0}x\right\rfloor + \left\lfloor nx\right\rfloor$ in particular.

(Here is a more intuitive explanation. Note that among all $nx$'s, $qx$ is the one with the smallest fractional part. Then whenever $kq + n\leq n_{\max}$, the sum of the fractional parts of $nx$ and that of $k$ copies of $qx$ should not "wrap around", i.e., it should be strictly less than $1$, because if we choose the smallest $k$ that $(kq + n)x$ wraps around, then since $((k-1)q + n)x$ should not wrap around, its fractional part is strictly less than $1$, which means that the fractional part of $(kq + n)x$ is strictly less than that of $qx$, contradicting to the minimality of the fractional part of $qx$. Hence, since the sum of all fractional parts is still strictly less than $1$, the integer part of $(kq + n)x$ should be just the sum of the integer parts of its summands.)

Next, using the claim, now we can characterize $n=1,\ \cdots\ ,n_{\max} - n_{0}$ which yields a larger value for

$$
  \frac{\left\lfloor (n_{0}+n)x \right\rfloor - \zeta}{n_{0}+n}
$$

than the case $n=0$. Indeed, we have the inequality

$$
  \frac{\left\lfloor (n_{0}+n)x \right\rfloor - \zeta}{n_{0}+n}
  \geq \frac{\left\lfloor n_{0}x \right\rfloor - \zeta}{n_{0}}
$$

if and only if

$$
  n_{0}\left\lfloor nx\right\rfloor - n_{0}\zeta
  \geq n\left\lfloor n_{0}x\right\rfloor - (n_{0}+n)\zeta,
$$

if and only if

$$
  \frac{\left\lfloor nx\right\rfloor}{n} \geq
  \frac{\left\lfloor n_{0}x\right\rfloor - \zeta}{n_{0}},
$$

or equivalently,

$$\label{eq:lower bound iteration criterion}
  x - \frac{\left\lfloor nx\right\rfloor}{n}
  \leq \frac{\zeta}{n_{0}}
  + \left(x - \frac{\left\lfloor n_{0}x\right\rfloor}{n_{0}}\right).
$$

Therefore, in a sense, $\frac{\left\lfloor nx\right\rfloor}{n}$ itself should be a good enough approximation of $x$, although it does not need to (and cannot) be a better approximation than $\frac{\left\lfloor n_{0}x\right\rfloor}{n_{0}}$.

At this point, we could just enumerate all rational approximations of $x$ from below satisfying the above bound and find out the one that maximizes $\frac{\left\lfloor (n_{0}+n)x \right\rfloor - \zeta}{n_{0}+n}$. Indeed, the theory of continued fractions allows us to develop an efficient algorithm for doing such a task. (See **Algorithm C.13** in my [paper](https://github.com/jk-jeon/dragonbox/blob/master/other_files/Dragonbox.pdf) on Dragonbox for an algorithm doing a closely related task.) However, we can do better here.

First of all, note that if $\zeta$ is small enough, then $\eqref{eq:lower bound iteration criterion}$ does not have any solution for $n=1,\ \cdots\ ,n_{\max} - n_{0}$, which means $n_{0}$ is indeed the maximizer we are looking for and there is nothing further we need to do. This conclusion is consistent with the intuition that $n_{0}$ should be close enough to the maximizer of $\frac{\left\lfloor nx\right\rfloor -\zeta}{n}$ at least if $\zeta$ is close to zero. But the problem occurs when $\zeta$ is not that small.

It is quite tempting to claim that the maximizer of the left-hand side of $\eqref{eq:lower bound iteration criterion}$ is the maximizer of $\frac{\left\lfloor (n_{0}+n)x \right\rfloor - \zeta}{n_{0}+n}$, but that is not true in general. Nevertheless, we can start from there, just like that we started from $n_{0}$ from the beginning.

In this reason, let $n_{1}$ be the largest maximizer of $\frac{\left\lfloor nx\right\rfloor}{n}$ for $n=1,\ \cdots\ ,n_{\max} - n_{0}$. As pointed out earlier, if such $n_{1}$ does not satisfy $\eqref{eq:lower bound iteration criterion}$, then there is nothing further to do, so suppose that the inequality is indeed satisfied with $n=n_{1}$.

Next, we claim that

$$
  \frac{\left\lfloor (n_{0}+n)x \right\rfloor - \zeta}{n_{0}+n}
  \leq \frac{\left\lfloor (n_{0}+n_{1})x \right\rfloor - \zeta}{n_{0}+n_{1}}
$$

holds for all $n=1,\ \cdots\ ,n_{1}$.

Note that by $\eqref{eq:floor splits; lower bound}$, this is equivalent to

$$\begin{aligned}
  &n_{0}\left\lfloor nx\right\rfloor
  + n_{1}\left(\left\lfloor n_{0}x\right\rfloor + \left\lfloor nx\right\rfloor\right)
  - (n_{0}+n_{1})\zeta  \\
  &\quad\quad\quad \leq
  n_{0}\left\lfloor n_{1}x\right\rfloor
  + n\left(\left\lfloor n_{0}x\right\rfloor + \left\lfloor n_{1}x\right\rfloor\right)
  - (n_{0}+n)\zeta,
\end{aligned}$$

and rearranging it gives

$$
  (n_{1}-n)(\zeta - \left\lfloor n_{0}x\right\rfloor)
  \geq 
  \left(n_{1}\left\lfloor nx\right\rfloor - n\left\lfloor n_{1}x\right\rfloor\right)
  + n_{0}\left(
    \left\lfloor nx\right\rfloor - \left\lfloor n_{1}x\right\rfloor
  \right).
$$

By adding and subtracting appropriate terms, we can rewrite this as

$$\begin{aligned}
  &n_{0}(n_{1}-n)\left(\frac{\zeta}{n_{0}}
  + \left(x - \frac{\left\lfloor n_{0}x\right\rfloor}{n_{0}}\right)\right) \\
  &\quad\quad\quad\geq
  n_{1}(n_{0}+n)\left(
    x - \frac{\left\lfloor n_{1}x\right\rfloor}{n_{1}}
  \right)
  - n(n_{0}+n_{1})\left(
    x - \frac{\left\lfloor nx\right\rfloor}{n}
  \right).
\end{aligned}$$

Recall that we already have assumed

$$
  x - \frac{\left\lfloor n_{1}x\right\rfloor}{n_{1}}
  \leq \frac{\zeta}{n_{0}}
  + \left(x - \frac{\left\lfloor n_{0}x\right\rfloor}{n_{0}}\right),
$$

so it is enough to show

$$\begin{aligned}
  &n_{0}(n_{1} - n)\left(
    x - \frac{\left\lfloor n_{1}x\right\rfloor}{n_{1}}
  \right) \\
  &\quad\quad\quad\geq
  n_{1}(n_{0}+n)\left(
    x - \frac{\left\lfloor n_{1}x\right\rfloor}{n_{1}}
  \right)
  - n(n_{0}+n_{1})\left(
    x - \frac{\left\lfloor nx\right\rfloor}{n}
  \right),
\end{aligned}$$

or equivalently,

$$\begin{aligned}
  n(n_{0}+n_{1})\left(x - \frac{\left\lfloor nx\right\rfloor}{n}\right)
  &\geq \left(n_{1}(n_{0}+n) - n_{0}(n_{1} - n)\right)
  \left(x - \frac{\left\lfloor n_{1}x\right\rfloor}{n_{1}}\right) \\
  &= n(n_{0}+n_{1})
  \left(x - \frac{\left\lfloor n_{1}x\right\rfloor}{n_{1}}\right),
\end{aligned}$$

which trivially holds by the definition of $n_{1}$. Thus the claim is proved.

As a result, we can reformulate our optimization problem again in the following way: define $N_{1}:=n_{0}+n_{1}$, then we are to find $n=0,\ \cdots\ ,n_{\max} - N_{1}$ which maximizes

$$
  \frac{\left\lfloor (N_{1}+n)x\right\rfloor - \zeta}{N_{1}+n}.
$$

As you can see, now it resembles quite a lot what we just have done. Then the natural next step is to see whether we have

$$
  \left\lfloor (N_{1}+n)x\right\rfloor =
  \left\lfloor N_{1}x\right\rfloor + \left\lfloor nx\right\rfloor,
$$

which, by $\eqref{eq:floor splits; lower bound}$, is equivalent to

$$
  \left\lfloor (n_{1}+n)x\right\rfloor =
  \left\lfloor n_{1}x\right\rfloor + \left\lfloor nx\right\rfloor.
$$

But we already know this: because $\frac{\left\lfloor n_{1}x\right\rfloor}{n_{1}}$ is a best rational approximation from below of $x$, the exact same proof of $\eqref{eq:floor splits; lower bound}$ applies.

Therefore, by the same procedure as we did, we get that

$$
  \frac{\left\lfloor (N_{1}+n)x \right\rfloor - \zeta}{N_{1}+n}
  \geq \frac{\left\lfloor N_{1}x \right\rfloor - \zeta}{N_{1}}
$$

holds for $n=1,\ \cdots\ ,n_{\max} - N_{1}$ if and only if

$$
  \frac{\left\lfloor nx\right\rfloor}{n}
  \geq \frac{\left\lfloor N_{1}x\right\rfloor - \zeta}{N_{1}}.
$$

Hence, we finally arrive at the following iterative algorithm for computing the maximizer of $\frac{\left\lfloor nx\right\rfloor - \zeta}{n}$:

><b id="lower-bound-algorithm">Algorithm 5</b> (Computing the lower bound)**.**
>
>Input: $x\in\mathbb{R}$, $n_{\max}\in\mathbb{Z}_{>0}$, $\zeta\geq 0$.
>
>Output: the largest maximizer of $\frac{\left\lfloor nx\right\rfloor - \zeta}{n}$ for $n=1,\ \cdots\ ,n_{\max}$.
>
>1. Find the largest $n=1,\ \cdots\ ,n_{\max}$ that maximizes $\frac{\left\lfloor nx\right\rfloor}{n}$ and call it $n_{0}$.
>2. If $n_{0} = n_{\max}$, then $n_{0}$ is the largest maximizer; return.
>3. Otherwise, find the largest $n=1,\ \cdots\ ,n_{\max} - n_{0}$ that maximizes $\frac{\left\lfloor nx\right\rfloor}{n}$ and call it $n_{1}$.
>4. Inspect the inequality
>\\[
>  \frac{\left\lfloor n_{1}x\right\rfloor}{n_{1}}
>  \geq \frac{\left\lfloor n_{0}x\right\rfloor - \zeta}{n_{0}}.
>\\]
>5. If the inequality does not hold, then $n_{0}$ is the largest maximizer; return.
>6. If the inequality does hold, then set $n_{0}\leftarrow n_{0} + n_{1}$ and go to Step 2.

*Remark*. In the above, we blackboxed the operation of finding the largest maximizer of $\frac{\left\lfloor nx\right\rfloor}{n}$. Actually, this is precisely how we obtain [**Theorem 2**](#floor-computation). If $x$ is a rational number whose denominator $q$ is at most $n_{\max}$, then obviously the largest maximizer is the largest multiple of $q$ bounded by $n_{\max}$. Otherwise, we just compute the best rational approximation from below of $x$ with the largest denominator $$q_{*}\leq n_{\max}$$, where the theory of continued fractions allows us to compute this very efficiently. (Especially, when $x$ is a rational number, this efficient algorithm is really just the extended Euclid algorithm.) Once we get the best rational approximation $$\frac{p_{*}}{q_{*}}$$ (which must be in its reduced form), we find the largest multiple $$kq_{*}$$ of $$q_{*}$$ bounded by $n_{\max}$. Then since $$\frac{p_{*}}{q_{*}}$$ is the maximizer of $\frac{\left\lfloor nx\right\rfloor}{n}$ for $n=1,\ \cdots\ ,n_{\max}$, it follows that $$\left\lfloor kq_{*}x\right\rfloor$$ must be equal to $$kp_{*}$$ and $$kq_{*}$$ is the largest maximizer of $\frac{\left\lfloor nx\right\rfloor}{n}$.

## The upper bound

The computation of the upper bound, that is, finding the smallest minimizer of

$$
  \frac{\left\lfloor nx\right\rfloor + 1 - \zeta}{n}
$$

for $n=1,\ \cdots\ ,n_{\max}$, is largely same as the case of lower bound. In this case, we start with the smallest minimizer $n_{0}$ of $\frac{\left\lfloor nx\right\rfloor + 1}{n}$. Then for any $n=1,\ \cdots\ ,n_{\max}$ with

$$
  \frac{\left\lfloor nx\right\rfloor + 1-\zeta}{n}
  \leq \frac{\left\lfloor n_{0}x\right\rfloor + 1 - \zeta}{n_{0}},
$$

we must have

$$
  -\frac{\zeta}{n} \leq -\frac{\zeta}{n_{0}},
$$

thus $n\leq n_{0}$ unless $\zeta=0$, which again is an uninteresting case.

Hence, our goal is to find $n=0,\ \cdots\ ,n_{0}-1$ minimizing

$$
  \frac{\left\lfloor (n_{0}-n)x\right\rfloor + 1 - \zeta}{n_{0} - n}.
$$

Again, we claim that

$$\label{eq:floor splits; upper bound}
  \left\lfloor (n_{0}-n)x\right\rfloor
  = \left\lfloor n_{0}x\right\rfloor
  - \left\lfloor nx\right\rfloor
$$

holds for all such $n$. We have two cases: (1) when $\frac{\left\lfloor n_{0}x\right\rfloor + 1}{n_{0}}$ is a best rational approximation from above of $x$, or (2) when it is not. The second case can happen only when $x$ is a rational number whose denominator is at most $n_{\max}$, so that the best rational approximation of it is $x$ itself. However, for such a case, if we let $x=\frac{p}{q}$, then according to how we derived [**Theorem 2**](#floor-computation), the remainder of $n_{0}p/q$ must be the largest possible value, $q-1$. Hence, the quotient of $(n_{0} - n)p/q$ cannot be strictly smaller than the difference between the quotients of $n_{0}p/q$ and $np/q$, so the claim holds in this case.

On the other hand, if we suppose that $\frac{\left\lfloor n_{0}x\right\rfloor + 1}{n_{0}}$ is a best rational approximation from above of $x$, then as we have seen in the case of lower bound, it must be a best rational approximation from above in the strong sense. Then it is not hard to see that $n_{0}$ is the minimizer of $\left\lceil nx\right\rceil - nx = \left\lfloor nx\right\rfloor + 1 - nx$, or equivalently, the maximizer of $nx - \left\lfloor nx\right\rfloor$, because we chose $n_{0}$ to be the smallest minimizer. This shows

$$
  (n_{0}-n)x - \left\lfloor(n_{0}-n)x\right\rfloor
  \leq n_{0}x - \left\lfloor n_{0}x\right\rfloor,
$$

thus

$$
  \left\lfloor(n_{0}-n)x\right\rfloor
  \geq \left\lfloor n_{0}x\right\rfloor - nx
  > \left\lfloor n_{0}x\right\rfloor - \left\lfloor nx\right\rfloor - 1,
$$

so we must have $\left\lfloor(n_{0}-n)x\right\rfloor = \left\lfloor n_{0}x\right\rfloor - \left\lfloor nx\right\rfloor$ as claimed.

Using the claim, we can again characterize $n=1,\ \cdots\ ,n_{0}-1$ which yields a smaller value of

$$
  \frac{\left\lfloor (n_{0}-n)x \right\rfloor + 1 - \zeta}{n_{0}-n}
$$

than the case $n=n_{0}$. Indeed, we have the inequality

$$
  \frac{\left\lfloor (n_{0}-n)x \right\rfloor + 1 - \zeta}{n_{0}-n}
  \leq \frac{\left\lfloor n_{0}x \right\rfloor + 1 - \zeta}{n_{0}}
$$

if and only if

$$
  -n_{0}\left\lfloor nx\right\rfloor + n_{0}(1-\zeta)
  \leq -n\left\lfloor n_{0}x\right\rfloor + (n_{0}-n)(1-\zeta)
$$

if and only if

$$
  n_{0}\left\lfloor nx\right\rfloor \geq
  n\left\lfloor n_{0}x\right\rfloor + n(1-\zeta)
$$

if and only if

$$
  \frac{\left\lfloor nx\right\rfloor}{n}
  \geq \frac{\left\lfloor n_{0}x\right\rfloor + 1 - \zeta}{n_{0}},
$$

or equivalently,

$$\label{eq:upper bound iteration criterion}
  x - \frac{\left\lfloor nx\right\rfloor}{n}
  \leq \frac{\zeta}{n_{0}}
  - \left(\frac{\left\lfloor n_{0}x\right\rfloor + 1}{n_{0}} - x\right).
$$

Next, let $n_{1}$ be the *largest maximizer* of $\frac{\left\lfloor nx\right\rfloor}{n}$ for $n=1,\ \cdots\ ,n_{0}-1$. If $\eqref{eq:upper bound iteration criterion}$ does not hold with $n=n_{1}$, then we know that $n_{0}$ is the one we are looking for, so suppose the otherwise.

Next, we claim that

$$
  \frac{\left\lfloor (n_{0}-n)x\right\rfloor + 1 - \zeta}{n_{0}-n}
  \geq \frac{\left\lfloor (n_{0}-n_{1})x\right\rfloor + 1 - \zeta}{n_{0}-n_{1}}
$$

holds for all $n=1,\ \cdots\ ,n_{1}$. By $\eqref{eq:floor splits; upper bound}$, we can rewrite the inequaltiy above as

$$\begin{aligned}
  &-n_{0}\left\lfloor nx\right\rfloor - n_{1}\left\lfloor n_{0}x\right\rfloor
  + n_{1}\left\lfloor nx\right\rfloor + (n_{0}-n_{1})(1-\zeta) \\
  &\quad\quad\quad \geq
  -n_{0}\left\lfloor n_{1}x\right\rfloor - n\left\lfloor n_{0}x\right\rfloor
  + n\left\lfloor n_{1}x\right\rfloor + (n_{0}-n)(1-\zeta),
\end{aligned}$$

or

$$
  (n_{1}-n)\left(\left\lfloor n_{0}x\right\rfloor + 1 - \zeta\right)
  \leq \left(
    n_{1}\left\lfloor nx\right\rfloor - n\left\lfloor n_{1}x\right\rfloor
  \right)
  - n_{0}\left(
    \left\lfloor nx\right\rfloor - \left\lfloor n_{1}x\right\rfloor
  \right).
$$

Then adding and subtracting appropriate terms gives

$$\begin{aligned}
  &n_{0}(n_{1}-n)\left(
    \frac{\zeta}{n_{0}}
    - \left(\frac{\left\lfloor n_{0}x\right\rfloor + 1}{n_{0}} - x\right)
  \right) \\
  &\quad\quad\quad\geq
  n_{1}(n_{0}-n)\left(
    x - \frac{\left\lfloor n_{1}x\right\rfloor}{n_{1}}
  \right)
  - n(n_{0}-n_{1})\left(
    x - \frac{\left\lfloor nx\right\rfloor}{n}
  \right).
\end{aligned}$$

Recall that we already supposed

$$
  x - \frac{\left\lfloor n_{1}x\right\rfloor}{n_{1}}
  \leq \frac{\zeta}{n_{0}}
  - \left(\frac{\left\lfloor n_{0}x\right\rfloor + 1}{n_{0}} - x\right),
$$

so it is enough to show

$$\begin{aligned}
  &n_{0}(n_{1}-n)\left(
    x - \frac{\left\lfloor n_{1}x\right\rfloor}{n_{1}}
  \right) \\
  &\quad\quad\quad\geq
  n_{1}(n_{0}-n)\left(
    x - \frac{\left\lfloor n_{1}x\right\rfloor}{n_{1}}
  \right)
  - n(n_{0}-n_{1})\left(
    x - \frac{\left\lfloor nx\right\rfloor}{n}
  \right),
\end{aligned}$$

or equivalently,

$$\begin{aligned}
  n(n_{0}-n_{1})\left(x - \frac{\left\lfloor nx\right\rfloor}{n}\right)
  &\geq \left(n_{1}(n_{0} - n) - n_{0}(n_{1}-n)\right)
  \left(x - \frac{\left\lfloor n_{1}x\right\rfloor}{n_{1}}\right) \\
  &= n(n_{0} - n_{1})
  \left(x - \frac{\left\lfloor n_{1}x\right\rfloor}{n_{1}}\right),
\end{aligned}$$

which trivially holds by the definition of $n_{1}$. Thus, the claim is proved.

As a result, we can reformulate our optimization problem in the following way: define $N_{1}=n_{0}-n_{1}$, then we are to find $n=0,\ \cdots\ ,N_{1}-1$ which minimizes

$$
  \frac{\left\lfloor (N_{1}-n)x\right\rfloor + 1 - \zeta}{N_{1}-n}.
$$

Again, the next step is to show that

$$
  \left\lfloor (N_{1}-n)x\right\rfloor
  = \left\lfloor N_{1}x\right\rfloor - \left\lfloor nx\right\rfloor
$$

holds for all such $n$, which by $\eqref{eq:floor splits; upper bound}$, is equivalent to

$$
  \left\lfloor (n_{1}+n)x\right\rfloor
  = \left\lfloor n_{1}x\right\rfloor + \left\lfloor nx\right\rfloor.
$$

But we already have seen this in the case of lower bound, becuase we know $n_{1}$ is a maximizer of $\frac{\left\lfloor nx\right\rfloor}{n}$ for $n = 1,\ \cdots\ ,n_{0} - 1$. Hence, again we can show that

$$
  \frac{\left\lfloor (N_{1}-n)x\right\rfloor + 1 - \zeta}{N_{1}-n}
  \leq \frac{\left\lfloor N_{1}x\right\rfloor + 1 - \zeta}{N_{1}}
$$

holds if and only if

$$
  \frac{\left\lfloor nx\right\rfloor}{n}
  \geq \frac{\left\lfloor N_{1}x\right\rfloor + 1 - \zeta}{N_{1}},
$$

and repeating this procedure gives us the smallest minimizer of $\frac{\left\lfloor nx\right\rfloor + 1 - \zeta}{n}$.

><b id="upper-bound-algorithm">Algorithm 6</b> (Computing the upper bound)**.**
>
>Input: $x\in\mathbb{R}$, $n_{\max}\in\mathbb{Z}_{>0}$, $\zeta\geq 0$.
>
>Output: the smallest minimizer of $\frac{\left\lfloor nx\right\rfloor + 1 - \zeta}{n}$ for $n=1,\ \cdots\ ,n_{\max}$.
>
>1. Find the smallest $n=1,\ \cdots\ ,n_{\max}$ that minimizes $\frac{\left\lfloor nx\right\rfloor + 1}{n}$ and call it $n_{0}$.
>2. If $n_{0} = 1$, then $n_{0}$ is the smallest minimizer; return.
>3. Find the largest $n=1,\ \cdots\ ,n_{0} - 1$ that maximizes $\frac{\left\lfloor nx\right\rfloor}{n}$ and call it $n_{1}$.
>4. Inspect the inequality
>\\[
>  \frac{\left\lfloor n_{1}x\right\rfloor}{n_{1}}
>  \geq \frac{\left\lfloor n_{0}x\right\rfloor + 1 - \zeta}{n_{0}}.
>\\]
>5. If the inequality does not hold, then $n_{0}$ is the smallest minimizer; return.
>6. If the inequality does hold, then set $n_{0}\leftarrow n_{0} - n_{1}$ and go to Step 2.

*Remark*. Similarly to the case of lower bound, we blackboxed the operation of finding the smallest minimizer of $\frac{\left\lfloor nx\right\rfloor}{n}$, which again is precisely how we obtain [**Theorem 2**](#floor-computation). If $x=\frac{p}{q}$ is a rational number with $q\leq n_{\max}$, then the minimizer is unique and is the largest $v\leq n_{\max}$ such that $vp\equiv -1\ (\mathrm{mod}\ q)$. Otherwise, we just compute the best rational approximation from above of $x$ with the largest denominator $$q_{*}\leq n_{\max}$$, where the theory of continued fractions allows us to compute this very efficiently. The resulting $$\frac{p_{*}}{q_{*}}$$ must be in its reduced form, so $$q_{*}$$ is the smallest minimizer of $\frac{\left\lfloor nx\right\rfloor + 1}{n}$.

## Finding feasible values of $\xi$ and $\zeta$

Note that our purpose of introducing $\zeta$ was to increase the gap between the lower and the upper bounds of $\xi=\frac{m}{2^{k}}$ when the required bit-width of the constant $m$ is too large. Thus, $\zeta$ is not something given, rather is something we want to figure out. In this sense, [**Algorithm 5**](#lower-bound-algorithm) and [**Algorithm 6**](#upper-bound-algorithm) might seem pretty useless because they work only when the value of $\zeta$ is already given.

Nevertheless, the way those algorithms work for a fixed $\zeta$ is in fact quite special in that it allows us to figure out a good $\zeta$ fitting into our purpose of widening the gap between two bounds. The important point here is that, *$\zeta$ only appears in deciding when to stop the iteration, and all other details of the actual iteration steps do not depend on $\zeta$ at all*.

Hence, our strategy is as follows. First, we assume $\zeta$ lives in the interval $[0,1)$ (otherwise $\left\lfloor nx\right\rfloor = \left\lfloor n\xi+\zeta\right\rfloor$ fails to hold for $n=0$). Then we can partition the interval $[0,1)$ into subintervals of the form $[\zeta_{\min},\zeta_{\max})$ where the numbers of iterations that [**Algorithm 5**](#lower-bound-algorithm) and [**Algorithm 6**](#upper-bound-algorithm) will take remain constant. Then we look at each of these subintervals one by one, from left to right, while proceeding the iterations of [**Algorithm 5**](#lower-bound-algorithm) and [**Algorithm 6**](#upper-bound-algorithm) whenever we move into the next subinterval and the numbers of iterations for each of them change.

Our constraint on $\xi$ and $\zeta$ is that the computation of $mn+s$ should not overflow, so suppose that there is a given limit $N_{\max}$ on how large $mn+s$ can be. Now, take a subinterval $[\zeta_{\min},\zeta_{\max})$ as described above. If there is at least one feasible choice of $\xi$ for some $\zeta\in[\zeta_{\min},\zeta_{\max})$, then such $\xi$ must lie in the interval

$$
  I:= \left(\frac{\left\lfloor n_{L,0}x\right\rfloor - \zeta_{\max}}{n_{L,0}}, \frac{\left\lfloor n_{U,0}x\right \rfloor + 1 - \zeta_{\min}}{n_{U,0}}\right),
$$

where $n_{L,0}$ and $n_{U,0}$ mean $n_{0}$ appearing in [**Algorithm 5**](#lower-bound-algorithm) and [**Algorithm 6**](#upper-bound-algorithm), respectively.

Next, we will take a loop over all candidates for $\xi$ in $I$ and check one by one if a chosen candidate is really a possible choice of $\xi$. Let $\Delta$ be the size of the interval above, that is,

$$
  \Delta := \frac{\left\lfloor n_{U,0}x\right \rfloor + 1 - \zeta_{\min}}{n_{U,0}}
  - \frac{\left\lfloor n_{L,0}x\right\rfloor - \zeta_{\max}}{n_{L,0}}.
$$

Define $k_{0}:=\left\lfloor\log_{2}\frac{1}{\Delta}\right\rfloor$, then we have

$$
  2^{k_{0}}\Delta \leq 1 < 2^{k_{0}+1}\Delta < 2,
$$

and this means that there is at most one integer in the interval $2^{k_{0}}I$ while there is at least one integer in the interval $2^{k_{0}+1}I$. Therefore, we first see if there is one in $2^{k_{0}}\Delta$, and if not, then have a look at $2^{k_{0}+1}\Delta$ where the existence is guaranteed. Also, note that if there were none in $2^{k_{0}}\Delta$, then $2^{k_{0}+1}\Delta$ should have exactly one integer and that integer must be odd.

Either case, we can find the smallest possible $t$ such that

$$
  \xi = \frac{t}{2^{k_{0}' - b}}
$$

is in the interval $I$ for some $b\geq 0$, where $k_{0}'$ is either $k_{0}$ or $k_{0}+1$ and $t$ is an odd number. Now, in order for this $\xi$ to be allowed by a choice of $\zeta$, according to [**Algorithm 5**](#lower-bound-algorithm), $\zeta$ should be at least

$$
  \zeta_{0} := \left\lfloor n_{L,0}x \right\rfloor - n_{L,0}\xi
  = \frac{2^{k_{0}'-b}\left\lfloor n_{L,0}x\right\rfloor - n_{L,0}t}
  {2^{k_{0}-b}}.
$$

Since we want our $\xi$ to satisfy the upper bound given by [**Algorithm 6**](#upper-bound-algorithm) as well, we want to choose $\zeta$ to be as small as possible as long as $\zeta\geq\zeta_{0}$ is satisfied. Hence, we want to take $\zeta=\zeta_{0}$, but this is not always possible because $\zeta_{0}\geq\zeta_{\min}$ is not always true. (The above $\zeta_{0}$ even can be negative, so in the actual algorithm we may truncate it to be nonnegative.)

If $\zeta_{0}\geq\zeta_{\min}$ is satisfied, then we may indeed take $\zeta=\zeta_{0}$. By the construction, $\zeta_{0}<\zeta_{\max}$ is automatically satisfied, so we want to check two things: (1) the smallness constraint $mn_{\max} + s \leq N_{\max}$ with $m=t$ and $s=2^{k_{0}'-b}\zeta_{0}$, and (2) the true upper bound

$$
  \xi < \frac{\left\lfloor n_{U,0}x\right \rfloor + 1 - \zeta_{0}}{n_{U,0}}
$$

given by [**Algorithm 6**](#upper-bound-algorithm). If both of the conditions are satisfied, then $k=k_{0}'-b$, $m=t$, and $s=2^{k_{0}'-b}\zeta_{0}$ are valid solutions. Otherwise, we conclude that $\xi$ does not yield an admisslbe choice of $(k,m,s)$.

If $\zeta_{0}<\zeta_{\min}$, then we have to increase the numerator of $\zeta_{0}$ to put in inside $[\zeta_{\min},\zeta_{\max})$, so let $\zeta$ be the one obtained by adding the smallest positive integer to the numerator of $\zeta_{0}$ which ensures $\zeta\geq\zeta_{\min}$. Now, since we cannot easily estimate how large $\zeta$ is compared to $\zeta_{0}$, we have to check $\zeta<\zeta_{\max}$ in addition to the other two conditions. Actually, the difference $\zeta-\zeta_{0}$ can be made small by considering a larger denominator of $\xi$ and $\zeta_{0}$, so we have to successively double the denominator and see there is a case where $\zeta$ satisfies all three conditions. Of course, if we enlarged the denominator too much, then the smallness constraint will get broken no matter how small $\zeta-\zeta_{0}$ is, so at that point we stop doing this doubling and conclude that $\xi$ does not yield an admissible choice of $(k,m,s)$.

If we failed to find any admissible choice of $(k,m,s)$ with our $\xi$, then we increase $k_{0}'$ by one and repeat the same procedure with the integers in the interval $2^{k_{0}'}\Delta$. Note that this time we do not need to consider even integers because those were already checked for smaller $k_{0}'$.

This loop over $k_{0}'$ should be stopped when we arrive at a point where the smallest odd integer $t$ in $2^{k_{0}'}\Delta$ already fails to satisfy

$$
  tn_{\max} + 2^{k_{0}'}\zeta_{0} \leq N_{\max}.
$$

If we exhausted all $k_{0}'$'s satisfying the above, then now we conclude that there is no $\zeta\in[\zeta_{\min},\zeta_{\max})$ yielding an admissible choice of $(k,m,s)$, so we should move onto the next subinterval.

After filling out some omitted details, we arrive at the following algorithm.

><b id="xi-zeta-finding-algorithm">Algorithm 7</b> (Finding feasible values of $\xi$ and $\zeta$)**.**
>
>Input: $x\in\mathbb{R}$, $$n_{\max}\in\mathbb{Z}_{>0}$$, $$N_{\max}\in\mathbb{Z}_{>0}$$.
>
>Output: $k$, $m$, and $s$, where $\xi = \frac{m}{2^{k}}$ and $\zeta = \frac{s}{2^{k}}$, so that we have $\left\lfloor nx\right\rfloor = \left\lfloor \frac{nm + s}{2^{k}}\right\rfloor$ for all $n=1,\ \cdots\ ,n_{\max}$.
>
>1. Find the largest $n=1,\ \cdots\ ,n_{\max}$ that maximizes $\frac{\left\lfloor nx\right\rfloor}{n}$ and call it $n_{L,0}$.
>2. Find the smallest $n=1,\ \cdots\ ,n_{\max}$ that minimizes $\frac{\left\lfloor nx\right\rfloor + 1}{n}$ and call it $n_{U,0}$.
>3. Set $\zeta_{\max} \leftarrow 0$, $\zeta_{L,\max} \leftarrow 0$, $\zeta_{U,\max} \leftarrow 0$, $n_{L,1} \leftarrow 0$, $n_{U,1} \leftarrow 0$.
>4. Check if $\zeta_{\max} = \zeta_{L,\max}$. If that is the case, then we have to update $\zeta_{L,\max}$. Set $n_{L,0}\leftarrow n_{L,0}+n_{L,1}$. If $n_{L,0}=n_{\max}$, then set $\zeta_{L,\max}\leftarrow 1$. Otherwise, find the largest $n=1,\ \cdots\ ,n_{\max}-n_{L,0}$ that maximizes $\frac{\left\lfloor nx\right\rfloor}{n}$, assign it to $n_{L,1}$, and set \\[ \zeta_{L,\max}\leftarrow \min\left(\frac{n_{L,1}\left\lfloor n_{L,0}x\right\rfloor - n_{L,0}\left\lfloor n_{L,1}x\right\rfloor}{n_{L,1}}, 1\right). \\]
>5. Check if $\zeta_{\max} = \zeta_{U,\max}$. If that is the case, then we have to update $\zeta_{U,\max}$. Set $n_{U,0}\leftarrow n_{U,0}-n_{U,1}$. If $n_{U,0}=1$, then set $\zeta_{U,\max}\leftarrow 1$. Otherwise, find the largest $n=1,\ \cdots\ ,n_{U,0}-1$ that maximizes $\frac{\left\lfloor nx\right\rfloor}{n}$, assign it to $n_{U,1}$, and set \\[ \zeta_{U,\max}\leftarrow \min\left(\frac{n_{U,1}\left(\left\lfloor n_{U,0}x\right\rfloor + 1\right) - n_{U,0}\left\lfloor n_{U,1}x\right\rfloor}{n_{U,1}}, 1\right). \\]
>6. Set $\zeta_{\min}\leftarrow \zeta_{\max}$, $\zeta_{\max} \leftarrow \min\left(\zeta_{L,\max},\zeta_{U,\max}\right)$, and \\[ \Delta \leftarrow \frac{\left\lfloor n_{U,0}x\right \rfloor + 1 - \zeta_{\min}}{n_{U,0}} - \frac{\left\lfloor n_{L,0}x\right\rfloor - \zeta_{\max}}{n_{L,0}}. \\] If $\Delta\leq 0$, then this means the search interval $I$ is empty. In this case, go to Step 15. Otherwise, proceed to the next step.
>7. Set $k_{0}\leftarrow \left\lfloor\log_{2}\frac{1}{\Delta}\right\rfloor$ and compute
>\\[
>  t \leftarrow \left\lfloor
>    \frac{2^{k_{0}}\left(\left\lfloor n_{L,0}x\right\rfloor - \zeta_{\max}\right)}
>    {n_{L,0}}
>  \right\rfloor + 1.
>\\]
>8. Inspect the inequality
>\\[
>  t < \frac{2^{k_{0}}\left(
>    \left\lfloor n_{U,0}x\right \rfloor + 1 - \zeta_{\min}
>  \right)}{n_{U,0}}.
>\\]
>    If it does not hold, then set $k_{0}\leftarrow k_{0} + 1$, recompute $t$ accordingly, and set $b\leftarrow 0$. If it does hold, then set $b$ be the greatest integer such that $2^{b}$ divides $t$, and set $t\leftarrow t/2^{b}$.
>9. Set
>\\[
>  \zeta_{0} \leftarrow \max\left(
>    \frac{2^{k_{0}-b}\left\lfloor n_{L,0}x\right\rfloor - n_{L,0}t}{2^{k_{0}-b}},
>  0\right)
>\\]
>    and inspect the inequality $tn_{\max} + 2^{k_{0}-b}\zeta_{0}\leq N_{\max}$. If it does not hold, then this means every candidate we can find in the interval $[\zeta_{\min},\zeta_{\max})$ are too large numbers, so we move onto the next subinterval. In this case, go to Step 18. Otherwise, proceed to the next step.
>10. If $\zeta_{0}\geq\zeta_{\min}$, then $\zeta_{0}\in[\zeta_{\min},\zeta_{\max})$ and $tn_{\max} + 2^{k_{0}-b}\zeta_{0} \leq N_{\max}$ hold, so it is enough to check the inequality
>\\[
>  \frac{t}{2^{k_{0}-b}}<\frac{\left\lfloor n_{U,0}x\right \rfloor + 1 - \zeta_{0}}
>  {n_{U,0}}.
>\\]
>If this holds, then we have found an admissible choice of $(k,m,s)$, so return. Otherwise, we conclude that $\xi=\frac{t}{2^{k_{0}-b}}$ does not yield an admissible answer. In this case, go to Step 16.
>11. If $\zeta_{0}<\zeta_{\min}$, then set $k\leftarrow k_{0}-b$.
>12. Set $m\leftarrow 2^{k-k_{0}+b}$. Set $a\leftarrow \left\lceil 2^{k}(\zeta_{\min} - \zeta_{0})\right\rceil$ and $\zeta\leftarrow \zeta_{0} + \frac{a}{2^{k}}$ so that $\zeta\geq\zeta_{\min}$ is satisfied. Check if $\zeta<\zeta_{\max}$ holds, and if that is not the case, then set $k\leftarrow k+1$ and go to Step 15.
>13. Set $m\leftarrow 2^{k-k_{0}+b}t$ and check if $mn_{\max} + 2^{k}\zeta \leq N_{\max}$ holds. If that is not the case, then set $k\leftarrow k+1$ and go to Step 15.
>14. Inspect the inequality
>\\[
>  \frac{m}{2^{k}}<\frac{\left\lfloor n_{U,0}x\right \rfloor + 1 - \zeta}{n_{U,0}}.
>\\]
>    If it does hold, then we have found an admissible choice of $(k,m,s)$, so return. Otherwise, set $k\leftarrow k+1$ and proceed to the next step.
>15. Check if $mn_{\max}+2^{k}\zeta_{0} \leq N_{\max}$ holds. If it does hold, then go back to Step 12. Otherwise, proceed to the next step.
>16. Set $k_{0}\leftarrow k_{0}+1$, $b\leftarrow 0$, and compute
>\\[
>  t \leftarrow \left\lfloor
>    \frac{2^{k_{0}}\left(\left\lfloor n_{L,0}x\right\rfloor - \zeta_{\max}\right)}
>    {n_{L,0}}
>  \right\rfloor + 1,\quad
>  \zeta_{0}\leftarrow \max\left(
>    \frac{2^{k_{0}}\left\lfloor n_{L,0}x\right\rfloor - n_{L,0}t}{2^{k_{0}}},
>  0\right).
>\\]
>17. If $t$ is even, then set $t\leftarrow t+1$. Inspect the inequality $tn_{\max} + 2^{k_{0}}\zeta_{0}\leq N_{\max}$. If this does not hold, then this means every other candidate in the interval $[\zeta_{\min},\zeta_{\max})$ we have not seen yet are too large numbers, so we move onto the next subinterval. In this case, go to Step 18. Otherwise, we repeat the procedure Step 10 - Step 15 for $t\leftarrow t+2\tau$ for all nonnegative integer $\tau$ such that $t+2\tau<\frac{2^{k_{0}}\left(\left\lfloor n_{U,0}x\right \rfloor + 1 - \zeta_{\min}\right)}{n_{U,0}}$ (note that $\tau=0$ might still violate the inequality), and if it fails for all such $\tau$, then go to Step 16. 
>18. We have failed to find any admissible choice of $(k,m,s)$ so far, so we have to move onto the next subinterval. If $\zeta_{\max}=1$, then we already have exhausted the whole interval, so retrun with **FAIL**. Otherwise, go back to Step 4.

**Remarks.**

1. Whenever we update $\zeta_{L,\max}$ and $\zeta_{U,\max}$, the updated values must be always strictly bigger than the previous values unless they already have reached their highest value $1$. Let us see the case of $\zeta_{L,\max}$; the case of $\zeta_{U,\max}$ is similar. Suppose that initially we had $n_{L,0} = n_{0}$, $n_{L,1} = n_{1}$, and after recomputing $n_{L,1}$ we got $n_{L,1} = n_{2}$. Then the new value of $\zeta_{L,\max}$ and the old value of it are respectively given as \\[\label{eq:gap between successive zeta max} \frac{n_{2}\left\lfloor(n_{0}+n_{1})x\right\rfloor - (n_{0}+n_{1})\left\lfloor n_{2}x\right\rfloor}{n_{2}},\quad \frac{n_{1}\left\lfloor n_{0}x\right\rfloor - n_{0}\left\lfloor n_{1}x\right\rfloor}{n_{1}},\\] respectively. Applying $\eqref{eq:floor splits; lower bound}$, it turns out that the first one minus the second one is equal to \\[ (n_{0}+n_{1})\left(\frac{\left\lfloor n_{1}x\right\rfloor}{n_{1}} - \frac{\left\lfloor n_{2}x\right\rfloor}{n_{2}} \right).\\] Now, recall the way $n_{1}$ is chosen: we first find the best rational approximation of $x$ from below in the range $$\{1,\ \cdots\ ,n_{\max}-n_{0}\}$$, call its denominator $$q_{*}$$, and set $n_{1}$ to be the largest multiple of $$q_{*}$$. Since $n_{1}$ is the largest multiple, it follows that $n_{\max} - n_{0} - n_{1}$ should be strictly smaller than $$q_{*}$$. Therefore, the best rational approximation in the range $$\{1,\ \cdots\ ,n_{\max}-n_{0}-n_{1}\}$$ should be strictly worse than what $$q_{*}$$ gives. This shows that $\eqref{eq:gap between successive zeta max}$ is strictly positive.
  
2. When there indeed exists at least one admissible choice of $(k,m,s)$ given $x$, $n_{\max}$, and $N_{\max}$, [**Algorithm 7**](#xi-zeta-finding-algorithm) favors the one with the smallest $k$, and among all admissible choices of $(k,m,s)$ with the smallest $k$, it favors the one with the smallest $s$.

An actual implementation of this algorithm can be found [here](https://github.com/jk-jeon/idiv/blob/main/include/idiv/idiv.h#L88).

## The result by Lemire et al.

Consider the special case when $x=\frac{1}{q}$, $q\leq n_{\max}$, and $\xi=\zeta$. In this case, we have the following result:

>**Theorem 8 ([Lemire et al.](https://doi.org/10.1016/j.heliyon.2021.e07442), 2021).**
>
>We have
>
>$$
>  \left\lfloor\frac{n}{q}\right\rfloor
>  = \left\lfloor (n+1)\xi\right\rfloor
>$$
>
>for all $n=0,\ \cdots\ ,n_{\max}$ if and only if
>
>$$
>  \left(1 - \frac{1}{\left\lfloor n_{\max}/q\right\rfloor q+1}\right)\frac{1}{q}
>  \leq \xi < \frac{1}{q}.
>$$

We can give an alternative proof of this fact using what we have developed so far. Essentially, this is due to the fact that [**Algorithm 5**](#lower-bound-algorithm) finishes its iteration at the first step and do not proceed further.

>Proof. $(\Rightarrow)$ Take $n=q-1$, then we have $0 = \left\lfloor q\xi\right\rfloor$, thus $\xi<\frac{1}{q}$ follows. On the other hand, take $n = \left\lfloor\frac{n_{\max}}{q}\right\rfloor q$, then we have
>
>$$
>  \left\lfloor\frac{n_{\max}}{q}\right\rfloor
>  \leq \left(\left\lfloor\frac{n_{\max}}{q}\right\rfloor q + 1\right)\xi,
>$$
>
>so rearranging this gives the desired lower bound on $\xi$.
>
> $(\Leftarrow)$ It is enough to show that $\xi$ satisfies
>
>$$
>  \max_{n=1,\ \cdots\ ,n_{\max}} \frac{\left\lfloor n/q\right\rfloor - \xi}{n}
>  \leq \xi <
>  \min_{n=1,\ \cdots\ ,n_{\max}} \frac{\left\lfloor n/q\right\rfloor + 1 - \xi}{n}.
>$$
>
>Following [**Algorithm 5**](#lower-bound-algorithm), let $n_{0}$ be the largest maximizer of $\frac{\left\lfloor n/q\right\rfloor}{n}$, i.e., $n_{0} = \left\lfloor\frac{n_{\max}}{q}\right\rfloor q$. Then we know from [**Algorithm 5**](#lower-bound-algorithm) that $n_{0}$ is the largest maximizer of $\frac{\left\lfloor n/q\right\rfloor - \xi}{n}$ if and only if
>
>$$
>  \frac{\left\lfloor n/q\right\rfloor}{n} <
>  \frac{\left\lfloor n_{0}/q\right\rfloor - \xi}{n_{0}}
>  = \frac{1}{q} - \frac{\xi}{\left\lfloor n_{\max}/q\right\rfloor q}
>$$
>
>holds for all $n=1,\ \cdots\ ,n_{\max}-n_{0}$. Pick any such $n$ and let $r$ be the remainder of $n/q$, then we have
>
>$$
>  \frac{\left\lfloor n/q\right\rfloor}{n}
>  = \frac{(n-r)/q}{n} = \frac{1}{q} - \frac{r}{nq}.
>$$
>
>Hence, we claim that
>
>$$
>  \frac{\xi}{\left\lfloor n_{\max}/q\right\rfloor} < \frac{r}{n}
>$$
>
>holds for all such $n$. This is clear, because $\xi<\frac{1}{q}$ while $n_{\max} - n_{0}$ is at most $q-1$, so the right-hand side is at least $\frac{1}{q-1}$. Therefore, $n_{0}$ is indeed the largest maximizer of $\frac{\left\lfloor n/q\right\rfloor - \xi}{n}$ for $n=1,\ \cdots\ ,n_{\max}$, and the inequality
>
>$$
>  \frac{\left\lfloor n_{0}/q\right\rfloor - \xi}{n_{0}} \leq \xi
>$$
>
>is equivalent to
>
>$$
>  \xi \geq \frac{\left\lfloor n_{0}/q\right\rfloor}{n_{0}+1}
>  = \frac{1}{q}\left(1 - \frac{1}{n_{0}+1}\right),
>$$
>
>which we already know.
>
>For the upper bound, note that it is enough to show that
>
>$$
>  \frac{1}{q} \leq \frac{\left\lfloor n/q\right\rfloor + 1}{n+1}
>$$
>
>holds for all $n=1,\ \cdots\ ,n_{\max}$. We can rewrite this inequality as
>
>$$
>  \frac{n}{q} - \left\lfloor\frac{n}{q}\right\rfloor \leq \frac{q-1}{q},
>$$
>
>which means nothing but that the remainder of $n$ divided by $q$ is at most $q-1$, which is trivially true. $\quad\blacksquare$

Note the fact that the numerator of $x$ is $1$ is crucially used in this proof.

[Lemire et al.](https://doi.org/10.1016/j.heliyon.2021.e07442) also shows that, if $n_{\max} = 2^{N} - 1$ and $q$ is not a power of $2$, then whenever the best magic constant predicted by [**Theorem 2**](#floor-computation) does not fit into a word, the best magic constant predicted by the above theorem should fit into a word. Therefore, these two are enough when $n_{\max} = 2^{N} - 1$, $x=\frac{1}{q}$, and $q\leq n_{\max}$. Then does [**Algorithm 7**](#xi-zeta-finding-algorithm) have any relevance in practice?

## An example usage of [**Algorithm 7**](#xi-zeta-finding-algorithm)

I do not know, I mean, as I pointed out before, I am not sure how important in general is to cover the case $x=\frac{p}{q}$ with $p\neq 1$. But anyway when $p\neq 1$, there certainly are some cases where [**Algorithm 7**](#xi-zeta-finding-algorithm) might be useful. Consider the following example:
```cpp
std::uint32_t div(std::uint32_t n) {
    return (std::uint64_t(n) * 7) / 18;
}
```
Here I provide three different ways of doing this computation:
```asm
div1(unsigned int):
        mov     ecx, edi
        lea     rax, [8*rcx]
        sub     rax, rcx
        movabs  rcx, 1024819115206086201
        mul     rcx
        mov     rax, rdx
        ret
```
```asm
div2(unsigned int):
        mov     eax, edi
        movabs  rcx, 7173733806472429568
        mul     rcx
        mov     rax, rdx
        ret
```
```asm
div3(unsigned int):
        mov     ecx, edi
        mov     eax, 3340530119
        imul    rax, rcx
        add     rax, 477218588
        shr     rax, 33
        ret
```
([Check it out!](https://godbolt.org/z/hxj1c6b6M))

The first version (`div1`) is what my compiler (clang) generated. It first computes `n * 7` by computing `n * 8 - n`. And then, it computes the division by $18$ using the multiply-and-shift technique. Especially, it deliberately chose the shifting amount to be $64$ so that it can skip the shifting. Clang is indeed quite clever in the sense that it actually leverages the fact that `n` is only a $32$-bit integer, because, the minimum amount of shifting required for the division by $18$ is $68$, if all $64$-bit dividends are considered.

But it is not clever enough to realize that two operations `* 7` and `/ 18` can be merged. The second version `div2` is what one can get by utilizing this fact and do these two computations in one multiply-and-shift. Using [**Theorem 2**](#floor-computation), it can be shown that the minimum amount of shifting for doing this with all $32$-bit dividends is $36$, and in that case the corresponding magic number is $26724240953$. Since the computation cannot be done in $64$-bit anyway, I deliberately multiplied $2^{28}$ to this magic number and got $7173733806472429568$ to make the shifting amount equal to $64$, so that the actual shifting instruction is not needed. And the result is a clear win over `div1`, although `lea` and `sub` are just trivial instructions and the difference is not huge.

Now, if we do this with multiply-add-and-shift, we can indeed complete our computation inside $64$-bit, which is the third version `div3`. Using [**Algorithm 7**](#xi-zeta-finding-algorithm), it turns out $(k,m,s) = (33, 3340530119, 477218588)$ works for all $32$-bit dividend, i.e., we have

$$
  \left\lfloor \frac{7n}{18}\right\rfloor
  = \left\lfloor \frac{3340530119\cdot n + 477218588}{2^{33}}\right\rfloor
$$

for all $n=0,\ \cdots\ ,2^{32}-1$. The maximum possible value of the numerator $3340530119\cdot n + 477218588$ is strictly less than $2^{64}$ in this case. Hence, we get `div3`.

Note that `div3` has one more instructions than `div2`, but it does not invoke the $128$-bit multiplication (the `mul` instruction both found in `div1` and `div2`) which usually performs not so impressively on typical modern x86-64 CPU's, so I think probably `div3` will perform better than `div2` although I have not benchmarked ro confirm this guess.

