---
title: "Polonius and the Secret of the Eliminated Quantifier"
date: 2021-04-30T14:37:41-07:00
katex: true
summary: "A possible solution to Polonius' higher-rank trait bound problem."
---

I've long been intrigued by [Polonius][], an improved formulation of the borrow
checker inspired by more traditional alias analyses.
The algorithm that became Polonius was first described by Niko Matsakis in ["An
alias-based formulation of the borrow checker"][niko-polonius], and has since
been implemented [out-of-tree][Polonius].

The project is currently on hiatus, however, and resumption does not seem
imminent. One roadblock is Polonius' inability to reason about [higher-rank
trait bounds (HRTBs)][nomicon-hrtb]. It's pretty rare for users to encounter
these, but they're an integral part of the language that appear when desugaring
some `Fn` trait bounds. This shortcoming was [described in detail][niko-hh] on
Niko's blog over 2 years ago in his last post on Polonius. I suggest reading
that post's immediate predecessor, [a more general discussion of placeholder
region errors][niko-placeholder], as well.

As far as I know, there is no accepted solution to these problems. This post
will explore one possibility, an algorithm for quantifier elimination in a
fragment of first-order logic on partially ordered sets. The idea (if not the
nomenclature) is pretty simple, but I haven't seen it discussed in any of the
usual forums.[^forums] This is somewhat surprising, since it appears to solve
the problems around HRTBs in a straightforward way.

## Quantifier elimination

In formal logic, the process of rewriting a formula that includes quantifiers
into an equivalent one without them is known as [quantifier elimination][qe]
(QE). Certain theories are said to have QE, which means that all formulas can
be rewritten this way, while others do not. For example, the real numbers with
addition, multiplication, equality, and strict ordering have QE,[^cad] but not
the integers with the same operators. If you disallow multiplication between
integer variables, however, you get [Presburger arithmetic][], which *does*
allow for QE.

Our goal, then, is to find a QE algorithm for the "theory" of placeholder
regions used by Polonius. Clearly, this will depend on both the domain of our
theory (regions), as well as the operations allowed on those regions.

## Regions are sets of loans

Niko's blog posts lay out the requirements for Polonius' placeholder region
logic. As he explains it, "regions are sets of loans", so our domain is that of
sets. However, placeholder regions are somewhat different than the sets of
loans we encounter when checking function bodies. Inside a function, there is a
finite number of loans, each corresponding to a borrow (`&` or `&mut`) in the
source.  Placeholder regions are more abstract in some sense; relationships
between placeholder regions in a function signature must hold across all
possible loans in all possible callers of that function. Therefore, we will
treat placeholder regions as if they may contain infinitely many loans, the set
of all loans that could be created in any valid Rust program.[^infinite-programs]

Sets, when ordered by inclusion, are an example of a common algebraic structure
called a [partially-ordered set or poset][poset]. If we extend our regions with
the canonical set operators---union ($\cup$) and intersection ($\cap$)---they
form a [lattice][]. Lattices are another common structure, and we
will use results from both to develop our QE algorithm. Our
lattice has a global lower bound, the empty set ($\empty$), but it has no
global upper bound because it is infinite. As Niko's posts show, this global
lower bound corresponds to the special `'static` lifetime in Rust.[^static]

The fundamental operation on these sets is the subset relation
($\sub$), which requires that the loans in one region also be present in
another. We are allowed to have many of these relations, all of
which must hold. This is logical conjunction ($\land$, sometimes abbreviated
with a comma).

And...That's it! You can combine these operators to express, for example,
equality between regions ($a \sub b \land b \sub a \implies a = b$), but these
primitives are all that's required for basic placeholder region
constraints. This is the logic we will extend with quantification.

## Existential quantification

We're ready to add existential quantification ($\exists x$) to the mix. I'll
use the approach from Lemma 2.4 in [this paper by Peter Revesz][revesz-elim] to
demonstrate an algorithm for QE. Revesz shows quantifier elimination for a
slightly more general theory,[^boc] one that allows for inequality constraints
with the empty set, but we won't need that here. First, I'll write a general
form for all possible existentially quantified formulas in our theory.

<p>
\begin{align*}
\exists x (
    &y_1 \sub x, \dots, y_m \sub x,          \\
    &x \sub z_1, \dots, x \sub z_n,          \\
    &u_1 \sub v_1, \dots, u_s \sub v_s)
\end{align*}
</p>

Here  $y$, $z$, $u$, and $v$ are collections of region variables distinct from
the existentially quantified variable $x$ but not necessarily from each other.
$y$ is a series of lower bounds on $x$, $z$ is a series of upper bounds, and
$u$ and $v$ are unrelated constraints.

Because $u$ and $v$ have no relation to $x$, we can move them outside of the
scope of the existential quantifier and ignore them for now. We'll rewrite the
remaining constraints as a single upper and lower bound on $x$ using the
following lattice identities for sets:

<p>
\begin{equation}
b \sub a \land c \sub a \iff (b \cup c) \sub a
\end{equation}
\begin{equation}
a \sub b \land a \sub c \iff a \sub (b \cap c)
\end{equation}
</p>

By applying these identities repeatedly to the lower and upper bounds, we can
express our existential quantifier as a single pair of subset constraints, one
on the union of the lower bounds and another on the intersection of the uppers.

$$
\exists x (\bigcup\limits_{i=1}^{m} y_i \sub x, x \sub \bigcap\limits_{j=1}^{n} z_j)
$$

If either $y$ or $z$ are empty---i.e., there are no upper (or lower) bounds---the
formula is trivially satisfiable by setting $x$ to the intersection of the
upper bounds or the union of the lower bounds respectively. Otherwise, we can
eliminate the quantified variable by applying the transitive property
of the subset operator ($(a \sub b) \land (b \sub c) \implies a \sub c$) and
then apply lattice identities $(1)$ and $(2)$ in reverse to get an equivalent
formula containing only subset constraints.

<p>
\begin{align*}
\exists x (\bigcup\limits_{i=1}^{m} y_i \sub x, x \sub \bigcap\limits_{j=1}^{n} z_j)
&\iff \bigcup\limits_{i=1}^{m} y_i \sub \bigcap\limits_{j=1}^{n} z_j \\
&\iff y_1 \sub \bigcap\limits_{j=1}^{n} z_j, \dots, y_m \sub \bigcap\limits_{j=1}^{n} z_j \\
&\iff \begin{array}{ccc}
y_1 \sub z_1, & \dots, & y_1 \sub z_n, \\
\vdots & \ddots & \vdots \\
y_m \sub z_1, & \dots, & y_m \sub z_n \\
\end{array}
\end{align*}
</p>

By adding our unrelated constraints back in, we obtain an equivalent quantifier
free formula for any existentially quantified one:

<p>
\begin{align*}
\begin{alignat*}{2}
\exists x (
    &y_1 \sub x, \dots, y_m \sub x,          \\
    &x \sub z_1, \dots, x \sub z_n,          \\
    &u_1 \sub v_1, \dots, u_s \sub v_s)
\end{alignat*}
\iff
\begin{array}{ccc}
    y_1 \sub z_1, & \dots, & y_1 \sub z_n, \\
    \vdots & \ddots & \vdots \\
    y_m \sub z_1, & \dots, & y_m \sub z_n, \\
    u_1 \sub v_1, & \dots, & u_s \sub v_s
\end{array}
\end{align*}
</p>

## Universal quantification

If our theory had logical negation ($\neg$) as well as existential QE, we could
stop here. It would be straightforward to eliminate universal quantifiers
($\forall$) by using the following identity to transform them into existential
ones.

$$
\forall x P(x) \implies \neg (\exists x \neg P(x))
$$

Unfortunately, our theory does not permit logical negation. In fact, as I'll
show later, it *cannot* (as long as we want QE). Luckily, due to the limits
we've imposed, we can construct a separate algorithm for eliminating universal
quantifiers.  We will take the same approach as for existential quantifiers by
splitting the bounds on a quantified variable into upper and lower ones and
considering them separately.

### Upper bounds

Upper bounds on universally quantified variables are never satisfiable. For
example, if we have a constraint like $\forall x (x \sub z)$ for some free
region $z$, we can always find a loan $L$ such that $L \notin z$ because the
set of loans is infinite. The set $\\{ L \\} \cup z$ provides a counterexample
to our quantified formula, so any universally quantified formula with an upper
bound on the quantified variable is unsatisfiable ($\bot$).

### Lower bounds

Lower bounds, however, *are* satisfiable due to the existence of a global lower
bound for regions. This is $\empty$ or, as we discussed earlier, `'static`.
Given a constraint such as the following,

$$\forall x (y \sub x)$$

we can substitute $\empty$ for $x$ to obtain the constraint $y \sub \empty$ or
equivalently $y = \empty$. The resulting constraint is both necessary ($\empty$
is a valid region), and sufficient ($\forall x (\empty \sub x)$) for the
original formula to hold and is therefore equivalent.

Universal quantification distributes over conjunction, so we can apply this
transformation across multiple constraints to obtain a universal QE algorithm for
conjunctions of subset constraints.

<p>
\begin{align*}
\forall x (y_1 \sub x, \dots, y_n \sub x) &\iff \forall x (y_1 \sub x), \dots, \forall x (y_n \sub x) \\
    &\iff y_1 \sub \empty, \dots, y_n \sub \empty
\end{align*}
</p>

Remember, we only need to handle lower bounds on the existential: any upper
bound would make the entire formula unsatisfiable.

## Putting it all together

By repeatedly applying these QE algorithms to the innermost quantifier, we can
simplify arbitrarily complex quantified formulae to equivalent unquantified
ones, emit an error if any of them are unsatisfiable and pass the resulting
constraints to Polonius. Coupled with one of the approaches to emitting
placeholder region errors laid out on [Niko's blog][niko-placeholder], this
should be enough to handle any lifetime constraint that can be expressed in
Rust.

Let's try out a few test cases just to be sure. We'll start with the example
`forall<'x, 'y> { 'x: 'y }` from Niko's blog. Our algorithm would show this to
be unsatisfiable by the following process:

<p>
\begin{align*}
\forall x \forall y (x \sub y)
&\implies \forall x (x \sub \empty) \\
&\implies \bot
\end{align*}
</p>

We begin by processing the innermost quantifier, which has $x$ as a lower bound
on the quantified variable $y$. This can be reduced to the constraint $x \sub
\empty$ via the algorithm for universal QE. But when we evaluate the second
quantifier, we see an upper bound on the quantified variable $x$, so this
formula is false.

Another interesting example comes from [rust-lang/rust#65232][issue-65232]:

$$
\exists a \forall b (b \sub a) \implies \bot
$$

The existing placeholder region logic needs a special-case to handle this type
of formula.[^leak-check] This is because formulations of the borrow checker
based on program locations contain both a global upper bound (`'static`, all
possible program locations) and a global lower bound (the [empty
region][re-empty]). While the former is a valid solution to a constraint
problem, the latter results in an error. The process is more elegant in our new
system based on sets of loans, which does not have a global upper bound; the
innermost quantifier is trivially false.

## Negation, disjunction, and implication

It's not all sunshine and roses, though. In the framework I've described above,
the presence of QE is mutually exclusive with logical negation.
Logical negation would allow us to express a **strict** subset relation ($\neg
(b \sub a) \to a \subset b$), as well as inequality constraints between
arbitrary regions. To see why this is problematic, consider the following
formula.

$$
\exists x (a \subset x \land x \subset b)
$$

It may look like we can eliminate $x$ via the transitive property as we did
above, resulting in a similar constraint $a \subset b$. However, this does not
hold when $b$ has exactly one more element than $a$. There's no "room" between
$a$ and $b$ to fit an extra element, so the formula is unsatisfiable in that
case. In reality, we would need an additional constraint on the cardinality of
$a$ and $b$:

$$
\exists x (a \subset x \land x \subset b)
    \implies (a \subset b) \land (|b| - |a| > 1)
$$

Surprisingly, if we extend our theory with these so-called "cardinality
constraints", it does have QE.[^cardinality] But these constraints, which have
no analogue in Rust, would leak into the borrow-checking parts of Polonius
where I have no idea how to handle them.

Unfortunately, without negation we cannot have implication, one of
Niko's hereditary Harrop clauses. Implication reduces to a combination of
negation and disjunction and would therefore require both of these extensions.

$$
(P \to Q) \iff (\neg P \lor Q)
$$

I'm somewhat more hopeful about incorporating disjunction ($\lor$) into
Polonius, although I've not made much effort in that direction. In the worst
case, we could allow it only outside of qualifiers, akin to unions of
conjunctive queries. It's also possible that a QE algorithm exists that can
handle disjunction in any position.

As far as I can tell, however, neither of these faculties are needed to express
the constraints available in current versions of Rust. Niko alludes to the fact
that richer constraints will be useful down the line, but perhaps the existence
of a simple QE algorithm outweighs this concern?

## Final thoughts

As I promised at the outset, the ideas I've presented here are all pretty
simple.  None involve anything beyond basic set theory. In fact, I have a
sneaking suspicion that one or both of these algorithms were known to Niko when
he wrote the instigating blog post; the examples he chose seem too
prescient otherwise. If that's true, I'm very curious to know why he omitted
them. Perhaps I've overlooked something?

In any case, I'm hopeful that at least some of these ideas are new to the
Polonius working group and can be used to formalize Rust's placeholder region
rules. If not, perhaps the more formal treatment and parallels with Revesz's
work will provide others with inspiration; I'd love to see Polonius move closer
to completion.

[^cad]: albeit in double exponential time via an algorithm known as [cylindrical
  algebraic decomposition][].

[^leak-check]: To be honest, I don't know much about the current system, called
  the [leak check][].

[^negation]: In fact it *cannot* support logical negation. I'll talk more about
  this later.

[^cardinality]: This result comes from ["Quantifier-Elimination for the
  First-Order Theory of Boolean Algebras with Linear Cardinality
  Constraints"][revesz-card], another paper by Peter Revesz.

[^infinite-programs]: There's a fun proof of the cardinality of valid programs
  in any general purpose programming language. If you accept that a program
  exists that can output any natural number, then there are an infinite number
  of valid programs. Furthermore, if you accept that all such programs can be
  encoded as a binary string, the number of valid programs is countably
  infinite. It does not necessarily follow that infinitely many programs means
  infinitely many possible loans---what is a loan anyways?---but does strongly
  suggest it.

[^static]: This was surprising to me, since I expected `'static` to be
  something like "the set of all possible loans". When you start working
  through the implications of constraints like `'static: 'a` and `'a: 'static`,
  it becomes clear that `'static` must be the empty set, but I still don't have
  an intuitive understanding of why `'static` contains no loans.

[^forums]: specifically [IRLO][] and `#wg-polonius` on Zulip.

[^boc]: He refers to this class of logic as "Boolean order constraints". You
  can read more in Revesz's textbook, [*Introduction to Constraint Databases*][].


[IRLO]: https://internals.rust-lang.org/t/blog-post-an-alias-based-formulation-of-the-borrow-checker/7411
[nomicon-hrtb]: https://doc.rust-lang.org/nomicon/hrtb.html
[niko-polonius]: http://smallcultfollowing.com/babysteps/blog/2018/04/27/an-alias-based-formulation-of-the-borrow-checker
[niko-hh]: https://smallcultfollowing.com/babysteps/blog/2019/01/21/hereditary-harrop-region-constraints/
[niko-placeholder]: https://smallcultfollowing.com/babysteps/blog/2019/01/17/polonius-and-region-errors/
[powerset]: https://en.wikipedia.org/wiki/Power_set
[lattice]: https://en.wikipedia.org/wiki/Lattice_(order)
[Polonius]: https://github.com/rust-lang/polonius
[qe]: https://mathworld.wolfram.com/QuantifierElimination.html
[revesz-elim]: http://cse.unl.edu/~revesz/papers/IJAC98.pdf
[revesz-card]: http://cse.unl.edu/~revesz/papers/ADBIS04.pdf
[Chalk]: https://github.com/rust-lang/chalk
[lattice]: https://en.wikipedia.org/wiki/Lattice_(order)
[poset]: https://en.wikipedia.org/wiki/Partially_ordered_set
[re-empty]: https://doc.rust-lang.org/stable/nightly-rustc/rustc_middle/ty/sty/enum.RegionKind.html#variant.ReEmpty
[*Introduction to Constraint Databases*]: https://www.springer.com/gp/book/9780387987293
[Presburger arithmetic]: https://en.wikipedia.org/wiki/Presburger_arithmetic
[leak check]: https://github.com/rust-lang/rust/issues/59490
[Cylindrical Algebraic Decomposition]: https://en.wikipedia.org/wiki/Cylindrical_algebraic_decomposition
[issue-65232]: https://github.com/rust-lang/rust/pull/65232

