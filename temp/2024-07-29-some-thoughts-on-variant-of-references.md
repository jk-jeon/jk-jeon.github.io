---
title: "Some thoughts on variant of references"
date: 2024-07-29
permalink: /posts/2024/07/some-thoughts-on-variant-of-references/
tags:
  - programming
  - C/C++
---

# Introduction

When working on my [`idiv` project](https://github.com/jk-jeon/idiv) I ran into a problem of implementing a variant, a.k.a. a tagged union. The reason why I did not want to use `std::variant` was initially twofold: first, I want to keep the project within C++20, but `std::variant` is not really `constexpr` in C++20 (which is retroactively fixed by the defect report [P2231R1](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2021/p2231r1.html) but at the time of writing, the patch does not seem to be widely available among existing standard library implementations, as far as I know), and second, I do not like the way `std::variant` deals with exceptions, i.e. the possibility of being `valueless_by_exception` which I personally think is the worst possible option among possible design choices.

Unfortunately, it turned out there was another issue that I needed to think about: one of my use-cases of `std::variant`-like types was involving a variant of reference-like proxy types, which thus brought the infamous issue of *variant-of-references* into my attention. As a result, I read again [this great article](https://brevzin.github.io/c++/2022/05/24/optional-assignment/) by Barry Revzin. This post is about what I thought after reviewing the article and my thoughts on the issue of *variant-of-references* in general. (That article almost exclusively talks about optional types, but the case of variants is basically the same.)

# The issue

Let us start with what precisely is the issue and what is the fundamental reason why the issue exists. For sake of comparison, let us think about the case of `std::tuple` first. As well-known, `std::tuple<T1, ... ,TN>` is supposed to model the *product* of types `T1`, ... , `TN`. Abstractly speaking, given an arbitrary collection $\left\{T_{i}\right\}_{i\in I}$, of types, we can think of the product type $T\coloneqq\prod_{i\in I}T_{i}$. The category-theoretic meaning of this is that, given an arbitrary collection $\left\{f_{i}\colon U \to T_{i}\right\}_{i\in I}$ of functions from another type $U$, there uniquely exists the "aggregated function" $f\colon U \to T$ making the following diagram commutative (i.e., the arrow $U\to T_{i}$ is equal to the composition $U\to T\to T_{i}$), where the arrow $T\to T_{i}$ is the canonical projection (a.k.a. the *component map*):

$$
  \begin{array}{ccc}
    U & \xrightarrow{f_{i}} & T_{i} \\
    \!\!\!\!\! {\scriptsize f}\downarrow & \nearrow \\
    T
  \end{array}
$$

This is called the *universal property (of products)*. Of course, this is just an incredibly convoluted way of saying the triviality: "you can aggregate all of $f_{i}$'s to form a function $f\colon U \to T$". Anyway, the point is that *forming $f$ out of $f_{i}$'s is a **natural** operation we can do with $T$*.

Now, let us think about the case of the assignment operator `T& T::operator=(U u)`. Since assignment is fundamentally a state-changing operation, the correct way of understanding it mathematically is to think of it as a function with signature $T\times U\to T$, that is, we take not only the literal input `u` but also *the current state* of the object pointed by `this` as the real input of the function, and we take the *state after the assignment* of said object as the real output. In this interpretation, we can immediately see that the assignment operator for $T$ follows naturally by aggregating the assignment operators for $T_{i}$'s.

Indeed, suppose that for each $i$ we have an assignment operator `Ti& Ti::operator=(Ui)`, that is, a function with the signature $T_{i}\times U_{i}\to T_{i}$. Let $U\coloneqq\prod_{i\in I}U_{i}$, then by composing with the canonical projections $T\to T_{i}$ and $U\to U_{i}$, we obtain the function $T\times U\to T_{i}$. The interpretation of this function is: given an instance $t = \left(t_{j}\right)_{j\in I}$ of $T$ and $u = \left(u_{j}\right)_{j\in I}$ of $U$, we first extract the $i$-th components $(t_{i},u_{i})$ of $(t,u)$, and then map them through the assignment operator $T_{i}\times U_{i}\to T_{i}$. Then we will get an instance of $T_{i}$.

Then by the universal property, we can aggregate these functions to obtain a function $T\times U\to T$. Again, this is just an incredibly convoluted way of saying the triviality: "the assignment from $U$ into $T$ is nothing but the aggregation of assignments from $U_{i}$ into $T_{i}$", and the only point here is just that everything is so natural.

What about `std::variant`? As well-known, `std::variant<T1, ... ,TN>` is supposed to model the *coproduct* (a.k.a. *sum*) of types `T1`, ... , `TN`. Abstractly speaking, given an arbitrary collection $\left\{T_{i}\right\}_{i\in I}$, of types, we can think of the coproduct type $T\coloneqq\coprod_{i\in I}T_{i}$. The category-theoretic meaning of this is that, given an arbitrary collection $\left\{f^{i}\colon T_{i} \to U\right\}_{i\in I}$ of functions *into* another type $U$, there uniquely exists the "aggregate function" $f\colon T \to U$ making the following diagram commutative, where the arrow $T_{i}\to T$ is the canonical inclusion (i.e., instantiating $T$ with an instance of $T_{i}$):

$$
  \begin{array}{ccc}
    U & \xleftarrow{f^{i}} & T_{i} \\
    \!\!\!\!\! {\scriptsize f}\uparrow & \swarrow \\
    T
  \end{array}
$$

This is called the *universal property (of coproducts)*. Note that the only difference from the case of products is that the direction of every arrow is reversed. Again, this is just an incredibly convoluted way of saying the triviality: "by visiting the currently engaged alternative and then invoking the corresponding $f^{i}$ with it, you can aggregate all of $f^{i}$'s to form a function $f\colon T \to U$".

Now, the problem is that, it is not so clear how to form the assignment operator for $T = \coprod_{i\in I}T_{i}$ out of operations we have on $T_{i}$'s, *using only the universal property*. That was a triviality for the case of products, but not quite for coproducts. Indeed, let us ask, can we build a function of the signature $T\times T\to T$ which is supposed to mean the assignment operator, out of operations we have on $T_{i}$'s, like the assignment operator $T_{i}\times T_{i}\to T_{i}$? More precisely, is there *only one canonical way* of building such a function?

The answer is no. If a function $T\times T\to T$ is built out of the universal property, then it must be the aggregation of either functions of the form $T_{i}\times T\to T$ or $T\times T_{j}\to T$, which both in turn should be aggregations of functions of the form $T_{i}\times T_{j}\to T$. Again, this abstract nonsense is nothing really deep, rather it simply means: "the way the assignment from an instance of $T$ into another instance of $T$ is specified, is by specifying what should happen when the assignee is engaged with an instance of $T_{i}$ and the assignor is engaged with an instance of $T_{j}$, for all $i,j\in I$". The point of the abstract nonsense is simply that this is *the only natural way* of building a function $T\times T\to T$.

What should be this function with the signature $T_{i}\times T_{j}\to T$ supposed to do? It should: "assign an instance of $T_{j}$ into $T$ which is currently engaged with an instance of $T_{i}$, and we get an instance of $T$ as a result". However, there are multiple possible interpretations of what this *really* means. Or, I would rather say there is no universally natural interpretation at all, but let us consider possible answers anyway.

The first approach is that, we assign $T_{j}$ into $T_{i}$, get $T_{i}$ as a result, and then inject it into $T$, *assuming that* assignments of the form $T_{i}\times T_{j}\to T_{i}$ are well-defined for all $i,j\in I$, and simply have no assignment operator for $T$ otherwise. This means that we assume that every $T_{j}$ can be assigned into every $T_{i}$, and we invoke that assignment when we assign a $T$ with a $T_{j}$-instance into another $T$ with a $T_{i}$-instance. As a result, we never change the engaged alternative of a variant, once constructed, through the assignment operator. I would say this approach is one possible answer, but of course, this would be considered not very intuitive by many. So this approach is not along a direction to pursue.

The second approach is that, we simply implement $T_{i}\times T_{j}\to T$ by dropping the first argument (and then injecting the resulting instance of $T_{j}$ into $T$). In the world of C++, this would mean that we destroy the currently engaged $T_{i}$-instance, and then copy-construct a $T_{j}$-instance into the storage. (This corresponds to "the first take" in the [aforementioned article](https://brevzin.github.io/c++/2022/05/24/optional-assignment/) by Barry Revzin.)

The third approach is that, in $T_{i}\times T_{j}\to T$, we do the same as above if $j\neq i$ (i.e. simply dropping the first argument), but invoke the copy-assignment operator $T_{i}\times T_{i}\to T_{i}$ if $j=i$ (and then injecting the resulting instance of $T_{i}$ into $T$). However, as Barry Revzin points out, essentially this is no different from the second approach: the copy-assignment operator $T_{i}\times T_{i}\to T_{i}$ is no different from just dropping the first argument. The only difference is possible performance advantage, unless $T_{i}$ is a total insanity whose copy-assignment operator does something genuinely different from copy-constructor. Hence, this is usually the preferred way of implementing the assignment operators for variants.

However, Barry Revzin's article then points out an important observation: *this equivalence is only valid if $T_{i}$'s are not reference types*. Indeed, *copy-constructor* of `T&` binds the reference into the constructor argument, while *copy-assignment* of `T&` assigns the argument into the referenced object, not the reference itself.

Now, suppose that we have an instance $t$ of $T$ whose currently engaged type, $T_{i}$, is a reference type, and suppose that we are assigning to $t$ from another instance $t'$ of $T$ whose currently engaged type is $T_{i}$ as well. If we follow the second approach, then since we are always invoking the copy-constructor, $t$ simply rebinds into a different object, the object referenced by $t'$. This behavior is called the *rebinding semantics*. On the other hand, if we follow the third approach, then the assignment affects the object referenced by the engaged $T_{i}$-instance, rather than replacing the $T_{i}$-instance itself. This behavior is called the *assign-through semantics*.

The C++ community seems to be generally convinced in that the rebinding semantics is the only sane answer for `std::optional` and `std::variant` of references, and the assign-through semantics is not even a candidate. However, I consider such a claim quite questionable.

# Refined analysis

Before getting into why I think so by looking at the reasonings given by Barry Revzin one-by-one, first let us correct some errors from what I wrote so far: interpreting the assignment operator `T& T::operator=(U)` as a function with the signature $T\times U\to T$ is in fact not quite correct.

The first issue is that the assignment may throw. Normally, the correct mathematical signature of a throwing function with nominal signature $X\to Y$ is $X\to (Y\amalg E)$, where $E$ is the coproduct of every possible exception type that can be thrown. (Recall that `std::expected` is also a sum type in the same vein.) However, this is not quite correct for state-changing functions, because the state of an out-parameter (or mutable globals/statics) is still there regardless of whether or not an exception has been thrown. For the case of the assignment operator specifically, the correct signature is $T\times U\to T\amalg (T\times E)$: if there is no exception, then the result is of type $T$, otherwise, the result is of type $T\times E$. Or, if we care only about whether or not an exception is thrown but not about the actual exception that has been thrown, then we can instead consider $T\times U\to T\times\{0,1\}$, where $0$ indicates no exception and $1$ indicates an exception. In any case, this correction regarding possible existence of exceptions does not matter very much in the subsequent discussion, so we will simply ignore it.

The second issue, which is directly relevant to the topic of this post, is that $T\times U\to T$ is only correct for *non-reference-like types*, which not only excludes genuine reference types but also proxy types like `std::tuple<T&>`, or even mixed-types like `std::tuple<T, U&>`. Indeed, suppose that $T$ is a reference type, then the assignment from $U$ does not really affect the assignee (of type $T$), rather it only affects the indirect object referenced by the assignee. To encode this behavior mathematically, we can take $V$, the "environment" or the "universe" representing the state of all other objects that $T$ can point to, into account, so that now the correct signature of the assignment from $U$ into $T$ is $T\times V\times U\to T\times V$. When $T$ is a value type, then this assignment does not touch the $V$-component and only changes the $T$-component, and when $T$ is a reference type, it does not touch the $T$-component and only changes the $V$-component. (This is still a quite over-simplified picture, but the point that will be made should stay even if we consider the fully general case.)

Note that this correction does not really affect the overall picture for the case of products. Everything is still natural, there is nothing to argue about. On the other hand, we need to think carefully about how the picture for the case of coproducts should be altered. We want to build a function $T\times V\times T\to T\times V$ where $T\coloneqq\coprod_{i\in I}T_{i}$, and this function is supposed to be an aggregate of functions of the form $T_{i}\times V\times T_{j}\to T\times V$. Let us then review the second and the third approaches in this corrected picture.

The second approach is still just to drop the $T_{i}$-component from $T_{i}\times V\times T_{j}$. Then we get an instance of $V\times T_{j}$, so we swap two components and inject the resulting pair into $T\times V$. The point here is that the $V$-component is never touched (unless the copy-constructor of $T_{j}$ does something spooky).

The third approach is that, when $j\neq i$, we still just drop the $T_{i}$-component and proceed as in the second approach, and when $j=i$, we invoke the assignment operator $T_{i}\times V\times T_{i}\to T_{i}\times V$. Unlike the second approach, the $V$-component may be touched if $T_{i}$ is or contains a reference-like type.

# Investigation of Barry Revzin's reasonings

Given this, let us investigate the reasonings given by Barry Revzin about why he favors the rebinding semantics.

>The result should be based on what we're assigning from, not what we're assigning to. That's decidedly not the case with the copy-assign implementation, where the two different assignments do two very different things.

If $T$ is a reference type, then the results of the assignment operator $T\times V\times U\to T\times V$, both the $T$- and the $V$-components, *do* depend on the value of the $T$-component. Indeed, the $T$-component of the input is mirrored into that of the output, and it indicates where in the $V$-component the actual assignment is happening. So the premise (that the result of the assignment operator should be only dependent on the value held by the assinor, not that of the assignee) is already false for reference-like types. It may still seem to be the case if one tries to look at it with the idea that assignment is a function of the form $T'\times U\to T'$ where $T'$ is $T$ with the reference stripped off, but such an idea is just wrong. The *result* of the assignment operator is an instance of the whole $T\times V$, regardless of whether $T$ is a reference or value or mixture or whatever. When $T$ is a reference, it is wrong to think that the *result* only consists of a tiny portion of $V$ where the assignment actually happened, in the sense that such an idea just does not scale and fails to give consistent answers if we start to ask about things like proxies.

Hence, if we ever want to consider a variant of references as a reference-like type (and I cannot think of any reason why we would not), then it is natural to expect that the result of the assignment operator can be somehow dependent on the current value held by the assignee.

But one can still ask, is the exact way of dependence implied by the third approach really natural? I would not say it is. I already claimed that there is no *canonical way* of defining assignment operators for variants, regardless of whether or not reference types are involved. So my claim is not along the line of "the assign-through semantics is sound", rather it is more along the line of "both are equally broken".

>Second, the copy-assign implementation is valuable as an optimization over the destroy + copy construct implementation. In this case, it does something different, which makes it very much not an optimization anymore. There was no other reason to choose this option to begin with.

"In this case" in the quote above refers to the third approach. The point Barry Revzin is making here is that the only reason why the third approach was ever considered along with the second approach was that copy-assignment is potentially more performant than destroy + copy-construction, and the "Platonic ideal" is and always has been the second approach.

Well, I simply do not agree. Everyone knows that for reference types, copy-assignment does something very different from copy-construction, and we are specifically talking about the case when reference types are involved. And again I think there is simply no canonical answer to how assignments should behave for variants, and I do not see any reason why the second approach is considered more natural than the third.

(Barry Revzin somewhat vaguely suggests that one of the reasons is its simpler, more natural-looking implementation. I agree that is a valid argument in favor of the second approach, but I do not think it alone is conclusive enough.)

>Third, even more than that, copy-assignment would actually be a pessimization for the `Optional<T&>` case. `Optional<T&>`'s storage would be a `T*`. The destroy + copy construct algorithm here actually devolves into a defaulted copy assignment operator, and the whole type ends up being trivially copyable. Which is great. But the copy-assign algorithm requires actually having a user-defined assignment operator, making this case no longer trivially copyable. Using `Optional<T&>`, at least in my experience, is much more common than copy-assigning an `Optional<U>` (for types `U` where copy-assign is more performant than destroy + copy construct), so this is a meaningful pessimization.

(`Optional` here is a simplified version of `std::optional` shown in the article to compare the rebinding and the assign-through semantics.)

To my understanding, the argument here is simply that the second approach is preferred because that will more easily result in trivially-copyable variants, which is a good thing for performance. Well, if we are discussing about what is the correct semantics, then obviously which one will yield a better performance is not really an argument, is it?

I feel like here the argument is actually based on the premise that `Optional<T&>` should be a "better `T*`", therefore `Optional<T&>` should be basically behaving just like `T*`. But honestly I am not sure if I very much agree with that premise. I personally think `T*` is fine as a nullable observer pointer. While I understand the desire for having a "better `T*`" that excludes all unwanted operations irrelevant for a nullable observer (thus having clearer semantics), but I am not sure if I want that "better `T*`" to be precisely `std::optional<T&>`. In my opinion, pointers and references are just completely different things from the type-system perspective, and any attempt to blur the boundary between the two inherentely leads to nonsensical results.

>The only argument to be made in favor the copy-assign implementation for `Optional<T&>`'s copy assignment operator is for consistency - that what `Optional<T>`'s copy assignment operator does is invoke the underlying type's copy assignment, therefore the same should hold for `Optional<T&>`. But as I've noted, this premise doesn't actually hold: there is no such requirement for `Optional<T>`'s copy assignment, so there is no such consistency (and even the copy assignment model doesn't always lead to copy assignment, only sometimes).

Nothing to disagree. But the same argument applies to the destroy + copy-construction approach as well.

>Another interesting argument to consider is generalizing out from Optional<T&> to Variant. What should this do:
>```
>int i = 1;
>float f = 3.14;
>
>Variant<int&, float&> va = i;
>Variant<int&, float&> vb = f;
>
>va = vb; // ???
>```

The argument here is that it is not clear what `va = vb` is supposed to do for the assign-through case. But the real question that is actually relevant here is more about whether our discarded first approach (or some variation of it) turns out to make any sense or not. If we look at the above argument from this viewpoint, it becomes something like: "the second approach is making the most sense, because if we discard the second approach, then we have to choose among the first and the third, and the choice seems ambiguous". Of course that is just not a valid argument in favor of the second approach. More correct way of viewing this should be to take all three into equal account, rather than putting one on one side and putting the other two grouped together on the other side.

>Copy Assignment for `Optional<tuple<T&>>`

This whole section is about the inconsistency resulting from implementing `Optional<T&>` with the second approach but following the third approach for regular `Optional<T>`. I feel like the entire section is an argument in favor of the assign-through semantics, but it seems the author does not say so.

As pointed out earlier, his opinion is that `Optional<T>` is implemented through copy-assignment rather than destroy + copy-construction only as an optimization, not since it is semantically a better fit. It sounds like he thinks `Optional<tuple<T&>>` is doing a semantically wrong thing, as in this case this optimization is not really an optimization, rather is something genuinely different.

I honestly think it is simply not correct to regard copy-assignment as an optimization of destroy + copy-construction, as witnessed by these so-called proxy types like `tuple<T&>`. It is only correct for value-like types, and the whole issue being discussed is entirely about reference-like types from the first place.

There is another argument in favor of the assign-through semantics, which the article by Barry Revzin does not mention: the variant of only one alternative must be isomorphic to the unique alternative given. Nothing deep, just saying that $T = \coprod_{i\in\set{0}}T_{i}$ is equal to $T_{0}$. In this case, the assignment operator for $T$ must behave the same as the assignment operator for $T_{0}$. The assign-through semantics achieves that, but the rebinding semantics does not if $T$ is a reference type.

# More discussions

Thus, it seems every argument the aforementioned article brought up feels kind of moot to me. To my view, since there is no "ideal" definition of variant assignments from the first place, any argument that tries to appeal to intuition, naturality or logical sanity just fails at the end of the day. The right question to ask is therefore, not "what feels more natural", rather it is "what is more useful in practice".

So, let us ask: why do we want assignments for variants from the first place? I think the answer is because we want to store something into a variant, and assignment is the most intuitive, and also the only "blessed" way of doing so which integrates naturally with the rest of the language and the ecosystem. In this viewpoint, I see that the assign-through semantics does not really work, in the sense that I cannot imagine a real application that may utilize it.

For instance, consider the case of `std::optional<T&>`. Assuming the assign-through semantics, how could it be any useful? Given an instance `x` of type `std::optional<int&>`, consider the assignment `x = y`. If we ever want to assign-through `x`, the natural expectation is that this assignment should work for `y` of type `int const`. But of course this does not work, because `x = y` is potentially constructing a new mutable reference to `int` bound to `y`, so `y` must not be `const`. The point is that, the assign-through semantics does not really make `std::optional<int&>` into a well-behaved reference-like type. Just so many innocuous ordinary operations on references do not work at all or even subtly fail on `std::optional<int&>`, if it follows the assign-through semantics.

Of course, the rebinding semantics is no strictly better in this regard, that it does not make `std::optional<int&>` any more reference-like. Rather, it makes it more pointer-like. Which I really dislike, but at least I consider the resulting `std::optional<int&>` is easier to grasp than the abomination resulting from the assign-through.

# Conclusion

I should emphasize again that I do consider the assign-through semantics rather awkward. But I also consider the rebinding semantics just equally awkward (if not more, honestly), and I frankly cannot understand why the whole community seems to be completely sold on the rebinding semantics.

I mean, it is fine that people have settled on the conclusion that the rebinding semantics is practically much more useful and also feels more intuitive to many, and I can live with it if `std::optional<T&>` ends up being specified with the rebinding semantics, even though it will add one more reason why C++ is just hopelessly broken when it comes to proxies. There already are tons of issues regarding proxies, so who cares.

Yet, I do not understand why the majority of people who have thought about this variants-of-references issue seem to be so adamant about their conclusion. I cannot list all the occassions, but I have seen many comments which basically say that anyone who does not prefer the rebinding semantics over the assign-through semantics is just completely clueless. Maybe I am missing something crucial. But I have no idea what is it.

In any case, I am not sure which route I should take for my own implementation of variants. As I pointed out in the introduction, one of my use-cases does involve proxy types. More specifically, I think those proxies will be `const`-views. In such a case, should I disallow assignment at all, or follow the rebinding semantics? Note that since my types are proxies rather than genuine references, the usual implementation of variants, using conditional copy-assignment for value types and unconditional destroy + copy-construction for reference types, does not really give me the rebinding semantics unless I pessimize the case of value types by always enforcing unconditional destroy + copy-construction. I do not think it is possible to reliably "detect" all kinds of proxy types and do something special for them.

One argument in favor of the rebinding semantics in this specific case is that, it does not hurt anyone to just provide more features, and you just do not use them if you do not need them. But it feels very awkward to me that something supposed to be a kind of `T const&` actually allows an assignment operator. I more and more feel like there is something seriously wrong about reference types in C++.