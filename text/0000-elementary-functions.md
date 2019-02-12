- Feature Name: elementary function calculation
- Start Date: 2019-02-11
- RFC PR: (leave this empty)
- Authors:
  - Matthias Güdemann (matthias.gudemann@iohk.io)

# Summary
[summary]: #summary

The delegation mechanism uses a decay function for refunds which involves
calculating the exponential function. The moving average of the performance of a
stake pool is calculated using exponentiation of two fractional numbers.

This proposal addresses the method of how to calculate the exponential function,
the natural logarithm and the exponentiation using a non-integer exponent.

# Motivation
[motivation]: #motivation

The delegation requires some non-integral calculations using elementary
functions. These calculations have to be approximated and are not standardized
in general.

In order to prevent forks because of different results on different
implementations or architectures, we provide our own implementation of the
necessary approximation functions.

Assuming the same representation of non-integral values and standardized basic
arithmetic operations, this leads to equivalent results using different
implementation languages or CPU architectures.

# Decision
[Decision]: #decision

The library `non-integer` exports the functions `exp'` which approximates the
exponential function, `ln'` which approximates the natural logarithm and the
infix operator `(***)` which approximates exponentiation.

Current implementation: `dependencies/non-integer` of `fm-ledger-rules@727cca6`

All input parameters for the functions need to support addition, subtraction,
multiplication and division. The approximation algorithm terminates if the basic
operations terminate. The approximation is incremental, bounded by both a fixed
constant and the precision of succeeding incremental approximations.

Exponentiation `x^y` is approximated via `x^y = exp(ln(x^y)) = exp(y*ln(x))`,
using `exp'` and `ln'`.

# Technical explanation
[technical-explanation]: #technical-explanation

The core of the approximation of `exp` and `ln` are continued fractions. For
each function, there exists a continued fraction representation (e.g.,
[log](http://functions.wolfram.com/ElementaryFunctions/Log/10/),
[exp](http://functions.wolfram.com/ElementaryFunctions/Exp/10/)). Continued
fractions can easily be calculated recursive using via a recurrence
relation which results in an intermediate approximation after each step.

Calculating continued fractions in pseudo code,

```
// let a be the vector of the partial numerators, a[i] the i-th partial numerator
// let b be the vector of the partial denomiators, b[i] the i-th partial denominator

// result: vectors A / B where A[n]/B[n] is the n-th convergent for n >= 0

// initialize
A[-1] := 1
B[-1] := 0
A[0]  := b_0
B[0]  := 1

n := 0

while(not_precise_enough())
  A[n+1] := b[n+1] * A[n] + a[n+1] * A[n-1]
  B[n+1] := b[n+1] * B[n] + a[n+1] * B[n-1]
  n := n + 1

return (A[n]/B[n])

```

Continued fractions do have advantages over finite Taylor / MacLaurin polynomial
approximations:

* For `ln` the Taylor series converges very badly and has a very high
  approximation error even within the interval [1,2].
* Taylor approximation required calculation of large factorials

The convergence for both approaches is best in small domains, therefore we use
mathematical transformations to scale the input parameters accordingly.

For `exp'` we use the fact that for an integer `n` we have `e^x = e^(x/n)^n` and
that `e^(-x)=1/e^x`. We therefore ensure that the parameter `y = x/n`for the
approximation is non-negative and scale it to [0,1] via `y = x/ceiling(x)`. The
final integer exponentiation can be done using the fast integer exponentiation
algorithm.

For `ln'` the continued fraction approximation is best for `[1,c]` with some
small `c`. We first calculate an integer `n` such that `e^n <= x < e^(n+1)`
(i.e., `1 <= x/(e^n) < e`) using bisection. Using the identity `ln(x*y) =
ln(x) + ln(y)` we get `ln(x) = ln(e^n * x / e^n) = ln(e^n) + ln(x/e^n)` where
`ln(e^n) = n` and `x/e^n` is an element of `[1,e)`, for which the approximation
converges well.

# Drawbacks/Implications
[drawbacks]: #drawbacksimplications
[implications]: #drawbacksimplications

The integer exponentiation at the end of the `exp'` approximation might
introduce a higher approximation error than expected. On the other hand, We use
the function for decay where large parameters lead to very close to full decay, and we could simply specify a cut-off after which the result is 0.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

- The more standard way for the logarithm seems to be to calculate the binary
  logarithm and then use a pre-calculated constant `ln(2)` to scale the result
  accordingly. I do not see any advantage of this apart from being able to use
  shifts instead of multiplications, i.e., increased efficiency.

- Taylor series approximation of `ln` seem to converge extremely slow, continued
  fractions have been shown to be much faster in experiments. See below for
  results (finite Taylor polynomial vs. non-scaled continued fraction).

  The following graphs show the relative error `|exp'(x) - exp(x)|/|exp(x)|` and
  `|ln'(x) - ln(x)|/|ln(x)|`:

  Finite Taylor series with 10/20/50 iterations for `ln`:
![Taylor Series Approximation of `ln` (10,20,50 Iterations)](ln_taylor.png)

  Continued Fraction Approximation with 10/20/50 iterations for `ln`:
![Continued Fraction Approximation of `ln` (10,20,50 Iterations)](ln_cf.png)

  For the exponential function, the difference is less pronounced, but still
  significant

  Finite Taylor series with 10/20 iterations for `exp`:
![Taylor Series Approximation of `exp` (10,20 Iterations)](taylor_exp.png)

  Continued Fraction Approximation with 10/20 iterations for `exp`:
![Continued Fraction Approximation of `exp` (10,20,50 Iterations)](cf_exp.png)

- Using scaling as above, we might also be able to get good convergence using a
  finite Taylor / MacLaurin series approximation.

# Prior art
[prior-art]: #prior-art

Continued fractions and scaling are a standard approach to approximate
elementary functions and are also used in verification to give upper or lower
bounds.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

- The main related issue is which representation of the numbers to choose. For
  the implementation any `RealFrac` can be used, but the precision and
  significant digits will vary accordingly.

- What should be clarified is the way to fix the recursion depth of the
  approximations and which additional tests for soundness apart from the
  existing ones are required.
