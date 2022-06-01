---
layout: post
title: "Notes on point compression with P384/secp384r1"
date: 2022-06-01
---
<script type="text/javascript" async
  src="https://cdn.mathjax.org/mathjax/latest/MathJax.js?config=TeX-MML-AM_CHTML">
</script>

Public keys represent a point on the curve, given a pair of ($x$,$y$) coordinates. In uncompressed form, a public key when encoded as bytes ([1], sec 2.3.4) it would be with the identifier $4$ prepended to both coordinates: `pk := 0x04 || x || y`.

However, it's not required to store the $y$-coordinate in order the use the public key. The compressed form of a public key leaves out $y$ and adds a sign instead, indicating whether $y$ was even or odd.

$y$ is _even_: `pk := 0x02 || x` 

$y$ is _odd_: `pk := 0x03 || x`

This allows us to essentially cut the length required to store a public key in half. In order to use a public key in compressed form for any operations, such as signature verification, we have to recover $y$.


The P384 curve (also known as "secp384r1"), is a prime order short Weierstrass curve with the following defined:

$E: y^2 = x^3 + ax + b$

$\mathbb{Z}_p$, where $p = 2^{384} - 2^{128} - 2^{96} + 2^{32} - 1$ (all operations on the curve are modulo the prime $p$)

$a = -3$ (the same as $p - 3$, so usually precomputed when used in implementations)

$b =$ `b3312fa7 e23ee7e4 988e056b e3f82d19 181d9c6e fe814112 0314088f 5013875a c656398d 8a2ed19d 2a85c8ed d3ec2aef`

<details>
    <summary>Note</summary>
b is the constant that nobody really knows how/why was chosen. It is a SHA-1 hash of some NSA-chosen input.
</details>


To recover $y$, we start be solving the polynomial (right-hand side of the curve's equation) by using the defined constant and the $x$-coordinate provided by the compressed public key:

$y^2 = x^3 + (a * x) + b$

In order to get $y$, we need to compute the square root of $y^2$. This is not a straight-forward $\sqrt{y2}$ as it needs to be done modulo the prime. A general algorithm that can be used for this is Tonelli-Shanks ([2]) which works when we have a non-zero $n$ ($y^2$ in this case) and an odd prime $p$ (which is also true for secp384r1).

First we have to check if $y^2$ has a square root at all. Euler's criterion states that ([3]): 

$n^{\frac{p-1}{2}} \equiv 1$ (mod p), then $n$ has a square root

$n^{\frac{p-1}{2}} \equiv -1$ (mod p), then $n$ _does not_ have a square root

This can be checked using the Legendre symbol ([4]) (or Jacobi symbol). The only difference to the Legendre symbol is that we define $0$ as the result if $n^{\frac{p-1}{2}} \equiv 0$ (mod p). In other words, we are checking whether or not $y^2$ is a quadratic residue. If the Legendre symbol of $y^2$ is anything but $1$, we cannot continue. If it is one, it means there is a square root and we can find it using Tonelli-Shanks.

There is a faster way to compute the square of $y^2$ in the case of secp384r1, in that we don't need the entire Tonelli-Shanks algorithm, just the "easy case". Because $p \equiv 3$ (mod $4$) holds true for the prime $p$ in secp384r1, the square root can be computed by $y = \sqrt{y^2} = y^2{^{\frac{p+1}{4}}}$. The only thing remaining is checking if $y$ is odd and the first byte of the public key is `0x03`, or if $y$ is even and the first byte of the public key is `0x02`. If this is not true, then set $y = p - y$. This leaves us with the original $y$-coordinate that was shaved off in the beginning. 


In implementations, you can expect to find values like $\frac{p+1}{4}$ and $\frac{p-1}{2}$ precomputed.


#### References

- \[1\]: https://www.secg.org/sec1-v2.pdf
- \[2\]: https://en.wikipedia.org/wiki/Tonelli%E2%80%93Shanks_algorithm
- \[3\]: https://en.wikipedia.org/wiki/Euler%27s_criterion
- \[4\]: https://en.wikipedia.org/wiki/Legendre_symbol