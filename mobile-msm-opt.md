# MSM optimization on mobile    

## 1. Use Twisted Edwards to do the add_mix/add

A important feature of Elliptic Curves is that they are often isomorphic and can be transformed to another curve. For example the general curve form is Weierstrass Form: 

$y^2 + a_1xy + a_3y = x^3 + a_2x^2 + a_4x+a_6$. 

It can be transformed to Short Weierstrass Form:

$y^2=x^3+ax+b$ 

when characteristic of $\mathbb{F}_p$​ is not 2 or 3 . This transformation method is called Curve Isogeny.

The point addition on Short Weistrass Curves costs 11M+5S according to http://www.hyperelliptic.org/EFD/g1p/auto-shortw-jacobian-0.html#addition-add-2007-bl. Mixed point addition costs 7M+4S according to http://www.hyperelliptic.org/EFD/g1p/auto-shortw-jacobian-0.html#addition-madd-2007-bl.

Another well known curve form is Twisted Edwards Curves, introduced by Daniel Bernstein et al. The curve formula is depicted as below:

$ax^2+y^2=1+dx^2y^2$

It has the fastest point addition by far and we can have unified point addition formula. The unified addition costs 8M+1k according to http://www.hyperelliptic.org/EFD/g1p/auto-twisted-extended-1.html#addition-add-2008-hwcd-3, while mixed addition costs 7M+1k according to http://www.hyperelliptic.org/EFD/g1p/auto-twisted-extended-1.html#addition-madd-2008-hwcd-3. The fastest algorithm requires another field to record $t = x*y$ though, as described in a paper https://iacr.org/archive/asiacrypt2008/53500329/53500329.pdf.

An natural idea of optimizing MSM is to utilize curve isogeny, which means we transform all the points on BLS12-377 to its isomorphic Twisted Edwards curve, run MSM on that curve and finalize the result, then transform from Twisted Edwards curve back to BLS12-377. We can have all the nicely features of Twisted Edwards curve, while retaining the general Short Weierstrass curve representation.

Not all Short Weierstrass Curves can be transformed to Twisted Edwards Curves. According to https://en.wikipedia.org/wiki/Montgomery_curve, we know that under certain conditions, a Short Weierstrass Curve is isomorphic to a Montgomery Curve, and a Montgomery Curve is isomorphic to a Twisted Edwards Curve. In order to transform from Short Weierstrass to Montgomery form, We quote:

> In contrast, an elliptic curve over base field $\mathbb{F}$ in Weierstrass form $E_{a,b}: v^2 = t^3+at+b$ can be converted to Montgomery form if and only if has order divisible by four and satisfies the following conditions:
>
> 1. $z^3 +az+b=0$ has at least one root $\alpha \in \mathbb{F}$; and
> 2. $3\alpha^2+a$ is a quadratic residue in $\mathbb{F}$
>
> When these conditions are satisfied, then for $s = (\sqrt{3\alpha^2+a})^{-1}$ we have the mapping
>
> $\psi^{-1}: E_{a,b} \rightarrow M_{A, B}$
>
> $(t, v) \mapsto (x, y)=(s(t-\alpha, sv)), A = 3\alpha s, B=s$

Then from Montgomery to Twisted Edwards, we quote:

> $2\frac{a+d}{a-d} =A$
>
> $\frac{4}{a-d}=B$ 
>
> $(u, v) \mapsto (u/v, (u-1)/(u+1))$

and that's it!

Very luckily, BLS12-377 can be transformed. The Short Weierstrass Form equation of BLS12-377 is: 

$y^2 = x^3 +1$, with $a = 0, b=1, q = 258664426012969094010652733694893533536393512754914660539884262666720468348340822774968888139573360124440321458177$. 

We choose

$A=30567070899668889872121584789658882274245471728719284894883538395508419196346447682510590835309008936731240225793$

$B=113327392486723791340039350366245770860783363395096105577549129978801514727083865406602966047408507739599254478215 $

for its Montgomery isogeny $M_{A, B}$, then transform to $E_{a, d}$ with

$a=157163064917902313978814213261261898218646390773518349738660969080500653509624033038447657619791437448628296189665 $

$d=101501361095066780517536410023107951769097300825221174390295061910482811707540513312796446149590693954692781734188$.

We can even optimize further by making $a=-1$ as depicted by Huseyin Hisil et al in https://iacr.org/archive/asiacrypt2008/53500329/53500329.pdf. The process is rather simple, we are not going to demonstrate it here.

So the point addition optimization is rather straight forward here. First we transform all the base points which are on BLS12-377 to another set of base points on a Twisted Edwards curve that is isomorphic to BLS12-377. Second we do the MSM (mainly operations are point additions/mixed point additions) on it and get an accumulated point. Finally we transform that single point to BLS12-377 again.

Since MSM is mostly used to commit to a polynomial, those base points are always fixed. This optimization approach should be valid for all ZKP systems, as long as the curve transformation is valid.

## 2. Tune the pippenger window size

We find the best window size is `ln_without_floats(size) + 1` on the target device.
