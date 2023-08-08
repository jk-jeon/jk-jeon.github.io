---
title: "On the Optimal Bounds for Integer Division by Constants"
date: 2023-08-07
permalink: /posts/2022/08/optimal-bounds-integer-division/
tags:
  - programming
  - C/C++
---

# !!! WIP !!!

It is well-known that the integer division is quite a heavy operation on modern CPU's - so slow, in fact, that it has even become a common wisdom to *avoid doing it at ALL cost* in performance-critical parts of the program. I do not know why division is particularly hard to optimize from the hardware perspective. I am just guessing, maybe that is because (1) every general algorithm is effectively just a minor variation of the good old long division, (2) which is almost impossible to parallelize in any way. But whatever, that is not the topic of this post.

Rather, this post is about some of the common optimization techniques for circumventing integer division. Also in a later section, I will present an optimal bound for one of such techniques, which seems to be not widely known yet. It would be interesting to see if modern compilers can take advantage of this new bound to better optimize the code.

Note that by *integer division*, we specifically mean computing the quotient and/or the remainder, not evaluating the result as a real number. More specifically, throughout this whole post, it will always mean taking the quotient unless specified otherwise. Also, I will confine myself into divisions of positive integers, though that of negative integers is also of practical importance. Finally, all assemblies shown here are for x86-64 architecture.

# Turning an integer division into a multiply-and-shift

One of the most widely used techniques is converting division into a multiplication if the divisor is a constant (or it remains mostly unchanged). The idea is quite simple: for example, dividing by $4$ is equivalent to multiplying by $0.25$, which can be further represented as multiplying by $25$ and then dividing by $100$, where dividing by $100$ is simply a matter of moving the decimal dot into left by two positions. Since we are only interested in taking the quotient, this means throwing away the last two digits.

Of course, in this specific case, $4$ is a divisor of $100$ so we indeed have a fairly short such a representation, but in general the divisor might not divide a power of $10$. Let us take $7$ as an example. In this case, we cannot write $\frac{1}{7}$ as $\frac{m}{10^{k}}$ for some positive integers $m,k$. However, we can still come up with a good *approximation*. Note that

$$
  \frac{1}{7} = 0.142857142857\cdots,
$$

so presumably something like $\frac{142858}{1000000}$ would be a good enough approximation of $\frac{1}{7}$. Taking that as our approximation means that we may want to compute $n/7$ by multiplying $142858$ to $n$ and then throwing away the last $6$ digits. (Note that we are taking $142858$ instead of $142857$ because the latter already fails when $n=7$. In general, we must do ceiling, not floor nor half-up rounding.)

This indeed gives the right answer for all $n=1,\ \cdots\ ,166668$, but it starts to produce a wrong answer when $n=166669$; in this case the correct answer is $23809$ but our method returns $23810$. And of course such a failure is expected. We are using an approximation with a nonzero error, and surely the error will eventually show up when the dividend is large enough. But the question is, *can we estimate how far it will go?* Or, *can we choose a good enough approximation when there is a limit on how big our $n$ can be?*

Obviously, we are gonna apply this idea for computers, so the denominators of the approximations will be powers of $2$, not $10$: just like that for humans dividing by $10^{k}$ is just throwing away $k$-digits, for computers, dividing by $2^{k}$ is just the right-shift by $k$-bits. So, for a given positive integer $d$, our goal is to find a good enough approximation of $\frac{1}{d}$ of the form $\frac{m}{2^{k}}$. Probably the most widely known formal result regarding this is the following theorem by Granlund-Montgomery:

>**Theorem 1 (Granlund-Montgomery, 1994).**
>
> Suppose $m$, $d$, $k$ are nonnegative integers such that $d\neq 0$ 
>
> $$
>   2^{N+k} \leq md \leq 2^{N+k}+2^{k}.
> $$
>
> Then $\left\lfloor n/d \right\rfloor = \left\lfloor mn/2^{N+k} \right\rfloor$ for every integer $n$ with $0\leq n< 2^{N}$.

Here, $d$ is the given divisor and we are supposed to approximate $\frac{1}{d}$ by $\frac{m}{2^{N+k}}$. An assumption here is that we want to perform the division $n/d$ for all $n$ from $0$ to $2^{N}-1$, where $N$ is supposed to be the bit-width of the integer type we are dealing with. In this setting, this theorem gives a sufficient condition where we can compute the quotient of $n/d$ by first multiplying $m$ to $n$ and then shifting the result to the right by $(N+k)$-bits. A premise here is that we will need more than $N$-bits because our dividend is of $N$-bits, so maybe the result of the multiplication $mn$ will need to be stored in $2N$-bits. Since we are shifting by $(N+k)$-bits, the lower half of the result is actually not needed, and we just take the upper half and shift it by $k$-bits.

We will not talk about the proof of this theorem because a much more general theorem will be presented in the next section. But let us see how results like this is being applied in the wild. For example, consider the following C++ code:
```cpp
std::uint64_t div(std::uint64_t n) noexcept {
    return n / 17;
}
```

Compilers these days are well-aware of this Granlund-Montgomery-style division tricks. Indeed, my compiler (clang) leveraged such a trick and translated the above code into the following lines of assemblies:
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

Here is an example, take $N=32$ and $d=102807$. In this case, the smallest $k$ that allows an integer $m$ that satisfies

$$
  2^{N+k} \leq md \leq 2^{N+k} + 2^{k}
$$

to exist is $k=17$, and in that case the unique $m$ satisfying the above is $5475793997$. This is kind of unfortunate, because the magic constant $m=5475793997$ is of $33$-bits, so the computation of $nm$ cannot be done inside $64$-bits. However, it turns out that we can take $k=16$ and $m=2737896999$ and still the equality $\left\lfloor \frac{n}{d}\right\rfloor = \left\lfloor \frac{nm}{2^{N+k}}\right\rfloor$ holds for all $n=0,\ \cdots\ ,2^{N}-1$, although the above inequality is not satisfied in this case. Now, the new constant $2737896999$ is of $32$-bits, so we can do our computation inside $64$-bits. This might result a massive difference in practice!

It [seems](https://godbolt.org/z/b3jcs9vMK) that even the most recent version of GCC (13.2) is still not aware of this, while clang knows that the above $m$ and $k$ works. (What GCC does in the link provided will be explained in a [later section](#when-the-magic-number-is-too-big).)

Then what is the best possible condition? I am not sure who was the first for finding the optimal bound, but at least it seems to be written down in the famous book *Hacker's Delight* by H. S. Warren Jr. Also, recently (in 2021), [Lemire et al](https://doi.org/10.1016/j.heliyon.2021.e07442) showed the optimality of an equivalent bound. I will not write down the optimal bounds obtained by these authors, because I will present a more general result proved by myself in the next section.

Here is one remark before getting into the next section. The aforementioned results on the optimal bound work for $n$ from the range $$\left\{1,\ \cdots\ ,n_{\max}\right\}$$ where $n_{\max}$ is *not necessarily of the form* $2^{N}-1$. However, it seems that even recent compilers do not seem to leverage this fact. For example, let us look at the following code:
```cpp
std::uint64_t div(std::uint64_t n) {
    [[assume(n < 10000000000)]];
    return n / 10;
}
```

Here we are relying on C++23 attribute `assume`. Clang does not seem to be aware of this new language feature so is irrelevant in this discussion, and GCC generated the following lines of assemblies:
```asm
div(unsigned long):
        movabs  rax, -3689348814741910323
        mul     rdi
        mov     rax, rdx
        shr     rax, 3
        ret
```

([Check it out!](https://godbolt.org/z/Pr48e7Kja))

My gripe with this is that it generated a shift instruction `shr` which is in fact not necessary. The compiler thinks that $k$ must be at least $67$ (which is why it shifted by $3$-bits, after throwing away the $64$-bit lower half), but actually $k=64$ is fine with the magic number $m=1844674407370955162$ thanks to the bound on $n$, in which case we do not need this additional shifting.

# Turning multiplication by a real number into a multiply-and-shift

Actually, during the development of [Dragonbox](https://github.com/jk-jeon/dragonbox), I was interested in a more general problem of multiplying a rational number to $n$ and then finding out the integer part of the resulting rational number. In other words, my problem was not just about division, rather about multiplication followed by a division. This presence of multiplier certainly makes the situation a little bit more complicated, but it is anyway possible to derive the optimal bound in a similar way, which generalizes the results mentioned in the previous section. *(Disclaimer: I am definitely not claiming to be the first who proved this, and I am sure an equivalent result could be found elsewhere, though I am not aware of any.)*

>**Theorem 2 (From [this paper](https://github.com/jk-jeon/dragonbox/blob/master/other_files/Dragonbox.pdf)).**
>
>Let $x$ be a real number and $n_{\max}$ a positive integer. Then for a real number $\xi$, we have the followings.
>
>  1. If $x=\frac{p}{q}$ is a rational number with $q\leq n_{\max}$, then we have $\left\lfloor nx \right\rfloor = \left\lfloor n\xi \right\rfloor$ for all $n=1,\ \cdots\ ,n_{\max}$ if and only if $x \leq \xi < x + \frac{1}{vq}$  holds, where $v$ is the greatest integer such that $vp\equiv -1\ (\mathrm{mod}\ q)$ and $v\leq n_{\max}$.
>  
>  2. If $x$ is either irrational or a rational number with the denominator strictly greater than $n_{\max}$, then we have $\left\lfloor nx \right\rfloor = \left\lfloor n\xi \right\rfloor$ for all $n=1,\ \cdots\ ,n_{\max}$ if and only if $$\frac{p_{*}}{q_{*}} \leq \xi < \frac{p^{*}}{q^{*}}$$ holds, where $$\frac{p_{*}}{q_{*}}$$, $$\frac{p^{*}}{q^{*}}$$ are the best rational approximations of $x$ from below and above, respectively, with the largest denominators $$q_{*},q^{*}\leq n_{\max}$$.

Note that $\left\lfloor nx \right\rfloor$ is supposed to be the one we actually want to compute, while $\xi$ is supposed to be the chosen approximation of $x$. For the special case when $x = \frac{1}{d}$, $n_{\max} = 2^{N}-1$, and $\xi = \frac{m}{2^{N+k}}$, we obtain the setting of Granlund-Montgomery.

So I said rational number in the beginning of this section, but actually it does not matter if our $x$ is rational or irrational. But it does matter if $x$ is whether *"effectively rational"* or not over the domain $n=1,\ \cdots\ ,n_{\max}$, i.e., if there exists or not a multiplier $n$ which makes $nx$ into an integer.

Since I was working on floating-point conversion problems when I derived this theorem, for me the more relevant case was the second case, that is, when $x$ is "effectively irrational", because the numerator and the denominator of $x$ I was considering were some high powers of $2$ and $5$. But the first case is more relevant in the main theme of this post, i.e., integer division, so let us forget about these jargons like *best rational approximations* and such. (**Spoiler**: they will show up again in the next section!)

So let us focus on the first case. First of all, note that if $p=1$, that is, when $x = \frac{1}{q}$, then $v$ has a simpler description: it is the last multiple of $q$ in the range $1,\ \cdots\ ,n_{\max}+1$ minus one. If you are curious enough, you can check the aforementioned paper by [Lemire et al](https://doi.org/10.1016/j.heliyon.2021.e07442) to see that their **Theorem 1** exactly says this. In fact, in this special case it is rather easy to see why the best bound should be something like that.

Indeed, note that having the equality

$$
  \left\lfloor nx \right\rfloor = \left\lfloor n\xi \right\rfloor
$$

for all $n=1,\ \cdots\ ,n_{\max}$ is equivalent to that the inequality

$$
  \frac{\left\lfloor nx\right\rfloor}{n} \leq \xi
  < \frac{\left\lfloor nx\right\rfloor + 1}{n}
$$

holds for all such $n$. Hence, it is enough to find the largest possible value of the left-hand side and the smallest possible value of the right-hand side. Since we are assuming that the denominator of $x$ is bounded by $n_{\max}$, obviously the maximum value of the left-hand side is just $x$. Thus, it is enough to find the minimum value of the right-hand side. Note that we can write

$$
  n = q\left\lfloor \frac{n}{q} \right\rfloor + r
  = q\left\lfloor nx\right\rfloor + r
$$

where $r$ is the remainder of the division $n/q$. Replacing $\left\lfloor nx\right\rfloor$ by $n$ and $r$ using the above equation, we get

$$
  \frac{\left\lfloor nx\right\rfloor + 1}{n}
  = \frac{(n-r)/q + 1}{n} = \frac{n + (q-r)}{qn}
  = \frac{1}{q} + \frac{q-r}{qn}.
$$

Therefore, minimizing the above is equivalent to minimizing $\frac{q-r}{n}$. Now, it seems reasonable to believe that the minimizer $n$ must have the largest possible remainder $r=q-1$, because for example if we take $r=q-2$ instead, then the numerator gets doubled, so we need to take an $n$ more than two times larger to compensate that increment. Also, among $n$'s with $r=q-1$, obviously the largest $n$ yields the smallest value of $\frac{q-r}{n}$, so it sounds rational to say that probably the greatest $n$ with $r=q-1$ is the minimizer of $\frac{q-r}{n}$. Indeed, this is quite easy to prove: suppose we call such $n$ as $v$, and suppose that there is $n$ which is even better than $v$:

$$
  \frac{q-r}{n} \leq \frac{1}{v},
  \quad (q-r)v \leq n.
$$

Now, since $v$ divided by $q$ has the remainder $q-1$, the left-hand side and the right-hand side have the same remainder when divided by $q$. Therefore, the difference between the two must be either zero or at least $q$. But since $v$ is the *largest* one with the remainder $q-1$, it must be at least $n_{\max} - q + 1$, thus $n$ cannot be larger than $v$ by $q$. Thus the only possible case is $n=v$.

When $x=\frac{p}{q}$ and $p\neq 1$, it is somewhat harder to be convinced at once, that still the minimizer of $\frac{\left\lfloor nx\right\rfloor + 1}{n}$ should be such that the remainder $r$ of $np/q$ is equal to $q-1$, because the way $r$ changes as $n$ changes looks pretty random. But essentially the same logic as above just works also in this case. The full proof can be found in the paper mentioned, or [one](https://jk-jeon.github.io/posts/2021/12/continued-fraction-floor-mult/) of my previous posts.

## Some applications

Here I collected some applications of the presented theorem.

### Finding the first error case

In the beginning of the previous section, I claimed that

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

We can also see now why a better magic number worked in the example from the previous section. Let me repeat it here with the notation of **Theorem 2**: we have $x=1/102807$ and $n_{\max}=2^{32}-1$, so we have

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

Suppose that we want to convert the temperature from Fahrenheit to Celsius. Obviously, representing such values only using integers is a funny idea, but let us pretend that we are completely serious (and hey, in 2023, *Fahrenheit itself* is a funny joke from the first place😂... except that I am currently living in an interesting country🤮). Or, if one *desperately* wants to make a really serious example, we can maybe think about doing the same thing with fixed-point fractional numbers. But whatever.

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

Might be, but there is a caveat: in general, the compiler is not allowed to optimize an expression `n * p / q` into `(n * m) >> k`, because `n * p` can overflow and in that case dividing by `q` will give a weird answer. To be more specific, for unsigned integer types, C/C++ standards say that any overflows should wrap around, so the expression `n * p / q` is not really supposed to compute $\left\lfloor \frac{np}{q}\right\rfloor$, rather it is supposed to compute $\left\lfloor \frac{(np\ \mathrm{mod}\ 2^{N})}{q}\right\rfloor$ where $N$ is the bit-width, even though it is quite likely that the one who wrote the code actually wanted the former. On the other hand, for signed integer types, (a signed-equivalent of) **Theorem 2** might be applicable, because signed overflows are specifically [defined to be undefined](https://en.cppreference.com/w/cpp/language/operator_arithmetic#Overflows). But presumably there are lots of code out there relying on the wrong assumption that signed overflows will wrap around, so maybe compiler-writers do not want to do this kind of optimizations.

Nevertheless, there are situations where doing this is perfectly legal. For example, suppose `n`, `p`, `q` are all of type `std::uint32_t` and the code is written like `static_cast<std::uint32_t>((static_cast<std::uint64_t>(n) * p) / q)` to intentionally avoid this overflow issue. Then the compiler might recognize such a pattern and do this kind of optimizations. Or more generally, with the new `assume` attribute (or some other equivalent compiler-specific mechanisms), the user might give some assumptions that ensure no overflow.

It seems that currently both clang and GCC [do not do this](https://godbolt.org/z/KhM14oqbE), so if they want to do so in a future, then **Theorem 2** might be useful. But how many code can benefit from such an optimization? Will it be really worth implementing? I do not know, but maybe not.

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
- We subtract the upper half $u$ from the input $n$ (the `sub` line), divide the result by 2 (the `shr` line), add the result back to $u$ (the `lea` line), and then store the end result into `rax`. Now this looks a bit puzzling, but what it really does is nothing but to compute $\left\lfloor (n + u)/2 \right\rfloor = \left\lfloor nm / 2^{65}\right\rfloor$. The reason why it first subtract $u$ from $n$ is to avoid overflow. And in case anyone is curious, the subtraction is totally fine as there can be no underflow, because $u = \left\lfloor nm'/2^{64}\right\rfloor$ is at most $n$, as $m'$ is less than $2^{64}$.
- Recall that our $k = 78$, so we want to compute $\left\lfloor nm/2^{78} \right\rfloor$. Since we got $\left\lfloor nm / 2^{65}\right\rfloor$ from the previous step, we just need to shift this further by $13$-bits.

This is not so bad, we just have two more trivial instructions compared to the happy case. But the reason why the above works is largely due to that the magic number is just one bit larger than $64$-bits. Well, can it be even larger than that?

The answer is: **No**, fortunately, the magic number being just one bit larger than the word size is the worst case.

To see why, note that the size of the interval where $\xi=\frac{m}{2^{k}}$ can possibly live is precisely $1/vq$. Therefore, if $k$ is large enough so that $2^{k}\geq vq$, then the interval between the endpoints of the inequality

$$
  \frac{2^{k}}{q} \leq m < 2^{k}\left(\frac{1}{q} + \frac{1}{vq}\right)
$$

has the length at least $1$, so it must admit an integer solution. Now, we are interested in the bit-width of $\left\lceil\frac{2^{k}}{q}\right\rceil$, which must be the smallest possible magic number if $k$ admits at least one solution. Since the smallest admissible $k$ is at most the smallest $k$ satisfying $2^{k}\geq vq$, thus we must have $vq>2^{k-1}$, so $\frac{2^{k}}{q} < 2v$. And the right-hand side is of at most one bit larger than the word size.

Actually, there is an alternative way of dealing with the case of too large magic number, which is to consider a slightly different formula: instead of just doing a multiplication and then a shift, perform an addition by another magic number between those two operations. Using the notations from **Theorem 2**, what this means is that we have some $\zeta$ satisfying the equality

$$
  \left\lfloor nx \right\rfloor = \left\lfloor n\xi + \zeta \right\rfloor
$$

instead of $\left\lfloor nx \right\rfloor = \left\lfloor n\xi \right\rfloor$. The presence of this $\zeta$ might allow us to have a smaller magic number so that it fits into a word. The topic of the next section is about the condition for having the above equality.

Here is a small remark before getting into the next section: this trick of having $\zeta$ is probably not so useful for $64$-bit divisions, because addition is no longer a trivial operation since the result of the multiplication $mn$ spans two $64$-bit blocks. We need an `adc` (add-with-carry) instruction or an equivalent, which is not particularly well-optimized in typical x86-64 CPU's. I did not do a benchmark, but I am guessing that it probably yields worse performance. Presumably, this trick however might give a better performance than the method explained in this section for the $32$-bit case, as every operation be done inside $64$-bits. But it [seems](https://godbolt.org/z/rxzY6raE4) compilers do not do so anyway. I do not know if this is because it actually performs worse, or because they just did not bother to implement it.

# Multiply-add-and-shift rather than multiply-shift

>**WARNING**: The contents of this section is **substantially** more complicated and math-heavy than previous sections.

As said in the last section, we will explore the condition for having

$$
  \left\lfloor nx \right\rfloor = \left\lfloor n\xi + \zeta \right\rfloor
$$

for all $n=1,\ \cdots\ ,n_{\max}$, where $x$, $\xi$ are real numbers and $\zeta$ is a nonnegative real number. We will derive the optimal bound, i.e., an "if and only if" condition. We remark that an optimal bound has been obtained in the paper by [Lemire et al](https://doi.org/10.1016/j.heliyon.2021.e07442) mentioned above for the special case when $x=\frac{1}{q}$ for some $q\leq n_{\max}$ and $\xi=\zeta$ and is rational. According to their paper, the proof of the optimality of their bound is almost identical to the case of having no $\zeta$, so they even did not bother to write down the proof. However, for the general case I am dealing here, the presence of $\zeta$ together with $x$ being not restricted to reciprocals of integers actually **do** complicate things a lot.

Just like the case $\zeta=0$ (i.e., **Theorem 2**), having the equality for all $n=1,\ \cdots\ ,n_{\max}$ is equivalent to having the inequality

$$
  \max_{n=1,\ \cdots\ ,n_{\max}}\frac{\left\lfloor nx\right\rfloor - \zeta}{n}
  \leq \xi
  <\min_{n=1,\ \cdots\ ,n_{\max}}\frac{\left\lfloor nx\right\rfloor + 1 - \zeta}{n},
$$

so the question is how to find the maximum and the minimum on the left-hand side and on the right-hand side, respectively.

## The lower bound

Since $\zeta\geq 0$ is supposed to be just a small constant, it sounds reasonable to believe that the maximizer of $\frac{\left\lfloor nx\right\rfloor}{n}$ is probably quite close to be the maximizer of $\frac{\left\lfloor nx\right\rfloor - \zeta}{n}$ as well. So let us start there, let $n_{0}$ be the largest among all maximizers. Now, what should happen if $n_{0}$ were not the maximizer of $\frac{\left\lfloor nx\right\rfloor - \zeta}{n}$? Say, what can we say about $n$'s such that

$$
  \frac{\left\lfloor nx\right\rfloor - \zeta}{n}
  \geq \frac{\left\lfloor n_{0}x\right\rfloor - \zeta}{n_{0}}
$$

holds? First of all, since this inequality implies

$$
  -\frac{\zeta}{n} \geq -\frac{\zeta}{n_{0}},
$$

we must have $n\geq n_{0}$ unless $\zeta=0$, which is an uninteresting case anyway. (Now the reason why we chose $n_{0}$ to be as large as possible becomes apparent.)

As a result, we can equivalently reformulate our optimization problem as follows: given $n=0,\ \cdots\ ,n_{\max}-n_{0}$, find the maximizer of

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

In other words, $\frac{p}{q}$ is a best rational approximation of $x$ if any better rational approximation must have larger denominator.

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

First, as pointed out before, our $\frac{\left\lfloor n_{0}x\right\rfloor}{n_{0}}$ is a best rational approximation from below of $x$. This is obvious from its definition: $n_{0}$ is the maximizer of $\frac{\left\lfloor nx\right\rfloor}{n}$ for $n=1,\ \cdots\ ,n_{\max}$, which means that whenever we have

$$
  \frac{\left\lfloor n_{0}x \right\rfloor}{n_{0}}
  < \frac{a}{b} \leq x,
$$

since $a \leq \left\lfloor bx\right\rfloor$ holds (because otherwise the inequality $\frac{a}{b}\leq x$ fails to hold), we must have $b>n_{\max}$, so in particular $n>n_{0}$.

Therefore, by the aforementioned equivalence of the two concepts, we immediately know that $\frac{\left\lfloor n_{0}x\right\rfloor}{n_{0}}$ is a best rational approximation from below of $x$ in the strong sense. This does *not* imply that $n_{0}$ is a minimizer of $nx - \left\lfloor nx\right\rfloor$ for $n=1,\ \cdots\ ,n_{\max}$ because $\frac{\left\lfloor n_{0}x\right\rfloor}{n_{0}}$ is not necessarily in its reduced form (as we chose $n_{0}$ to be the largest maximizer), but if we denote its reduced form as $\frac{p}{q}$, then certainly $q$ is the minimizer of it (and $p$ must be equal to $\left\lfloor qx\right\rfloor$). Indeed, by the definition of best rational approximations from below in the strong sense, if $n$ is the smallest integer such that $n>q$ and

$$
  nx - \left\lfloor nx\right\rfloor < qx - p,
$$

then $\frac{\left\lfloor nx\right\rfloor}{n}$ itself must be a best rational approximation from below in the strong sense, thus also in the weak sense. This means $\frac{\left\lfloor nx\right\rfloor}{n}$ is at least $\frac{p}{q} = \frac{\left\lfloor n_{0}x\right\rfloor}{n_{0}}$ as $n>q$, and in fact they cannot be equal because otherwise we get

$$
  nx - \left\lfloor nx \right\rfloor
  = nx - \frac{n}{q}p
  = \frac{n}{q}(qx - p)
  > \frac{n}{q}\left(nx - \left\lfloor nx\right\rfloor\right),
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

(Here is a more intuitive explanation. Note that among all $nx$'s, $qx$ is the one with the smallest fractional part. Then whenever $kq + n\leq n_{\max}$, the sum of the fractional parts of $nx$ and that of $k$ copies of $qx$ should not "wrap around", i.e., it should be strictly less than $1$, because if we choose the smallest $k$ that $(kq + n)x$ wraps around, then since $((k-1)q + n)x$ should not wrap around, so its fractional part is strictly less than $1$, which means that the fractional part of $(kq + n)x$ is strictly less than that of $qx$, contradicting to the minimality of the fractional part of $qx$. Then since the fractional parts do not wrap around when added, the integer part of $(kq + n)x$ should be just the sum of the integer parts of its summands.)

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

At this point, we could just enumerate all rational approximations of $x$ from below satisfying the above bound and find out the one that optimizes $\frac{\left\lfloor (n_{0}+n)x \right\rfloor - \zeta}{n_{0}+n}$. Indeed, the theory of continued fractions allows us to develop an efficient algorithm for doing such a task. (See **Algorithm C.13** in my [paper](https://github.com/jk-jeon/dragonbox/blob/master/other_files/Dragonbox.pdf) on Dragonbox for an algorithm doing a closely related task.) However, we can do better here.

First of all, note that if $\zeta$ is small enough, then the only solution to the above inequality is $n=0$, so there is nothing further we need to do. This conclusion is consistent with the intuition that $n_{0}$ should be close enough to the maximizer of $\frac{\left\lfloor nx\right\rfloor -\zeta}{n}$ at least if $\zeta$ is close to zero. But the problem occurs when $\zeta$ is not that small.

It is quite tempting to claim that the optimizer of the left-hand side of $\eqref{eq:lower bound iteration criterion}$ is the optimizer of $\frac{\left\lfloor (n_{0}+n)x \right\rfloor - \zeta}{n_{0}+n}$, but that is not true in general. Nevertheless, we can start from there, just like that we started from $n_{0}$ from the beginning.

In this reason, let $n_{1}$ be the largest $n=1,\ \cdots\ ,n_{\max} - n_{0}$ such that

$$
  x - \frac{\left\lfloor nx\right\rfloor}{n}
$$

is the smallest, or equivalently, $\frac{\left\lfloor nx\right\rfloor}{n}$ is the largest. That is, we just choose the best rational approximation from below of $x$ with the greatest denominator $n_{1} \leq n_{\max} - n_{0}$. As pointed out earlier, if such $n_{1}$ does not satisfy $\eqref{eq:lower bound iteration criterion}$, then there is nothing further to do, so suppose that the inequality is indeed satisfied with $n=n_{1}$. Just like we did before, now we claim that any $n$ that yields a larger value for $\frac{\left\lfloor (n_{0}+n)x \right\rfloor - \zeta}{n_{0}+n}$ should be at least $n_{1}$. That is, suppose we have $n=1,\ \cdots\ ,n_{\max} - n_{0}$ with

$$
  \frac{\left\lfloor (n_{0}+n)x \right\rfloor - \zeta}{n_{0}+n}
  \geq \frac{\left\lfloor (n_{0}+n_{1})x \right\rfloor - \zeta}{n_{0}+n_{1}}.
$$

Then by $\eqref{eq:floor splits; lower bound}$, this is equivalent to

$$\begin{aligned}
  &n_{0}\left\lfloor nx\right\rfloor
  + n_{1}\left(\left\lfloor n_{0}x\right\rfloor + \left\lfloor nx\right\rfloor\right)
  - (n_{0}+n_{1})\zeta  \\
  &\quad\quad\quad \geq
  n_{0}\left\lfloor n_{1}x\right\rfloor
  + n\left(\left\lfloor n_{0}x\right\rfloor + \left\lfloor n_{1}x\right\rfloor\right)
  - (n_{0}+n)\zeta,
\end{aligned}$$

and rearranging it gives

$$
  (n-n_{1})(\zeta - \left\lfloor n_{0}x\right\rfloor) \geq
  n_{0}\left(\left\lfloor n_{1}x\right\rfloor - \left\lfloor nx\right\rfloor\right)
  + \left(n\left\lfloor n_{1}x\right\rfloor - n_{1}\left\lfloor nx\right\rfloor\right).
$$

By adding and subtracting appropriate terms, we can rewrite this as

$$\begin{aligned}
  (n-n_{1})\left(\left(n_{0}x - \left\lfloor n_{0}x\right\rfloor\right) + \zeta\right)
  &\geq n_{0}\left(
    \left(nx - \left\lfloor nx\right\rfloor\right)
    - \left(n_{1}x - \left\lfloor n_{1}x\right\rfloor\right)
  \right) \\
  &\quad\quad\quad
  + nn_{1}\left(
    \frac{\left\lfloor n_{1}x\right\rfloor}{n_{1}}
    - \frac{\left\lfloor nx\right\rfloor}{n}
  \right).
\end{aligned}$$

Note that by definition of $n_{1}$, the right-hand side is always nonnegative, because $\frac{\left\lfloor n_{1}x\right\rfloor}{n_{1}}$ is not only a best rational approximation from below of $x$ in the weak sense but also in the strong sense. Therefore, we must have $n\geq n_{1}$ to satisfy the above inequality, as claimed.

As a result, we can reformulate our optimization problem again in the following way: define $N_{1}=n_{0}+n_{1}$, then we are to find $n=0,\ \cdots\ ,n_{\max} - N_{1}$ which maximizes

$$
  \frac{\left\lfloor (N_{1}+n)x\right\rfloor - \zeta}{N_{1}+n}.
$$

As you can see, now it resembles quite a lot what we just did before. Thus, the natural next step is to see whether we have

$$
  \left\lfloor (N_{1}+n)x\right\rfloor =
  \left\lfloor N_{1}x\right\rfloor + \left\lfloor nx\right\rfloor,
$$

which, by $\eqref{eq:floor splits; lower bound}$, is equivalent to

$$
  \left\lfloor (n_{1}+n)x\right\rfloor =
  \left\lfloor n_{1}x\right\rfloor + \left\lfloor nx\right\rfloor.
$$

But we already know that this is indeed the case, because $\frac{\left\lfloor n_{1}x\right\rfloor}{n_{1}}$ is a best rational approximation from below of $x$, so the exact same proof of $\eqref{eq:floor splits; lower bound}$ applies.

Therefore, by the same procedure as we did before, we get that

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

>**Algorithm 5** (Computing the lower bound)**.**
>
>Input: $x\in\mathbb{R}$, $n_{\max}\in\mathbb{Z}_{>0}$, $\zeta\geq 0$.
>
>Output: the largest maximizer of $\frac{\left\lfloor nx\right\rfloor - \zeta}{n}$ for $n=1,\ \cdots\ ,n_{\max}$.
>
>1. Find the largest $n=1,\ \cdots\ ,n_{\max}$ that maximizes $\frac{\left\lfloor nx\right\rfloor}{n}$ and call it $n_{0}$.
>2. If $n_{0} = n_{\max}$, then $n_{0}$ is the largest maximizer; return.
>3. Otherwise, set $n_{\max}\leftarrow n_{\max} - n_{0}$.
>4. Find the largest $n=1,\ \cdots\ ,n_{\max}$ that maximizes $\frac{\left\lfloor nx\right\rfloor}{n}$ and call it $n_{1}$.
>5. Inspect the inequality
>  
>  $$
>    \frac{\left\lfloor n_{1}x\right\rfloor}{n_{1}}
>    \geq \frac{\left\lfloor n_{0}x\right\rfloor - \zeta}{n_{0}}.
>  $$
>  
>6. If the inequality does not hold, then $n_{0}$ is the largest maximizer; return.
>7. If the inequality does hold, then set $n_{0}\leftarrow n_{0} + n_{1}$ and go to Step 2.

*Remark*. In the above, we blackboxed the operation of finding the largest maximizer of $\frac{\left\lfloor nx\right\rfloor}{n}$. Actually, this is precisely how we obtain **Theorem 2**. If $x$ is a rational number whose denominator is at most $n_{\max}$, then obviously the largest maximizer is the largest multiple of $q$ not strictly bigger than $n_{\max}$. Otherwise, we just compute the best rational approximation from below of $x$ with the largest denominator $$q_{*}\leq n_{\max}$$, where the theory of continued fractions allows us to compute this very efficiently. In particular, if $x$ is a rational number, this is really just the extended Euclid algorithm. Once we get the best rational approximation $$\frac{p_{*}}{q_{*}}$$ (which must be in its reduced form), we find the largest multiple $$kq_{*}$$ of $$q_{*}$$ that is still at most $n_{\max}$. Then since $$\frac{p_{*}}{q_{*}}$$ is the maximizer of $\frac{\left\lfloor nx\right\rfloor}{n}$ for $n=1,\ \cdots\ ,n_{\max}$, it follows that $$\left\lfloor kq_{*}x\right\rfloor$$ must be equal to $$kp_{*}$$ and $$kq_{*}$$ is the largest maximizer of $\frac{\left\lfloor nx\right\rfloor}{n}$.

## The upper bound

The computation of the upper bound, that is, solving the minimization problem

$$
  \min_{n=1,\ \cdots\ ,n_{\max}}\frac{\left\lfloor nx\right\rfloor + 1 - \zeta}{n},
$$

is a little bit more involved than the lower bound. However, the overall idea is the same.

Similarly to the case of lower bound, we start with the smallest minimizer $n_{0}$ of $\frac{\left\lfloor nx\right\rfloor + 1}{n}$. Then for any $n=1,\ \cdots\ ,n_{\max}$ with

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

holds for all such $n$. We have two cases: (1) when $\frac{\left\lfloor n_{0}x\right\rfloor + 1}{n_{0}}$ is a best rational approximation from above of $x$, or (2) when it is not. The second case can happen only when $x$ is a rational number whose denominator is at most $n_{\max}$, so that the best rational approximation of it is $x$ itself. However, for such a case, if we let $x=\frac{p}{q}$, then according to how we derived **Theorem 2**, the remainder of $n_{0}p$ divided by the denominator $q$ of $x$ must be the largest possible value, $q-1$. Hence, the quotient of the division $(n_{0} - n)p/q$ cannot be strictly smaller than the difference between the quotients of the divisions $n_{0}p/q$ and $np/q$, so the claim holds in this case.

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

Using the claim, we can again characterize $n=1,\ \cdots\ ,n_{0}-1$ which yields a smaller value for

$$
  \frac{\left\lfloor (n_{0}-n)x \right\rfloor + 1 - \zeta}{n_{0}-n}
$$

than the case $n=n_{0}$. Indeed, we have the inequality

$$
  \frac{\left\lfloor (n_{0}-n)x \right\rfloor + 1 - \zeta}{n_{0}-n}
  \geq \frac{\left\lfloor n_{0}x \right\rfloor + 1 - \zeta}{n_{0}}
$$

if and only if

$$
  -n_{0}\left\lfloor nx\right\rfloor + n_{0}(1-\zeta)
  \geq -n\left\lfloor n_{0}x\right\rfloor + (n_{0}-n)(1-\zeta)
$$

if and only if

$$
  n_{0}\left\lfloor nx\right\rfloor \leq
  n\left\lfloor n_{0}x\right\rfloor + n(1-\zeta)
$$

if and only if

$$
  \frac{\left\lfloor nx\right\rfloor}{n}
  \leq \frac{\left\lfloor n_{0}x\right\rfloor + 1 - \zeta}{n_{0}},
$$

or equivalently,

$$\label{eq:upper bound iteration criterion}
  x - \frac{\left\lfloor nx\right\rfloor}{n}
  \geq \frac{\zeta}{n_{0}}
  - \left(\frac{\left\lfloor n_{0}x\right\rfloor + 1}{n_{0}} - x\right).
$$

Next, let $n_{1}$ be the *largest minimizer* of

$$
  x - \frac{\left\lfloor nx\right\rfloor}{n}
$$

for $n=1,\ \cdots\ ,n_{0}-1$. If the above quantity is strictly bigger than

$$
  \frac{\zeta}{n_{0}}
  - \left(\frac{\left\lfloor n_{0}x\right\rfloor + 1}{n_{0}} - x\right),
$$

then we know for sure that $n_{0}$ is the one we are looking for, so suppose the otherwise.

Next, we claim that we have the inequality

$$
  \frac{\left\lfloor (n_{0}-n)x\right\rfloor + 1 - \zeta}{n_{0}-n}
  < \frac{\left\lfloor (n_{0}-n_{1})x\right\rfloor + 1 - \zeta}{n_{0}-n_{1}}
$$

can ever hold for some $n=1,\ \cdots\ ,n_{0}-1$ only when $n>n_{1}$. By $\eqref{eq:floor splits; upper bound}$, we can rewrite the inequaltiy above as

$$\begin{aligned}
  &-n_{0}\left\lfloor nx\right\rfloor - n_{1}\left\lfloor n_{0}x\right\rfloor
  + n_{1}\left\lfloor nx\right\rfloor + (n_{0}-n_{1})(1-\zeta) \\
  &\quad\quad\quad <
  -n_{0}\left\lfloor n_{1}x\right\rfloor - n\left\lfloor n_{0}x\right\rfloor
  + n\left\lfloor n_{1}x\right\rfloor + (n_{0}-n)(1-\zeta),
\end{aligned}$$

or

$$
  (n-n_{1})\left(\left\lfloor n_{0}x\right\rfloor + 1 - \zeta\right) <
  n_{0}\left(\left\lfloor nx\right\rfloor - \left\lfloor n_{1}x\right\rfloor\right)
  + \left(n\left\lfloor n_{1}x\right\rfloor - n_{1}\left\lfloor nx\right\rfloor\right).
$$

Then adding and subtracting appropriate terms gives

$$\begin{aligned}
  (n-n_{1})\left(
    \left(\left\lfloor n_{0}x\right\rfloor + 1 - n_{0}x\right) - \zeta
  \right) &<
  n_{0}\left(
    \left(n_{1}x - \left\lfloor n_{1}x\right\rfloor\right)
    - \left(nx - \left\lfloor nx\right\rfloor\right)
  \right) \\
  &\quad\quad\quad
  + nn_{1}\left(
    \frac{\left\lfloor n_{1}x\right\rfloor}{n_{1}}
    - \frac{\left\lfloor nx\right\rfloor}{n}
  \right).
\end{aligned}$$

So far, it is still not immediately obvious why this implies $n>n_{1}$, so we use proof by contradiction. Suppose $n\leq n_{1}$ and the above inequality hold at the same time. Let us rewrite the inequality above as

$$\begin{aligned}
  &n_{0}(n_{1}-n)\left(
    \frac{\zeta}{n_{0}}
    - \left(\frac{\left\lfloor n_{0}x\right\rfloor + 1}{n_{0}} - n_{0}x\right)
  \right) \\
  &\quad\quad\quad<
  n_{1}(n_{0}-n)\left(x - \frac{\left\lfloor n_{1}x\right\rfloor}{n_{1}}\right)
  - n(n_{0}-n_{1})\left(x - \frac{\left\lfloor nx\right\rfloor}{n}\right).
\end{aligned}$$

Recall that we already supposed

$$
  x - \frac{\left\lfloor n_{1}x\right\rfloor}{n_{1}}
  \leq \frac{\zeta}{n_{0}}
  - \left(\frac{\left\lfloor n_{0}x\right\rfloor + 1}{n_{0}} - x\right),
$$

so we have

$$\begin{aligned}
  n(n_{0}-n_{1})\left(x - \frac{\left\lfloor nx\right\rfloor}{n}\right)
  &< \left(n_{1}(n_{0} - n) - n_{0}(n_{1}-n)\right)
  \left(x - \frac{\left\lfloor n_{1}x\right\rfloor}{n_{1}}\right) \\
  &= n(n_{0} - n_{1})
  \left(x - \frac{\left\lfloor n_{1}x\right\rfloor}{n_{1}}\right).
\end{aligned}$$

Since $n(n_{0}-n_{1})>0$, this implies that $\frac{\left\lfloor nx\right\rfloor}{n}$ is a strictly better approximation of $x$ than $\frac{\left\lfloor n_{1}x\right\rfloor}{n_{1}}$, which contradicts to the definition of $n_{1}$. Therefore, we must have $n\geq n_{1}$.

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
  \leq \frac{\left\lfloor N_{1}x\right\rfloor + 1 - \zeta}{N_{1}},
$$

and repeating this procedure gives us the smallest minimizer of $\frac{\left\lfloor nx\right\rfloor + 1 - \zeta}{n}$.

> **Algorithm 6** (Computing the upper bound)**.**
>
> Input: $x\in\mathbb{R}$, $n_{\max}\in\mathbb{Z}_{>0}$, $\zeta\geq 0$.
>
> Output: the smallest minimizer of $\frac{\left\lfloor nx\right\rfloor + 1 - \zeta}{n}$ for $n=1,\ \cdots\ ,n_{\max}$.
>
> 1. Find the smallest $n=1,\ \cdots\ ,n_{\max}$ that minimizes $\frac{\left\lfloor nx\right\rfloor + 1}{n}$ and call it $n_{0}$.
> 2. If $n_{0} = 1$, then $n_{0}$ is the smallest minimizer; return.
> 3. Otherwise, set $n_{\max}\leftarrow n_{0} - 1$.
> 4. Find the largest $n=1,\ \cdots\ ,n_{\max}$ that maximizes $\frac{\left\lfloor nx\right\rfloor}{n}$ and call it $n_{1}$.
> 5. Inspect the inequality
>
> $$
>   \frac{\left\lfloor n_{1}x\right\rfloor}{n_{1}}
>   \leq \frac{\left\lfloor n_{0}x\right\rfloor + 1 - \zeta}{n_{0}}.
> $$
>
> 6. If the inequality does not hold, then $n_{0}$ is the smallest minimizer; return.
> 7. If the inequality does hold, then set $n_{0}\leftarrow n_{0} - n_{1}$ and go to Step 2.

*Remark*. Similarly to the case of lower bound, we blackboxed the operation of finding the smallest minimizer of $\frac{\left\lfloor nx\right\rfloor}{n}$, which again is precisely how we obtain **Theorem 2**. If $x=\frac{p}{q}$ is a rational number with $q\leq n_{\max}$, then the minimizer is unique and is the largest $v\leq n_{\max}$ such that $vp\equiv -1\ (\mathrm{mod}\ q)$. Otherwise, we just compute the best rational approximation from above of $x$ with the largest denominator $$q_{*}\leq n_{\max}$$, where the theory of continued fractions allows us to compute this very efficiently. The resulting $$\frac{p_{*}}{q_{*}}$$ must be in its reduced form, so $$q_{*}$$ is the smallest minimizer of $\frac{\left\lfloor nx\right\rfloor + 1}{n}$.

## Finding feasible values of $\xi$ and $\zeta$

Note that our purpose of introducing $\zeta$ was to increase the gap between the lower and the upper bounds of $\xi=\frac{m}{2^{k}}$ when the required bit-width of the constant $m$ is too large. Thus, $\zeta$ is not something give, rather is something we want to figure out. In this sense, **Algorithm 5** and **Algorithm 6** might seem pretty useless because they work only when the value of $\zeta$ is already given.

Nevertheless, the way those algorithms work for a fixed $\zeta$ is in fact quite special in that it allows us to figure out a good $\zeta$ fitting into our purpose of widening the gap between two bounds. The important point here is that, *$\zeta$ only appears in deciding when to stop the iteration, and all other details of the actual iteration steps do not depend on $\zeta$ at all*.

Hence, our strategy is as follows. First, set the initial search range of $\zeta$ to be the interval $[0,1)$ (otherwise $\left\lfloor nx\right\rfloor = \left\lfloor n\xi+\zeta\right\rfloor$ fails to hold for $n=0$). Then we can partition the interval $[0,1)$ into subintervals of the form $[\zeta_{\min},\zeta_{\max})$ where the numbers of iterations that **Algorithm 5** and **Algorithm 6** will take remain constant. Then we look at these subintervals one by one, from left to right, while proceeding the iterations of **Algorithm 5** and **Algorithm 6** if needed whenever we move into the next subinterval.

Now, take a subinterval $[\zeta_{\min},\zeta_{\max})$. If there is at least one feasible choice of $\xi$ for some $\zeta\in[\zeta_{\min},\zeta_{\max})$, then such $\xi$ must lie in the interval

$$
  I:= \left(\frac{\left\lfloor n_{L,0}x\right\rfloor - \zeta_{\max}}{n_{L,0}}, \frac{\left\lfloor n_{U,0}x\right \rfloor + 1 - \zeta_{\min}}{n_{U,0}}\right),
$$

where $n_{L,0}$ and $n_{U,0}$ mean $n_{0}$ for **Algorithm 5** and **Algorithm 6**, respectively.

Next, we will take a loop over all candidates for $\xi$ in the interval and check one by one if that candidate is really a possible choice of $\xi$. Let $N$ be the maximum allowed bit-width for the magic constants. Let $\Delta$ be the size of the interval above, that is,

$$
  \Delta := \frac{\left\lfloor n_{U,0}x\right \rfloor + 1 - \zeta_{\min}}{n_{U,0}}
  - \frac{\left\lfloor n_{L,0}x\right\rfloor - \zeta_{\max}}{n_{L,0}}.
$$

Define $k_{0}:=\left\lfloor\log_{2}\frac{1}{\Delta}\right\rfloor + 1$, then we have

$$
  1 < 2^{k_{0}}\Delta \leq 2,
$$

and this means that there are at least one and at most two integers in the interval $2^{k_{0}}I$.

If there is an even integer in $2^{k_{0}}I$, then call it $2^{b}t$ for some odd number  $t$, then our first candidate $\xi$ is

$$
  \xi = \frac{t}{2^{k_{0} - b}}.
$$

This $t$ is the smallest possible value of the magic constant $m$ in the current subinterval. Since we want to take $\xi$ and $\zeta$ to share the same denominator, we may need to increase the exponent $k_{0}-b$ in order to allow a feasible value of $\zeta$. This of course requires the numerator $t$ to be multiplied by the same power of $2$, and we want the resulting number to be strictly less than $2^{N}$. This gives us the range of $k$'s we need to consider.

Next, suppose that $k$ is given in that range and the numerator $m$ of $\xi = \frac{m}{2^{k}}$ is taken accordingly. Then we first find the smallest $\zeta$ that allows the above $\xi$ to be at least $\frac{\left\lfloor n_{L,0}x\right\rfloor - \zeta}{n_{L,0}}$; that is, take

$$
  \zeta_{0} := \left\lfloor n_{L,0}x \right\rfloor - n_{L,0}\xi
  = \frac{2^{k}\left\lfloor n_{L,0}x\right\rfloor - n_{L,0}m}{2^{k}}.
$$

This $\zeta_{0}$ may be not feasible since it can be smaller than $\zeta_{\min}$, so we find the smallest integer which when added to the numerator of $\zeta_{0}$, it becomes at least $\zeta_{\min}$. Then we call the resulting number $\zeta$. If the numerator of $\zeta$ is strictly smaller than $2^{N}$, then we finally compute the true upper bound

$$
  \frac{\left\lfloor n_{U,0}x\right \rfloor + 1 - \zeta}{n_{U,0}}
$$

for $\xi$ using this $\zeta$ and see if $\xi$ is strictly less than the above. If either of them is not the case, then we increase $k$ and retry. If we run out of all possible values of $k$, then we conclude $\xi$ is not feasible.

(Or, we may reduce the range of $k$ in advance by looking at the smallest possible numerator of $\zeta_{0}$, which is $2^{k_{0}-b}\left\lfloor n_{L,0}x\right\rfloor - n_{L,0}t$. Then the maximum $k$ we have to consider is decided by the larger one between this and $t$.)

If there is no even integer in $2^{k_{0}}I$ or there is one but is determined to be not feasible, then we look at the unique odd integer $t$ in $2^{k_{0}}I$. Then we now try with the candidate

$$
  \xi = \frac{t}{2^{k_{0}}}.
$$

We do the same thing; figure out the range of $k$, and for each $k$ find the smallest $\zeta$ and see if $\xi$ lies in the interval dictated by $\zeta$.

If we still have no feasible choice of $\xi$ and $\zeta$ so far, then we now look at the interval $2^{k_{0}+1}I$ and repeat the same procedure with candidate $\xi$'s in that interval. Note that this time we only need to consider those with odd numerators, because those with even numerators are already checked. If there still is no feasible choice of $\xi$ and $\zeta$, then we double the interval again and try candidate $\xi$'s there with odd numerators. We stop and conclude failure if the smallest candidate $\xi$ in the interval already has the numerator greater than or equal to $2^{N}$, in which case we should move into the next subinterval of $\zeta$.

Writing this out formally, we arrive at the following algorithm.

>**Algorithm 7** (Finding feasible values of $\xi$ and $\zeta$)**.**
>
>Input: $x\in\mathbb{R}$, $n_{\max}\in\mathbb{Z}_{>0}$, $N\in\mathbb{Z}_{>0}$.
>
>Output: $k$, $m$, and $s$, where $\xi = \frac{m}{2^{k}}$ and $\zeta = \frac{s}{2^{k}}$, so that we have $\left\lfloor nx\right\rfloor = \left\lfloor \frac{nm + s}{2^{k}}\right\rfloor$ for all $n=1,\ \cdots\ ,n_{\max}$.
>
>1. Set $\zeta_{\min} \leftarrow 0$, $\zeta_{L,\max} \leftarrow 1$, $\zeta_{U,\max} \leftarrow 1$, $n_{L,\max}\leftarrow n_{\max}$, $n_{U,\max}\leftarrow n_{\max}$.
>2. Find the largest $n=1,\ \cdots\ ,n_{\max}$ that maximizes $\frac{\left\lfloor nx\right\rfloor}{n}$ and call it $n_{L,0}$.
>3. Find the smallest $n=1,\ \cdots\ ,n_{\max}$ that minimizes $\frac{\left\lfloor nx\right\rfloor + 1}{n}$ and call it $n_{U,0}$.
>4. 