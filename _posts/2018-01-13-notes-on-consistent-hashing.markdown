---
layout: post
title:  "Notes on consistent hashing"
date:   2018-01-13 18:00:00 +0100
---
<script src="https://cdnjs.cloudflare.com/ajax/libs/mathjax/2.7.2/MathJax.js?config=TeX-AMS-MML_HTMLorMML" type="text/javascript"></script>

I first learnt about consistent hashing from a paper[^paper] that my manager at Google suggested I read. Recently I started thinking about it again as I was considering implementing it in one of my projects. I wanted to make sure that it was reliable enough but, looking again at the paper and searching online, I could only find simple introductions and implementations for the technique and no in-depth theoretical analyses. I thus ended up figuring this out myself, and wrote it down here for future reference.

[^paper]: David Karger et al. "Web caching with consistent hashing." Computer Networks 31.11 (1999): 1203-1213.

TL;DR
---
*[TL;DR]: Too Long; Didn't Read

In a consistent hashing setting with \\(N\\) servers and \\(K\\) markers per server, the fraction of load handled by each server is distributed according to the beta distribution \\(Β(K, (N-1)K)\\). In order for every server to stay within \\(1+\epsilon\\) of its expected load with probability \\(1 - \delta\\) a safe choice of \\(K\\) is \\(N (\epsilon^2 \delta)^{-1}\\).

What is consistent hashing?
---

Consistent hashing is a solution to the following problem. Suppose we have some clients that need to perform operations on some objects. These objects are shared among many servers. We want the clients to be able to determine which server hosts which object in a distributed and consistent way. That is, a client should be able to decide this autonomously, without consulting a central authority (or even other clients/servers), and all clients should agree on these decisions.

If that were all then the possibly simplest solution would be to put a canonical order on the servers (on which all clients can agree) and then, for every object, compute a hash of it, take the result modulo the number of servers and use that value as an index to look up a server in the sorted list.

But we want to add another requirement: we need to be able to add servers to the pool or remove them dynamically at any time and we want this to cause the least number of objects to be moved from one server to another. The above modulo-based approach fails at this.

Consistent hashing takes some inspiration from the modulo-based approach. It consists in imagining a circle of unit length and mapping each server to a random uniformly distributed point on it (since all clients need to agree on the position of this point, it won't actually be random but it will be based on a hash that is sufficiently uniform). The server that hosts a given object is determined as follows: compute a hash of the object, map it to a point on the circle and then walk clockwise along the circle until you reach the first marker of one of the servers; that server is the one the object maps to. Equivalently, we can say that a server's marker covers the arc of the circle that "precedes" it (in clockwise direction) until the previous server's marker. That server will then host all objects whose hashes fall onto that arc.

When using hashes this technique allows to deterministically compute the object-to-server function using only the servers' and objects' hashes, hence every client can perform this calculation locally and agree on the result. Moreover, when adding a server to the pool, its marker will fall on the arc covered by some other server's marker and cut that arc in two, with one portion staying covered by the other server's marker and another portion going under the control of the new server's marker. Hence the object that see their "ownership" transferred are only those whose hashes fall in the arc that the new server took over from the other one. The inverse happens when a server leaves the pool. Hence the only transferred objects are the ones belonging to the server that joins or leaves the pool, which is the optimal behavior.

Since the technique is (pseudo-)random there's inherent uncertainty in its performances. In particular, while each server is expected to receive on average the same amount of objects (and thus of load), it may happen that its marker covers a very long arc and that therefore it receives many more objects (and thus requests) than other servers. To hedge against this risk it is common to have every server place more than one marker on the circle, as if there were many "virtual copies" of it. The larger the number of copies the more its actual load will be likely to be close to the expected ideal average.

Suppose we have \\(N\\) servers, each with \\(K\\) markers. My question is...

How much load does one server get?
---

Let's call \\(R_j^{(i)}\\) the length of the arc covered by the \\(j\\)-th marker of the \\(i\\)-th server. We have of course that \\(\sum_{i=1}^N \sum_{j=1}^K R_j^{(i)} = 1\\). The random variables \\(R_j^{(i)}\\) are not independent but they are identically distributed. This is enough to tell us that \\(\operatorname{E}(R_j^{(i)}) = (NK)^{-1}\\) by symmetry. Let's analyze the distribution of \\(R_j^{(i)}\\) in more detail.

We can suppose without loss of generality that the \\(j\\)-th marker of the \\(i\\)-th server falls on "coordinate zero" of the circle (if that were not the case we could just shift the coordinate system). The length of the arc it covers is equal to coordinate of the marker that follows it, which is the first among all other markers, that is, the one with smallest coordinate. The other markers fall uniformly on the circle, so what we're looking for is the minimum of \\(NK-1\\) independent values uniformly distributed on \\([0, 1]\\).

We'll later need a generalization of this, so let's do the extra work now. Consider \\(M\\) other markers (rather than \\(NK - 1\\)) and suppose the interval on which they fall has length \\(L\\) (rather than \\(1\\)). Let \\(T_{L,M}\\) be a random variable defined as the minimum of \\(M\\) independent random variables uniformly distributed on \\([0, L]\\). We have that
\\[
    \Pr(T_{L,M} \ge t) = \left(\frac{L - t}{L}\right)^{M}
\\]
since all \\(M-1\\) uniformly distributed random variables need to fall in \\([t, L]\\) for that event to happen, each of them does that with probability \\(\frac{L-t}{L}\\) and, as they are all independent, the total probability is the product of the individual ones.

From that we can deduce the probability density function \\(\tau_{L,M}\\) of \\(T_{L,M}\\), which is
\\[
    \tau_{L,M}(t) = \frac{\mathrm{d}}{\mathrm{d}t} \Pr(T_{L,M} \le t) = \frac{\mathrm{d}}{\mathrm{d}t} \left(1 - \Pr(T_{L,M} \ge t) \right) = - \frac{\mathrm{d}}{\mathrm{d}t} \left(\frac{L - t}{L}\right)^{M} = \frac{M}{L} \left(\frac{L - t}{L}\right)^{M-1}
\\]

Since \\(R_j^{(i)} = T_{1,NK-1}\\) the pdf of \\(R_j^{(i)}\\) is \\(\tau_{1,NK-1}\\).

We introduce \\(S_d^{(i)}\\) as \\(\sum_{j=1}^d R_j^{(i)}\\). We are interested in the distribution of \\(S_K^{(i)}\\) but we will show a more general result, namely that the pdf \\(\sigma_d\\) of \\(S_d^{(i)}\\) is:
\\[
    \sigma_d(x) = \frac{(NK-1)!}{(d-1)! (NK-d-1)!} x^{d-1} (1-x)^{NK-d-1}
\\]

Since \\(S_1^{(i)} = R_1^{(i)} = T_{1,NK-1}\\) we have already shown the result for \\(d = 1\\), because \\(\sigma_1(x) = \tau_{1,NK-1}(x)\\). Let's show it for every \\(d\\) by induction:
\\[
\begin{split}
    \sigma_d(x) &= \int_0^1 \sigma_{d-1}(t) \tau_{1-t,NK-d}(x - t) \,\mathrm{d}t \\\
                &= \int_0^1 \frac{(NK-1)!}{(d-2)! (NK-d)!} t^{d-2} (1-t)^{NK-d} \frac{NK-d}{1-t} \left(\frac{1-x}{1-t}\right)^{NK-d-1} \,\mathrm{d}t \\\
                &= \frac{(NK-1)!}{(d-2)! (NK-d-1)!} (1-x)^{NK-d-1} \int_0^1 t^{d-2} \,\mathrm{d}t \\\
                &= \frac{(NK-1)!}{(d-2)! (NK-d-1)!} (1-x)^{NK-d-1} \frac{x^{d-1}}{d-1}
\end{split}
\\]
The integral goes over all possible values \\(t\\) and multiplies the probability that the first \\(d-1\\) arcs have lengths that sum up to \\(t\\) and that the \\(d\\)-th arc has a length that is \\(x-t\\) (in the remaining length of the circle, which is \\(1-t\\)).

It turns out that \\(\sigma_d\\) is in fact the pdf of the [beta distribution](https://en.wikipedia.org/wiki/Beta_distribution) with parameters \\(\alpha = d\\) and \\(\beta = NK - d\\) (recall that for natural numbers \\(\Gamma(n) = (n-1)!\\)). Hence \\(S_K^{(i)}\\) is distributed as \\(Β(K, (N-1)K)\\), which is exactly what we were looking for.

As it is a very well known distribution we can just look up its properties on Wikipedia rather than computing them ourselves. We find out that the expected value of \\(S_K^{(i)}\\) is \\(N^{-1}\\) (which we already knew) and that its variance is
\\[
    \operatorname{Var}(S_K^{(i)}) = \frac{N-1}{N^2 (NK + 1)}
\\]

That's all quite interesting, but...
---

What do we make of it?

Suppose that given \\(N\\) we want to choose a \\(K\\) that guarantees that a server will receive at most \\(1+\epsilon\\) times its ideal average load with probability at least \\(1 - \delta\\). What I mean by this is: in ideal conditions every server should have markers whose covered arcs sum up to exactly one \\(N\\)-th of the circle and should thus receive one \\(N\\)-th of the load; for, say, \\(\epsilon = 0.1\\) and \\(\delta = 0.001\\), we are estimating the probability that a server stays under 110% of its ideal load with 99.9% probability.

Using Chebyshev's inequality we can figure out what values of \\(K\\) are appropriate. This will most likely be an overestimate, but it will give a simple closed-form answer. The inequality can be stated in the following form:
\\[
    \Pr(|X - \operatorname{E}(X)| \ge \epsilon \operatorname{E}(X)) \le \frac{\operatorname{Var}(X)}{\epsilon^2 \operatorname{E}^2(X)}
\\]

If we make the right-hand side smaller than \\(\delta\\) then the probability on the left-hand side will be as well, which is what we want. Let's express that condition and plug in the values of \\(\operatorname{E}(X)\\) and \\(\operatorname{Var}(X)\\):
\\[
    \frac{N-1}{\epsilon^2 (NK + 1)} \le \delta
\\]
which means that any \\(K\\) such that
\\[
    K \ge \left(1 - \frac{1}{N}\right)\frac{1}{\epsilon^2 \delta} - \frac{1}{N}
\\]
will work. We can lose the dependency on \\(N\\) by taking a slightly larger value of \\(K\\), namely
\\[
    K = \frac{1}{\epsilon^2 \delta}
\\]

In our previous example with \\(\epsilon = 0.1\\) and \\(\delta = 0.001\\) this gives us \\(K = 100000\\).

Observe that we only looked at a single server. If we want every server's load to stay within a \\(1 + \epsilon\\) factor of its ideal average with probability \\(1 - \delta'\\) then, by applying the union bound, we find out that each of them must satisfy the previous condition for \\(\delta = \frac{\delta'}{N}\\). This again is an overestimate.

Tighter bounds can be obtained. The tightest bound involves inverting the cumulative distribution function of the beta distribution. As far as I know this can only be done numerically, and therefore doesn't lead to nice closed-form expressions.
