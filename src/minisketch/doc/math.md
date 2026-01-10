# The mathematics of Minisketch sketches

This is an unconventional mathematical overview of the PinSketch algorithm without references to coding theory[\[1\]](math.md#myfootnote1).

## Set sketches

A sketch, for the purpose of this description, can be seen as a "set checksum" with two peculiar properties:

* Sketches have a predetermined capacity, and when the number of elements in the set is not higher than the capacity, minisketch will always recover the entire set from the sketch. A sketch of _b_-bit elements with capacity _c_ can be stored in _bc_ bits.
* The sketches of two sets can be combined by adding them (XOR) to obtain a sketch of the [symmetric difference](https://en.wikipedia.org/wiki/Symmetric_difference) between the two sets (_i.e._, all elements that occur in one but not both input sets).

This overview explains how sets can be converted into a sketch and how a set can be recovered from a sketch.

## From field elements to sketches

**Data entries as field elements**

Every integer in the range _\[1...2b-1]_ (the acceptable data elements for a Minisketch sketch with field size _b_) can be mapped to a nonzero field element of _GF(2b)_. In this [finite field](https://en.wikipedia.org/wiki/Finite_field), we can add and multiply elements together, with many of the expected properties for those operations. Addition (and subtraction!) of field elements corresponds to bitwise XOR of the integers they correspond to, though multiplication is more involved.

**Sets as power series**

We define a function _S_ which maps field elements _m_ to the following [formal power series](https://en.wikipedia.org/wiki/Formal_power_series) (similar to a polynomial, except there can be an infinite number of terms, and we don't care about concepts like convergence as we're never going to actually evaluate it for a specific value of _x_):

* _S(m) = 1 + mx + m2x2 + m3x3 + ..._.

We then extend this function to operate on sets of field elements, by adding together the images of every set element. If _M = {m1, m2, ... }_:

* _S(M) = S({m1,m2,...}) = S(m1) + S(m2) + ... = (1 + 1 + ...) + (m1 + m2 + ...)x + (m12 + m22 + ...)x2 + (m13 + ..._

Because in our field addition corresponds to XOR of integers, it holds for every _a_ that _a + a = 0_. This carries over to the _S_ function, meaning that _S(a) + S(a) = 0_ for every _a_. This means that the coefficients of these power series have the second of the properties we desire from a sketch, namely that an efficient operation exists to combine two sketches such that the result is a sketch of the symmetric difference of the sets. It holds that _S({m1,m2}) + S({m2,m3}) = S(m1) + (S(m2) + S(m2)) + S(m3) = S(m1) + S(m3) = S({m1,m3})_. The question is whether we can also efficiently recover the elements from their power series' coefficients.

**An infinity of coefficients is hard**

To make reasoning about these power series easier, notice that the series for a single element is in fact a [geometric series](https://en.wikipedia.org/wiki/Geometric_series). If we were working over real numbers rather than a finite field and _|mx| < 1_, it would converge to _(1 - mx)-1_. Convergence has no meaning in formal power series, however it is still the case that:

* _(1 - mx) S(m) = 1_

You can verify this by seeing that every coefficient except the constant one gets cancelled out by the multiplication. This can be generalized to the series for multiple set elements. For two elements we have:

* _(1 - m1x) (1 - m2x) S({m1,m2}) = (1 - m1x) (1 - m2x) (S(m1) + S(m2)) = (1 - m2x) + (1 - m1x)_

And for three:

* _(1 - m1x) (1 - m2x) (1 - m3x) S({m1,m2,m3}) = (1 - m1x) (1 - m2x) (1 - m3x) (S(m1) + S(m2) + S(m3)) = (1 - m2x)(1 - m3x) + (1 - m1x)(1 - m3x) + (1 - m1x)(1 - m2x)_

In each case, we notice that multiplying _S(M)_ with _(1 - mix)_ for each element _mi ∈ M_ results in a polynomial of degree _n-1_.

**Solving for the set elements**

The above insight lets us build a solver that extracts the set elements from the coefficients of a power series. If we can find a polynomial _L_ that is the product of _n_ different _(1 - mix)_ factors for various values of _mi_, such that _P = S(M)L_ is an _n-1_ degree polynomial, then those values _mi_ are the elements of _M_.

The coefficients of _P_ are nontrivial expressions of the set elements themselves. However, we can just focus on the coefficients of degree _n_ and higher in _P_, as those are all 0. Let _si_ be the coefficients of _S(M)_, and _li_ the coefficients of L. In other words, _S(M) = s0 + s1x + s2x2 + s3x3 + ..._ and _L = l0 + l1x + l2x2 + l3x3 + ... + lnxn_. Note that _l0 = 1_, as it is the product of all the _1_ terms in the _(1 - mix)_ factors.

Here are the equations for the coefficients of _S(M)L_ of degree _n+1_ through _2n_:

* _sn+1 + sn+0l1 + sn-1l2 + sn-2l3 + ... + s1ln = 0_
* _sn+2 + sn+1l1 + sn+0l2 + sn-1l3 + ... + s2ln = 0_
* _sn+3 + sn+2l1 + sn+1l2 + sn+0l3 + ... + s3ln = 0_
* ...
* _s2n + s2n-1l1 + s2n-2l2 + s2n-3l3 + ... + snln = 0_

These are _n_ linear equations with _n_ unknowns (the _li_ values, for _i=1..n_), which can be solved using [Gaussian elimination](https://en.wikipedia.org/wiki/Gaussian_elimination). After doing so, we have the coefficients of _L_, which can then be [factored](https://en.wikipedia.org/wiki/Factorization_of_polynomials_over_finite_fields) into first degree factors of the form _(1 - mix)_. The resulting _m_ values are our set elements.

**Putting it all together**

Interestingly, only _2n_ coefficients of _S(M)_ were needed for solving the set of equations above. This means we have our answer: the coefficients _1_ through _2n_ of _S(M)_, or the list _\[m1 + m2 + ..., m12 + m22 + ..., ..., m12n + m22n + ...]_ functions as a sketch, satisfying the two properties we want:

* Sketches can be combined to form the sketch of their symmetric difference, by simply pairwise adding the list elements together.
* With _2n_ list elements we can efficiently recover _n_ elements from a sketch.

**Capacity and difference**

The approach above only works when the number of elements _n_ in the sketch is known. Of course we want to support cases where only an upper bound on the number of elements in the sketch is known, the capacity _c_. Given that we can reconstruct a set of size _c_ from a sketch with _2c_ terms, we should be able to reconstruct a set of size _n_ too as long as _n ≤ c_. This is simply a matter of trying to solve the above set of equations assuming values of _n_ that count down from _c_ until a solution is found for one. This is known as the [Peterson-Gorenstein-Zierler algorithm](https://en.wikipedia.org/wiki/BCH_code#Peterson%E2%80%93Gorenstein%E2%80%93Zierler_algorithm).

## Optimizations

**Halving the sketch size**

We can in fact only include the odd terms in the sketch, and reconstruct the even ones before solving the equation to find _L_. This means the size of a sketch becomes just _c_ field elements, the same size as would be needed to send its contents naively.

To see how this is possible, we need the [Frobenius endomorphism](https://en.wikipedia.org/wiki/Frobenius_endomorphism), which in short states that in fields where _x + x = 0_ it holds that _(x + y)2 = x2 + y2_ for every _x_ and _y_ (the dream of every high school math student!). This means that:

* _s2 = m12 + m22 + ... = (m1 + m2 + ...)2 = s12_.
* _s4 = m14 + m24 + ... = (m12 + m22 + ...)2 = s22_.
* _s6 = m16 + m26 + ... = (m13 + m23 + ...)2 = s32_.
* ...

In other words, we only need to send _s1, s3, s5, ..., s2n-1_ to recover all _2n_ _si_ values, and proceed with reconstruction.

**Quadratic performance rather than cubic**

Using Gaussian elimination to solve the set of equations above for the _li_ values requires _O(n3)_ field operations. However, due to the special structure in the equations (look at the repeated _si_ values), it can be solved in _O(n2)_ time using a number of techniques, including the [Berlekamp-Massey algorithm](https://en.wikipedia.org/wiki/Berlekamp%E2%80%93Massey_algorithm) (BM).

**Roots instead of factorization**

As explained above, the polynomial _L_ can be factored into _(1 - mix)_ factors, where the values _mi_ are the set elements. However, since we know that a decodable sketch must result in a polynomial that is fully factorizable into degree-_1_ factors, we can instead use a more efficient root-finding algorithm rather than a factorization algorithm. As the root of each _(1 - mix)_ factor is _mi-1_, we conclude that the set elements are in fact the inverses of the roots of _L_.

**Avoiding inversions**

As inversions are a relatively expensive operation, it would be useful to avoid them.

Say that we're trying to find the inverses of the roots of _L = 1 + l1x + l2x2 + ... + lnxn_, then we're really interested in the solutions _y_ for _1 + l1y-1 + l2y-2 + ... + lny-n = 0_. By multiplying both sides in the equations with _yn_, we find _ln + ln-1y + ln-2y2 + ... + yn = 0_.

In other words, we can find the inverses of the roots of _L_ by instead factoring the polynomial with the coefficients of _L_ in reverse order.

* \[1] For those familiar with coding theory: PinSketch communicates a set difference by encoding the set members as errors in a binary [BCH](https://en.wikipedia.org/wiki/BCH_code) codeword 2bits in size and sends the syndromes. The linearity of the syndromes provides all the properties needed for a sketch. Sketch decoding is simply finding the error locations. Decode is much faster than an ordinary BCH decoder for such a large codeword because the need to take a discrete log is avoided by storing the set in the roots directly instead of in an exponent (logically permuting the bits of the codeword).
