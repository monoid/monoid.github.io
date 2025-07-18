---
title: "Baby-step-giant-step algorithm in functional encryption"
date: 2025-07-12T18:44:31+02:00
---

The [Shanks' baby-step-giant step (BSGS)
algorithm](https://en.wikipedia.org/wiki/Baby-step_giant-step) is the final
stage of certain functional encryption schemes based on elliptic curve pairings.
This article discusses how the application of BSGS in these schemes differs from the
classical one.

## Boundaries

The textbook implementations use the group's order $q$ as an input parameter.
However, it would make the functional encryption scheme infeasible, as the order of
the pairing target group is large enough to ensure cryptographic security.

The functional encryption implementations use the upper bound on the possible result
instead. For example, if our input vector has size $n$, and each element is
bounded by $M$, then the upper bound for an encrypted inner product is $n\cdot
M^2$. In case of quadratic functional encryption, if we have two vectors and a
square matrix of dimension $n$ and an upper bound for the matrix and vector
elements is $M$, then the result is bounded by $n^2\cdot M^3$. For a bidiagonal
matrix, the bound becomes $2n\cdot M^2$. It is usually much smaller than $q$
(and if it is not, we are in trouble).

The ability to use bounds much smaller than the group order is what makes BSGS a
perfect method for discrete logarithm computation in functional encryption
schemes.

## Mitigating timing attacks

In the classical implementation, execution time is proportional to the number of
giant steps, which leaks the higher-order bits of the result. While this is
largely irrelevant for breaking a cipher, it can be critical for functional
encryption. A constant-time implementation is possible, but it would be slow.
Consider a timing-resistant implementation that has variable execution time, but
from which an attacker won't be able to learn much.

**Idea 1.** We could build the giant step table and iterate over baby steps. If
we iterate over baby steps, only the lower half of the bits will be available to
a timing attacker. Usually, it is less useful, but we can hide those as well.

However, it will make execution slightly slower. If the input values have a
uniform distribution, the product doesn't. The classical version starts with
smaller values, which are more probable, but unfortunately, the same feature
makes timing attacks easier.

**Idea 2.** If we iterate over baby steps in random order, the execution time
will not correlate with the result. It can be achieved by precomputing and
shuffling the baby-step array. It doubles the memory usage and initialization
time, but it is a fair price for improved timing attack resistance.

We can keep both the giant-step hash table and the baby-step array in persistent
memory. They contain no secret information, can be shared between parties, and
only need to be regenerated when the elliptic curve parameters (including generator)
or bounds change, which should happen rarely.

Maximum and average execution times will be almost the same as those of Idea 1.
Our benchmarks show that memory cache effects from sequential array access are
negligible compared to the cost of multiplication in the pairing target group.

## Disclaimer

This text was written by Ivan Boldyrev. I used AI only for proofreading, but its
help was invaluable.
