---
title: "The Fourier Transform of the Heaviside Step Function"
date: 2024-08-27
permalink: /posts/2024/08/fourier-transform-of-heaviside-step-function/
tags:
  - analysis
  - distribution-theory
---

Back in 2010, as an electrical engineering major student I was taking an introductory course on signals and systems taught by my formal advisor. The course was mainly about four different kinds of Fourier transforms ([*Discrete-Time Fourier Series* a.k.a. *Discrete Fourier Transform*](https://en.wikipedia.org/wiki/Discrete_Fourier_transform), [*Discrete-Time Fourier Transform*](https://en.wikipedia.org/wiki/Discrete-time_Fourier_transform), [*Continuous-Time Fourier Series*](https://en.wikipedia.org/wiki/Fourier_series), and [*Continuous-Time Fourier Transform*](https://en.wikipedia.org/wiki/Fourier_transform)) and some additional topics like the famous [sampling theorem](https://en.wikipedia.org/wiki/Nyquist%E2%80%93Shannon_sampling_theorem).

The course was decent, the topics taught were interesting and I learned a lot. However, since it was not a course for math majors,  many of the arguments made for developing the theory were not very rigorous, and one of the things that especially bothered me was how the Fourier transform of the [Heaviside step function](https://en.wikipedia.org/wiki/Heaviside_step_function) is obtained.

Here is how the argument given in the class goes. Given a function $f\colon\mathbb{R}\to\mathbb{R}$, its *Fourier transform* is the function $\hat{f}\colon\mathbb{R}\to\mathbb{R}$ given as

$$
  \hat{f}\colon\xi\mapsto \int_{\mathbb{R}}f(x)e^{-2\pi ix\xi}\,dx.
$$

Now consider the Heaviside step function

$$
  H\colon x\mapsto \begin{cases}0 & \textrm{if $x < 0$,} \\ 1 & \textrm{otherwise.}\end{cases}
$$

We want to compute the integral

$$
  \int_{\mathbb{R}}H(x)e^{-2\pi ix\xi}\,dx
  = \int_{0}^{\infty}e^{-2\pi ix\xi}\,dx,
$$

but of course the integral does not converge. To workaround this issue, we first consider the [signum function](https://en.wikipedia.org/wiki/Sign_function)

$$
  \mathrm{sgn}\colon x\mapsto \begin{cases}
    -1 & \textrm{if $x < 0$,} \\ 1 & \textrm{otherwise.}\end{cases}
$$

Since $H = \frac{1}{2}(\mathrm{sgn}+1)$, we know $\hat{H} = \frac{1}{2}\widehat{\mathrm{sgn}} + \frac{1}{2}\delta$ where $\delta$ is the [Dirac delta function](https://en.wikipedia.org/wiki/Dirac_delta_function). But how to compute $\widehat{\mathrm{sgn}}$? Doesn't the integral still diverge? The trick is, first we multiply the exponential decay $$x\mapsto\exp(-\lambda\vert x\vert)$$ for some $\lambda>0$ to $\mathrm{sgn}$, compute the Fourier transform of the resulting function, and then send the decay factor $\lambda$ to $0$. Let us see how it works out.

First, given $\lambda>0$, we compute:

$$
\begin{align*}
  \int_{\mathbb{R}}\mathrm{sgn}(x)e^{-\lambda|x| - 2\pi i\xi x}\,dx
  &= \int_{-\infty}^{0}-e^{(\lambda - 2\pi i\xi)x}\,dx
  + \int_{0}^{\infty}e^{-(\lambda + 2\pi i\xi)x}\,dx \\
  &= -\frac{1}{\lambda - 2\pi i\xi} + \frac{1}{\lambda + 2\pi i \xi} \\
  &= -\frac{4\pi i\xi}{\lambda^{2} + 4\pi^{2}\xi^{2}}.
\end{align*}
$$

Send $\lambda\to 0^{+}$, then we get:

$$
  \widehat{\mathrm{sgn}}(\xi)
  = -\frac{4\pi i\xi}{4\pi^{2}\xi^{2}}
  = \frac{1}{\pi i\xi}.
$$

As a result, we conclude:

$$
  \hat{H}(\xi) = \frac{1}{2\pi i\xi} + \frac{1}{2}\delta(\xi).
$$

Although the end result turns out to be correct, anyone encountering this argument for the first time will ask this question: why do we need to first work with $\mathrm{sgn}$? Why not multiply the exponential decay to $H$ directly?

The thing is, *that gives an incorrect answer*: given $\lambda>0$, we have

$$
  \int_{\mathbb{R}}H(x)e^{-\lambda|x|-2\pi i\xi x}\,dx
  = \int_{0}^{\infty}e^{-(\lambda+2\pi i\xi)x}\,dx
  = \frac{1}{\lambda + 2\pi i\xi},
$$

thus sending $\lambda\to 0^{+}$ gives

$$
  \hat{H}(\xi) = \frac{1}{2\pi i\xi}.
$$

Eh... what happened?

The professor told us that he honestly does not really buy this exponential decay argument, and unfortunately he has no better explanation, so for the moment we should just believe that $\hat{H}(\xi) = \frac{1}{2\pi i\xi} + \frac{1}{2}\delta(\xi)$ is the right one, and we will see it is indeed the right one as we learn through and relate more things together.

Nevertheless, I desparatly wanted to know why the argument seems to work for $\mathrm{sgn}$ but not for $H$, and I knew that the [distribution theory](https://en.wikipedia.org/wiki/Distribution_(mathematics)) is the right thing to look at. However, at that time my mathematical skill was not mature enough to truly appreciate the theory, and although I kind of figured out a way to make the argument rigorous, it was much later when I finally got a full explanation of what is really happening. (If I recall correctly, it was around 2020, *10 years after* I first saw this!) This post is about that explanation.

## The distribution theory explanation

First of all, in facy the exponential decay trick is *valid regardless of the function* we are looking at. This is in fact a simple consequence of the [dominated convergence theorem](https://en.wikipedia.org/wiki/Dominated_convergence_theorem): let $f\colon\mathbb{R}\to\mathbb{R}$ be a [locally integrable function](https://en.wikipedia.org/wiki/Locally_integrable_function) and for each $\lambda>0$, define

$$
  f_{\lambda}\colon x\mapsto e^{-\lambda|x|}f(x).
$$

Then $f_{\lambda} \to f$ as $\lambda\to 0^{+}$ in the distributional sense; that is, for any given test function $\phi\in\mathcal{C}_{c}^{\infty}(\mathbb{R})$, we have

$$
  \left\langle f_{\lambda},\phi\right\rangle
  \mathrel{\unicode{x2254}}
  \int_{\mathbb{R}}f_{\lambda}(x)\phi(x)\,dx
  \to \left\langle f,\phi\right\rangle
$$

as $\lambda\to 0^{+}$. As pointed out, this follows immediately from the dominateed convergence theorem. By the same reason, if $f\colon\mathbb{R}\to\mathbb{R}$ is a locally integrable function that defines a [tempered distribution](https://en.wikipedia.org/wiki/Distribution_(mathematics)#Tempered_distributions) and $\phi\in\mathcal{S}(\mathbb{R})$ is a [Schwartz function](https://en.wikipedia.org/wiki/Distribution_(mathematics)#Schwartz_space), then we have

$$
  \left\langle f_{\lambda},\phi\right\rangle
  \to \left\langle f,\phi\right\rangle
$$

as $\lambda\to 0^{+}$. (That $f$ defines a tempered distribution simply means that $f\phi$ is integrable for every $\phi\in\mathcal{S}(\mathbb{R})$, so the dominated convergence theorem still applies.)

In particular, we do have both $$\mathrm{sgn}_{\lambda}\to\mathrm{sgn}$$ and $$H_{\lambda}\to H$$ as $\lambda\to 0^{+}$.

Next, we see that the Fourier transform $\hat{\cdot}\colon\mathcal{S}'(\mathbb{R})\to\mathcal{S}'(\mathbb{R})$ is continuous with respect to the [weak-$$*$$ topology](https://en.wikipedia.org/wiki/Weak_topology#The_weak_and_weak*_topologies). Because of [how it is defined](https://en.wikipedia.org/wiki/Distribution_(mathematics)#Fourier_transform), this is in fact very trivial: for given [net](https://en.wikipedia.org/wiki/Net_(mathematics)) $$\left(u_{\alpha}\right)_{\alpha\in D}$$ in $\mathcal{S}'(\mathbb{R})$ convergent to some $u\in\mathcal{S}'(\mathbb{R})$ with respect to the weak-$*$ topology, we have

$$
  \lim_{\alpha\in D}\left\langle \hat{u}_{\alpha}, \phi\right\rangle
  = \lim_{\alpha\in D}\left\langle u_{\alpha}, \hat{\phi}\right\rangle
  = \left\langle u,\hat{\phi}\right\rangle
  = \left\langle \hat{u},\phi\right\rangle
$$

for each $\phi\in\mathcal{S}(\mathbb{R})$.

Therefore, we have both $$\widehat{\mathrm{sgn}}_{\lambda} \to \widehat{\mathrm{sgn}}$$ and $$\hat{H}_{\lambda}\to \hat{H}$$ as $\lambda\to 0^{+}$.

But isn't this a contradiction, as we have already seen that $\hat{H}_{\lambda}$ does not converge to $\frac{1}{2}\widehat{\mathrm{sgn}} + \frac{1}{2}\delta$? Actually, it does converge to it. The error committed by the informal, naïve argument given at the beginning of the post is not in the idea of using the exponential decay, rather it is in the way it is executed.

## The limit of $H_{\lambda}$'s

Henceforth, let us directly compute the correct distributional limit of $H_{\lambda}$'s. Fix $\lambda>0$ and $\phi\in\mathcal{S}(\mathbb{R})$, then by definition,

$$
  \left\langle \hat{H}_{\lambda},\phi\right\rangle
  = \int_{0}^{\infty}\int_{\mathbb{R}}e^{-(\lambda + 2\pi i\xi)x}\phi(\xi)\,d\xi\,dx.
$$

Note that the integrand is (absolutely) integrable, so we can apply [Fubini's theorem](https://en.wikipedia.org/wiki/Fubini%27s_theorem) to get

$$
  \left\langle \hat{H}_{\lambda},\phi\right\rangle
  = \int_{\mathbb{R}}\phi(\xi)\int_{0}^{\infty}e^{-(\lambda + 2\pi i\xi)x}\,dx\,d\xi
  = \int_{\mathbb{R}}\frac{\phi(\xi)}{\lambda + 2\pi i\xi}\,d\xi.
$$

In other words, the formula $\hat{H}_{\lambda}(\xi) = \frac{1}{\lambda + 2\pi i\xi}$ is correct. However, the interesting part is on taking the limit $\lambda\to 0^{+}$: since there is no single integrable function that uniformly bounds the integrands for all small enough $\lambda>0$, we cannot blindly take the pointwise limit, and this is where the naïve argument got it wrong.

The key observation we will leverage here is the cancellation between the positive and the negative regions of $\xi$. Let us write our integral as

$$
\begin{align*}
  \int_{\mathbb{R}}\frac{\phi(\xi)}{\lambda + 2\pi i\xi}\,d\xi
  &= \int_{0}^{\infty}\frac{\phi(\xi)}{\lambda + 2\pi i\xi}
  + \frac{\phi(-\xi)}{\lambda - 2\pi i\xi}\,d\xi \\
  &= \int_{0}^{\infty}\frac{\lambda(\phi(\xi) + \phi(-\xi)) - 2\pi i\xi(\phi(\xi) - \phi(-\xi))}
  {\lambda^{2} + 4\pi^{2}\xi^{2}} \,d\xi.
\end{align*}
$$

Now, we separately compute

$$
  I_{1} \mathrel{\unicode{x2254}} \int_{0}^{\infty}\frac{\lambda(\phi(\xi) + \phi(-\xi))}
  {\lambda^{2} + 4\pi^{2}\xi^{2}} \,d\xi,\quad\quad
  I_{2} \mathrel{\unicode{x2254}} \int_{0}^{\infty}\frac{-2\pi i\xi(\phi(\xi) - \phi(-\xi))}
  {\lambda^{2} + 4\pi^{2}\xi^{2}} \,d\xi.
$$

For $I_{1}$, we further divide it into the sum

$$
  I_{1} = \int_{0}^{\infty}\frac{\lambda(\phi(\xi) + \phi(-\xi) - 2\phi(0))}
  {\lambda^{2} + 4\pi^{2}\xi^{2}}
  + \frac{2\lambda\phi(0)}
  {\lambda^{2} + 4\pi^{2}\xi^{2}}\,d\xi.
$$

For each $\xi\in[0,\infty)$, [Taylor's theorem](https://en.wikipedia.org/wiki/Taylor%27s_theorem) shows that $\phi(\xi) + \phi(-\xi) -2\phi(0) = \frac{\xi^{2}}{2}\phi''(\bar{\xi})$ for some $\bar{\xi}\in[0,\xi]$, thus we have

$$
  |\phi(\xi) + \phi(-\xi) - 2\phi(0)|
  \leq \min\left(4\left\|\phi\right\|_{\mathcal{C}^{0}},
  \frac{\xi^{2}}{2}\left\|\phi''\right\|_{\mathcal{C}^{0}}
  \right)
$$

where $$\left\|\,\cdot\,\right\|_{\mathcal{C}^{0}}$$ is the [uniform norm](https://en.wikipedia.org/wiki/Uniform_norm).

Therefore, by splitting the range of integral for $\xi\leq 1$ and $\xi>1$, we can see that the integrand of

$$
  \int_{0}^{\infty}\frac{\lambda(\phi(\xi) + \phi(-\xi) - 2\phi(0))}
  {\lambda^{2} + 4\pi^{2}\xi^{2}}\,d\xi
$$

is uniformly bounded by an integrable function for all $\lambda\in(0,1]$, independently to $\lambda$. Consequently, we can apply the dominated convergence theorem to see

$$
  \int_{0}^{\infty}\frac{\lambda(\phi(\xi) + \phi(-\xi) - 2\phi(0))}
  {\lambda^{2} + 4\pi^{2}\xi^{2}}\,d\xi
  \to 0
  \quad\textrm{as}\quad \lambda\to 0^{+}.
$$

For the second term of $I_{1}$, as usual we apply the change of variables $\xi = \frac{\lambda}{2\pi}\tan\theta$, then we get

$$
  \int_{0}^{\infty}\frac{2\lambda\phi(0)}
  {\lambda^{2} + 4\pi^{2}\xi^{2}}\,d\xi
  = \int_{0}^{\pi/2}\frac{2\lambda\phi(0)}{\lambda^{2}\sec^{2}\theta}
  \cdot \frac{\lambda}{2\pi}\sec^{2}\theta\,d\theta
  = \frac{\phi(0)}{\pi}\int_{0}^{\pi/2}d\theta = \frac{\phi(0)}{2}.
$$

Therefore, we obtain $I_{1}\to \frac{1}{2}\phi(0) = \frac{1}{2}\left\langle \delta, \phi\right\rangle$ as $\lambda\to 0^{+}$.

Now, for $I_{2}$, since we want to see if we can replace the integrand by what we obtain when $\lambda = 0$, let us split $I_{2}$ in this way:

$$
\begin{align*}
  I_{2} &= \int_{0}^{\infty}\left(\phi(\xi) - \phi(-\xi)\right)
  \left(
    \frac{-2\pi i\xi}{\lambda^{2} + 4\pi^{2}\xi^{2}}
    - \frac{1}{2\pi i\xi}
    + \frac{1}{2\pi i\xi}
  \right)\,d\xi \\
  &= \int_{0}^{\infty}\frac{\phi(\xi) - \phi(-\xi)}{2\pi i\xi}
  - \frac{\lambda^{2}(\phi(\xi) - \phi(-\xi))}
  {2\pi i\xi(\lambda^{2} + 4\pi^{2}\xi^{2})}\,d\xi
\end{align*}
$$

Similarly as for $I_{1}$, using the bound

$$
  |\phi(\xi) - \phi(-\xi)|
  \leq \min\left(2\left\|\phi\right\|_{\mathcal{C}^{0}},
  2\xi\left\|\phi'\right\|_{\mathcal{C}^{0}}
  \right)
$$

that follows from Taylor's theorem and then splitting the range of integral into $\xi\leq 1$ and $\xi>1$, the dominated convergence theorem yields

$$
  \int_{0}^{\infty}\frac{\lambda^{2}(\phi(\xi) - \phi(-\xi))}
  {2\pi i\xi(\lambda^{2} + 4\pi^{2}\xi^{2})}\,d\xi \to 0
  \quad\textrm{as}\quad \lambda\to 0^{+}.
$$

Consequently, we get

$$
  \left\langle \hat{H}_{\lambda},\phi\right\rangle
  \to \frac{1}{2}\left\langle \delta, \phi\right\rangle
  + \int_{0}^{\infty}\frac{\phi(\xi) - \phi(-\xi)}{2\pi i\xi}\,d\xi
  \quad\textrm{as}\quad \lambda\to 0^{+}.
$$

Note that the integral $\int_{0}^{\infty}\frac{\phi(\xi) - \phi(-\xi)}{2\pi i\xi}\,d\xi$
converges absolutely because of the bound

$$
  |\phi(\xi) - \phi(-\xi)|
  \leq 2\xi\left\|\phi'\right\|_{\mathcal{C}^{0}},
$$

so we know

$$
  \int_{0}^{\infty}\frac{\phi(\xi) - \phi(-\xi)}{2\pi i\xi}\,d\xi
  = \lim_{\epsilon\to 0^{+}}
  \int_{\epsilon}^{\infty}\frac{\phi(\xi) - \phi(-\xi)}{2\pi i\xi}\,d\xi.
$$

Then since each term of the integrand of the right-hand side is bounded for a fixed $\epsilon>0$, we have

$$
  \int_{\epsilon}^{\infty}\frac{\phi(\xi) - \phi(-\xi)}{2\pi i\xi}\,d\xi
  = \int_{\epsilon}^{\infty}\frac{\phi(\xi)}{2\pi i\xi}\,d\xi
  + \int_{\epsilon}^{\infty}\frac{\phi(-\xi)}{2\pi i(-\xi)}\,d\xi
  = \int_{|\xi|\geq\epsilon}\frac{\phi(\xi)}{2\pi i\xi}\,d\xi,
$$

thus

$$
  \int_{0}^{\infty}\frac{\phi(\xi) - \phi(-\xi)}{2\pi i\xi}\,d\xi
  = \lim_{\epsilon\to 0^{+}}
  \int_{|\xi|\geq \epsilon}\frac{\phi(\xi)}{2\pi i\xi}\,d\xi.
$$

The distribution

$$
  \phi \mapsto \lim_{\epsilon\to 0^{+}}
  \int_{|\xi|\geq \epsilon}\frac{\phi(\xi)}{2\pi i\xi}\,d\xi
$$

is known as the [*principal value integral*](https://en.wikipedia.org/wiki/Cauchy_principal_value#Distribution_theory) of $\frac{1}{2\pi i\xi}$, and is denoted as $\mathrm{p.v.}\left(\xi\mapsto \frac{1}{2\pi i\xi}\right)$.

Putting all pieces together, we now obtain:

$$
  \hat{H} = \mathrm{p.v.}\left(\xi\mapsto \frac{1}{2\pi i\xi}\right) + \frac{1}{2}\delta,
$$

which verifies that

$$
  \hat{H}(\xi) = \frac{1}{2\pi i\xi} + \frac{1}{2}\delta(\xi)
$$

is more or less the correct formula.

I will omit the details, but the same procedure shows that

$$
  \widehat{\mathrm{sgn}} = \mathrm{p.v.}\left(\xi\mapsto\frac{1}{\pi i\xi}\right)
$$

as expected, and is of course consistent with $H = \frac{1}{2}\left(\mathrm{sgn} + 1\right)$.