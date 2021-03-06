---
layout: page
title: IVy as a theorem prover
---

In the development of systems, we sometimes have to reason about
mathematical functions and relations in ways that automated theorem
provers can't handle reliably. For these cases, IVy provides a
facility that allows the user to supply the necessary proofs. IVy's
approach to theorem proving is designed to make maximal use of
automated provers. We do this by localizing the proof into
"isolates". Our verification conditions in each isolate are confined
to a logical fragment or theory that an automated prover can handle
reliably.


Primitive judgments
-------------------

A logical development in IVy is a succession of statements or
*judgments*. Each judgment must be justified by a primitive axiom or
inference rule.

Type declarations
=================

On such primitive is a type declaration, like this:

```
    type t

```
This judgment can be read as "let `t` be a type". It is admissible
provided `t` is new symbol that has not been used up to this point
in the development.

Functions and individuals
=========================

We can introduce a constant like this:

```
    individual n : t

```
where `n` is new. This judgment can be read as "let `n` be a term of type
`t`" and is admissible if symbol `n` has not been used up to this point
in the development.  

Similarly, we can introduce new function and relation symbols:

```
    function f(X:t) : t
    relation r(X:t,Y:t)


```
The first judgment can be read as "for any term *X* of type *t*, let
*f*(*X*) be a term of type *t*".  The second says "for any terms
*X,Y* of type *t*, let *r*(*X*,*Y*) be a proposition" (a term of
type `bool`).

Axioms
======

An *axiom* is a proposition whose truth is admitted by fiat. For
example:

```
    axiom [symmety_r] r(X,Y) -> r(Y,X)


```
The free variables *X*,*Y* in the formula are taken as universally
quantified. The text `symmetry_r` between brackets is a name for the
axiom and is optional. The axiom is simply admitted in the current
context without proof. Axioms are dangerous, since they can
introduce inconsistencies. You should use axioms only if you are
developing a foundational theory and you know what you are doing, or
to make a temporary assumption that will later be removed.

Properties
==========

A *property* is a proposition that can be admitted as true only if
it follows logically from judgments in the current context. For
example:

```
    property [myprop] r(n,X) -> r(X,n)


```
A property requires a proof (see below). If a proof is not supplied,
IVy applies its default proof tactic.  This calls the
automated prover Z3 to attempt to prove the property from the
previously admitted judgments in the current context.

The default tactic works by generating a *verification condition* to
be checked by Z3. This is a formula whose validity implies that the
property is true in the current context. In this case, the
verification condition is:

```
    #-   (forall X,Y. r(X,Y) -> r(Y,X)) -> (r(n,X) -> r(X,n))

```
That is, it states that axiom `symmetry_r` implies property `myprop`. 
IVy checks that this formula is within a logical fragment that Z3 can
decide, then passes the formula to Z3. If Z3 finds that the formula is
valid, the property is admitted.


Definitions
===========

A definition is a special form of axiom that cannot introduce an
inconsistency. As an example:

```
    function g(X:t) : t

    definition g(X) = f(X) + 1

```
This can be read as "for all *X*, let *g*(*X*) equal *f*(*X*) + 1". The
definition is admissible if the symbol *g* is "fresh", meaning it
does not occur in any existing properties or definitions in the
current context.  Further *g* must not occur on the right-hand side
of the equality (that is, a recursive definition is not admissible
without proof -- see "Recursion" below).

Theory instantiations
=====================

IVy has built-in theories that can be instantiated with a specific type
as their carrier. For example, the theory of integer arithmetic is
called `int`. It has the signature `{+,-,*,/,<}`, plus the integer
numerals, and provides the axioms of Peano arithmetic. To instantiate
the theory `int` using type *u* for the integers, we write:

```
    type u
    interpret u -> int

```
This declaration requires that type `u` is not previously interpreted
and that the symbols `{+,-,*,/,<}` in the signature of `int` are
fresh. Notice, though, that the symbols `{+,-,*,/,<}` are
overloaded. This means that `+` over distinct types is considered to
be a distinct symbol.  Thus, we can we can instantiate the `int`
theory over different types (and also instantiate other theories that
have these symbols in their signature).

Schemata
--------

A *schema* is a compound judgment that takes a collection of judgments
as input (the premises) and produces a judgment as output (the
conclusion). If the schema is valid, then we can provide any
type-correct values for the premises and the conclusion will follow.

Here is a simple example of a schema taken as an axiom:

```
    axiom [congruence] {
        type d
        type r
        function f(X:d) : r
        #--------------------------
        property X = Y -> f(X) = f(Y)
    }

```
The schema is contained in curly brackets and gives a list of premises
following a conclusion. In this case, it says that, given types *d* and *r* and a function *f* from
*d* to *r* and any values *X*,*Y* of type *d*,  we can infer that *X*=*Y* implies
*f*(*X*) = *f*(*Y*). The dashes separating the premises from the conclusion are
just a comment. The conclusion is always the last judgment in the schema.
Also, notice the declaration of function *f* contains a variable *X*. The scope of this
variable is only the function declaration. It has no relation to the variable *X* in the conclusion.

The keyword `axiom` tells IVy that this schema should be taken as valid
without proof. However, as we will see, the default
tactic treats a schema differently from a simple proposition. That is,
a schema is never used by default, but instead must be explicitly
instantiated.  This allows is to express and use facts that, if they
occurred in a verification condition, would take us outside the
decidable fragment.

Any judgment that has been admitted in the current context can be
*applied* in a proof. When we apply a schema, we supply values for its
premises to infer its conclusion.


```
    property [prop_n] Z = n -> Z + 1 = n + 1
    proof {
        apply congruence
    }

```
The `proof` declaration tells IVy to apply the axiom schema `congruence` to prove the property. 
IVy tries to match the proof goal `prop_n` to the schema's conclusion by picking particular
values for premises, that is, the types *d*,*r* and function *f*. It also chooses terms for the
the free variables *X*,*Y* occurring in the schema. In this case, it
discovers the following assignment:

```
    #- d = t
    #- r = t
    #- X = Z
    #- Y = n
    #- f(N) = N + 1

```
After plugging in this assignment, the conclusion of the rule exactly matches
the property to be proved, so the property is admitted.

The problem of finding an assignment such as the one above is one of
"second order matching".  It is a hard problem, and the answer is not
unique. In case IVy did not succeed in finding the above match, we
could also have written the proof more explicitly, like this:

```
    property [prop_n_2] Z = n -> Z + 1 = n + 1
    proof {
        apply congruence with X=Z, Y=n, f(X) = X:t + 1
    }

```
Each of the above equations acts as a constraint on the
assignment. That is, it must convert *X* to
*Z*, *Y* to *n* and *f*(*X*) to *X* + 1. Notice that we had to
explicitly type *X* on the right-hand side of the last equation,
since its type couldn't be inferred (and in fact it's not the same
as the type of *X* on the left-hand side, which is *d*). 

It's also possible to write constraints that do not allow for any
assignment. In this case, Ivy complains that the provided match is
inconsistent.

Forward proofs
==============

In the normal way of writing proofs, we start from a set of premises,
then proceed to prove a sequence of facts (or lemmata), ending in a
conclusion. Each fact must be seen to follow from the preceding facts
by applying a valid schema. 

For example, consider the following axiom schema:

```
    relation man(X:t)
    relation mortal(X:t)

    axiom [mortality_of_man] {
        property [prem] man(X)
        #---------------
        property mortal(X)
    }


```
The scope of free variables such as *X* occurring in properties is
the entire schema. Thus, this schema says that, for any term *X* of
type `t`, if we can prove that `man(X)` is true, we can prove that
`mortal(X)` is true.

We take as a given that Socrates is a man, and prove he is mortal:

```
    individual socrates : t

    axiom [soc_man] man(socrates)

    property mortal(socrates)
    proof {
        apply mortality_of_man with prem = soc_man
    }


```
The axiom `mortality_of_man`, requires us supply the premise
`man(socrates)`.  Fortunately, we have this fact as an axiom.
To justify the conclusion `mortal(socrates)` we tell Ivy to apply
axiom `mortality_of_man` with `soc_man` as its premise. 

Ivy achieves this proof by *matching* the axiom `mortality_of_man`
against the provided premise and conclusion. In this process, it
discovers the substitution `X = socrates`. Applying this substitution to
the axiom yields the deduction we wish to make.  However, if we wanted
to be more explicit about the substitution, we could have written:

```
    property mortal(socrates)
    proof {
        apply mortality_of_man with prem = soc_man, X = socrates
    }


```
Theorems
========

Thus far, we have seen schemata used only as axioms. However, we can also
prove the validity of a schema as a *theorem*. For example, here is a theorem
expressing the transitivity of equality:

```
    theorem [trans] {
        type t
        property [left] X:t = Y
        property [right] Y:t = Z
        #--------------------------------
        property X:t = Z
    }


```
We don't need to provide a proof for this Ivy's default tactic using Z3 can handle it. 
The verification condition that Ivy generates is:

```
    #-   X = Y & Y = Z -> X = Z

```
Here is a theorem that lets us eliminate universal quantifiers:

```
    theorem [elimA] {
        type t
        function p(W:t) : bool
        property [prem] forall Y. p(Y)
        #--------------------------------
        property p(X)
    }

```
It says, for any predicate *p*, if `p(Y)` holds for all *Y*, then
`p(X)` holds for a particular *X*. Again, Z3 can
prove this easily. Now let's derive a consequence of these facts. A
function *f* is *idempotent* if applying it twice gives the same
result as applying it once. This theorem says that, if *f* is
idempotent, then applying *f* three times is the same as applying it
once:

```
    theorem [idem2] {
        type t
        function f(X:t) : t
        property [idem] forall X. f(f(X)) = f(X)
        #--------------------------------
        property f(f(f(X))) = f(X)
    }

```
The default tactic can't prove this because the premise contains a
function cycle with a universally quantified variable. Here's the
error message it gives:

```
    #- error: The verification condition is not in the fragment FAU.
    #-
    #- The following terms may generate an infinite sequence of instantiations:
    #-   proving.ivy: line 331: f(f(X_h))

```
This means we'll need to apply some other tactic. Here is one possible proof:

```
    proof {
        property [lem1] f(f(f(X))) = f(f(X)) proof {apply elimA with prem=idem}
        property [lem2] f(f(X)) = f(X) proof {apply elimA with prem=idem}
        apply trans with left = lem1
    }

```
We think this theorem holds because `f(f(f(X)))` is equal to `f(f(X))`
(by idempotence of *f*) which in turn is equal to `f(X)` (again, by
idempotence of *f*). We then apply transitivity to infer `f(f(f(X))) = f(X)`.

To prove that `f(f(f(X))) = f(f(X))` from idempotence, we apply the
`elimA` theorem with `idem` as the premise. Let's look in a little more detail at how this works.
The `elimA` theorem has a premise `forall X. p(X)`. Ivy matches has to find a substitution that
will match this formulas `idem`, the premise that we want to use, which is `forall X. f(f(X)) = f(X)`.
It discovers the following substitution:

```
    #- t = t
    #- p(Y) = f(f(Y)) = f(Y)

```
Now Ivy tries to match the conclusion `p(X)` against the property `lem1` we are proving, which
is `f(f(f(X))) = f(f(X))`. It first plugs in the above substitution, so `p(X)` becomes:

```
    #- f(f(X)) = f(X)

```
Matching this with our goal `lem1`, it then gets this substitution:

```
    #- X = f(X)

```
Plugging these substitutions into `elimA` as defined above, we obtain:

```
    #- theorem [elimA] {
    #-     property [prem] forall Y. f(f(Y)) = f(Y)
    #-     #--------------------------------
    #-     property f(f(f(X))) = f(f(X))
    #- }

```
which is just the inference we want to make. This might give the impression that Ivy can
magically arrive at the instantiation of a lemma needed to produce a desired inference.
In general, however, this is a hard problem. If Ivy fails, we can still write out the
full substitution manually, like this:

```
    #- property [lem1] f(f(f(X))) = f(f(X))
    #- proof {
    #-     apply elimA with t=t, p(Y) = f(f(Y)) = f(Y), X=f(X)
    #- }

```
The second step of our proof, deriving the fact `f(f(X) = f(X)` is a
similar application of `idem`.  The final step uses transitivity,
with our two lemmas as premises. In this case, we only have to
specify the the `left` premise of `trans` is `lem1`. This is enough
to infer the needed substitution.  There would be no harm, however,
in writing `right=lem2`.


Natural deduction
=================

The above proof is an example of reasoning in a style called 
[natural deduction](https://en.wikipedia.org/wiki/Natural_deduction).
In fact, `elimA` is a primitive inference rule in natural deduction. 
In the natural deduction style, a premise of a theorem is often itself a theorem.
For example, here is a rule of natural deduction that is used to prove an implication:

```
    theorem [introImp] {
        function p : bool
        function q : bool
        theorem [lem] {
            property [prem] p
            #----------------------
            property q
        }
        #----------------------
        property p -> q
    }

```
This rule says that, for any predicates *p* and *q*, if we can prove *q* from *p*, then
we can prove `p -> q`. The premise `lem` of this rule is itself a lemma, which we have to
supply in order to use the rule.

As an application of `introImp`, let's try to prove the following theorem:

```
    relation p
    relation q

    property p & (p->q) -> q

```
To apply `introImp` to prove the implication `p & (p->q) -> q`, we first need a proof of `q`
from `p & (p->q)`. So here's the start of a proof:

```
    proof {
        theorem [lem1] {
            property [p1] p & (p -> q)
            property q
        } 
        apply introImp
    }

```
Notice we didn't actually prove `lem1`. By leaving out the proof, we
effectively pass the problem of proving `lem1` on to Z3. With this
lemma, we can then apply `introImp` to give us our goal. We could also
have said `with lem=lem1`, but we didn't have to because Ivy can
figure out the premise needed by `introImp` just by matching the
conclusions. 

Now, if we don't want to burden Z3 with the proof of `lem1`, we can
fill it in. We'll use this rule for eliminating implications:

```
    theorem [elimImp] {
        function p : bool
        function q : bool
        property [prem1] p
        property [prem2] p -> q
        #----------------------
        property q
    }

```
This rule says that, for any predicates *p* and *q*, if we can
prove *p* and we can prove `p -> q`, then we can prove `q`.  You
might also recognize the `elimImp` rule as *modus ponens*.

Here's a slightly more detailed proof of our theorem that includes a
proof of `lem1`::

```
    property p & (p->q) -> q
    proof {
        theorem [lem1] {
            property [p1] p & (p -> q)
            property q
        } proof {
            property [p2] p
            property [p3] p -> q 
            apply elimImp with prem1 = p1
        }
        apply introImp
    }

```
That is, from `p & (p -> q)` we can infer `p` and `p -> q`.
Notice that when we applied `elimImp`, we again specified
only one premise, since this was enough to determine the
substitution for `p` in the rule.

If we want to add more detail to the proof, we can can go another
level deeper by adding proofs for `p2` and `p3`. We'll use these two
additional rules for eliminating "and" operators:

```
    axiom [elimAndL] {
        function p : bool
        function q : bool
        property [prem] p & q
        #----------------------
        property p
    }

    axiom [elimAndR] {
        function p : bool
        function q : bool
        property [prem] p & q
        #----------------------
        property q
    }

```
Here is the more detailed proof using `elimAndL` and `elimAndR`:

```
    property p & (p->q) -> q
    proof {
        theorem [lem1] {
            property [p1] p & (p -> q)
            property q
        } proof {
            property [p2] p proof {apply elimAndL with prem = p1}
            property [p3] p -> q proof {apply elimAndR with prem = p1}
            apply elimImp with prem1 = p2
        }
        apply introImp
    }

```
Notice the hierarchical style of proofs in natural deduction. Each lemma is
proved by a sequence of sub-lemmas, until we reach the bottom level at which
the lemmas are proved by primitive rules.

The full set of inference rules of natural deduction (including
equality) can be found in the the Ivy standard header `deduction`.
Many tutorial resources may be found on the World-Wide Web on writing
natural deduction proofs.

Backward proof
==============

An alternative way to construct a proof in Ivy is *backward chaining*.
While it is perhaps most natural to use proof rules to work forward
from premises to conclusions, it is equally possible to use the same
rules to work backward from conclusions to premises.  When applying a
rule, Ivy does not require that all premises of the rule be
immediately supplied.  An unsupplied premise becomes a *subgoal* which
we must prove later. In effect, the proof of the missing premise is
"pushed on the stack". 

As an example, let's look at an alternative proof of the mortality
of Socrates:

```
    property mortal(socrates)
    proof {
        apply mortality_of_man
        apply soc_man
    }


```
When we apply `mortality_of_man` to prove `mortal(socrates)` we lack the
necessary premise `man(socrates)`. Ivy is undeterred by this. It simply
matches the conclusion `mortal(X)` to `mortal(socrates)`, instantiates
the axiom `mortality_of_man` and pushes the needed premise `man(socrates)`
on the goal stack. We then discharge this goal using `soc_man`, leaving the
goal stack empty. Each proof step is applied to the top goal on the stack. 
As you may have guessed, goals remaining on the stack at the end of the 'proof'
are passed to Z3. 

When chaining proof rules backward, it is helpful to be able to see
the intermediate subgoals, since we do not write them
explicitly. This can be done with the `showgoals` tactic, like this:

```
    property mortal(socrates)
    proof {
        apply mortality_of_man
        showgoals
        apply soc_man
    }

```
When checking the proof, the `showgoals` tactic has the effect of
printing the current list of proof goals, leaving the goals
unchanged.  A good way to develop a backward proof is to start with
just the tactic `showgoals`, and to add tactics before it. Running
the Ivy proof checker in an Emacs compilation buffer is a convenient
way to do this.

More examples of backward proofs
--------------------------------

Here, once again, is our theorem about idempotence:

```
    theorem [idem3] {
       type t
       function f(X:t) : t
       property forall X. f(f(X)) = f(X)
       #--------------------------------
       property f(f(f(X))) = f(X)
    }

```
Here is one possible proof in the backward style:

```
    proof  {
        apply trans with Y = f(f(X))
        apply elimA with X = f(X)
        apply elimA with X = X
    }

```
In fact, it is the same proof that we wrote in the forward style, but
it is now both more succinct and more difficult to understand. 

As another example, here is our proof of theorem `mp` written in the
backward style:


```
    property p & (p->q) -> q
    proof {
        apply introImp
        apply elimImp with p = p
        apply elimAndL with q = (p->q)
        apply elimAndR with p = p
    }

```
Again, it is much more succinct, but offers no intuition as to *why* the
theorem is true. 

When is a backward proof appropriate? One use case is when we wish to
make a small transformation to a very complex formula before passing
the problem to Z3. In this situation, writing out the resulting formula
as an explicit lemma is probably a waste of effort, since the formula is
intended to be consumed by an automated tool rather than a human user. 

Skolemization and instantiation
===============================

Quantifiers are the usual reason that a proof goal lies outside
Ivy's decidable fragment. Usually, the fastest way to prove
something that is not in the decidable fragment is to eliminate the
quantifier that is causing the problem. This is possible with
natural deduction, but sometimes very inconvenient. A better alternative is to
use Skolemization and instantiation.

Here is the idempotence lemma done in this style:

```
    theorem [idem4] {
        type t
        function f(W:t) : t
        property [idem] forall Y. f(f(Y)) = f(Y)
        #--------------------------------
        property f(f(f(X))) = f(X)
    }
    proof {
        instantiate idem with Y = X
    }

```
Remember that when we tried to prove this before, Ivy complained the
the term `f((Y))` put us outside the decidable fragment. This is
because `Y` was universally quantified in the premise `idem`. In
this proof, however, we use the `instantiate` tactic to plug in the
value `X` for the universally quantified `Y` in `idem`. This is
called *quantifier instantiation* and gives us the following proof
goal:

```
    #- theorem [idem4] {
    #-     type t
    #-     function f(W:t) : t
    #-     property [idem] f(f(X)) = f(X)
    #-     #--------------------------------
    #-     property f(f(f(X))) = f(X)
    #- }

```
Without the quantifier, Z3 has no problem handling this proof goal.

It turns out we can always handle purely first-order proof goals in
this way, provided we apply a further tactic called
`skolemize`. This tactic allows us to deal with alternations of quantifiers.

Here is an example:


```
    theorem [sk1] {
        type t
        individual a : t
        relation r(X:t,Y:t)
        property [prem] forall X. exists Y. r(X,Y)
        #---
        property exists Z. r(a,Z)
    }
    proof {
        tactic skolemize
        instantiate prem with X=a
        instantiate with Z = _Y(a)
    }

```
The `skolemize` tactic replaces the existentially quantified `Y` in `prem`
by a fresh function `_Y(X)`. Here is the resulting proof goal:

```
    #- theorem [sk1] {
    #-     function _Y(V0:t) : t
    #-     type t
    #-     individual a : t
    #-     relation r(V0:t,V1:t)
    #-     property [prem] forall X. r(X,_Y(X))
    #-     property exists Z. r(a,Z)
    #- }

```
When `skolemize` is done, the premises have only universal
quantifiers, and the conclusion has only existential quantifiers.
Now, we can finish the proof by instantiating the universal in
`prem` with `a`, giving `r(a,_Y(a))`, and then instantiating the
existentially quantified `Z` in the conclusion with `_Y(a)` , which
also gives `r(a,_Y(a))`. One way to describe the instantiation of
the conclusion is that we have provided the term `_Y(a)` as a
*witness* for the existential quantifier. Our final proof goal is
trivial:

```
    #- theorem [sk1] {
    #-     function _Y(V0:t) : t
    #-     type t
    #-     individual a : t
    #-     relation r(V0:t,V1:t)
    #-     property [prem] r(a,_Y(a))
    #-     property r(a,_Y(a))
    #- }

```
All of this wasn't really necessary, since the original theorem is
in the decidable fragment.  However, the approach of Skolemization
followed by selective instantiation of quantifiers can be a quick
way to reduce a proof problem to a decidable one. First, we try to
get Z3 to discharge the proof goal automatically. If Ivy complains
about a quantified variable, we eliminate that variable by
Skolemization and instantiation. If we guess the wrong instantiation,
Ivy will give us a counterexample.

Avoiding name clashes
---------------------

How does Skolemization work if there is more than one variable with
the same name? As an example, suppose we have this axiom:

```
    relation s(X:t,Y:t)
    axiom [prem] forall X. exists Y. s(X,Y)

```
We would like to prove this property:

```
    individual a : t
    individual b : t
    property [sk2] exists Z,W. s(a,Z) & s(b,W)

```
To prove this, we would like to use the axiom twice, once for `X=a`
and once for `X=b`. So let's create two instances of it:

```
    proof {
        instantiate [p1] prem with X=a
        instantiate [p2] prem with X=b
        tactic skolemize
        showgoals
    }

```
Notice we gave the two instances distinct names, `p1` and
`p2`. After Skolemization, this is the proof goal we got:

```
    #- theorem [prop1] {
    #-     individual _Y : t
    #-     individual _Y_a : t
    #-     property [p1] s(a,_Y)
    #-     property [p2] s(b,_Y_a)
    #-     property exists W,Z. s(a,Z) & s(b,W)
    #- }

```
The Skolem symbol in `p2` was automatically given a fresh name
`_Y_a` so as not to clash with the symbol `_Y` used in `p1`. Now we
could witness the conclusion with `W=_Y` and `Z=_Y_a`. This is
fragile, though, because changes to the proof might result in a name
different from `_Y_a`. A better approach is to use *alpha renaming* to
the two quantifiers different variable names. For example:

```
    property exists Z,W. s(a,Z) & s(b,W)
    proof {
        instantiate [p1] prem<Y1/Y> with X=a
        instantiate [p2] prem<Y2/Y> with X=b
        tactic skolemize
        instantiate with Z=_Y1, W=_Y2
    }

```
By renaming the bound variables in our two copies of the axiom to
be `Y1` and `Y2`, we can make sure that the Skolem symbols have
predictable names. Again, there was no need to use manual
instantiation for this simple problem, but in more complex cases
with nested quantifiers, this approach can make it possible to let Z3
do most of the quantifier instantiation work by only instantiating
the quantifiers that the fragment checker complains about.



Recursion
=========

Recursive definitions are permitted in IVy by instantiating a
*definitional schema*. As an example, consider the following axiom schema:

```
    #- axiom [rec[u]] {
    #-     type q
    #-     function base(X:u) : q
    #-     function step(X:q,Y:u) : q
    #-     fresh function fun(X:u) : q
    #-     #---------------------------------------------------------
    #-     definition fun(X:u) = base(X) if X <= 0 else step(fun(X-1),X)
    #- }

```
This axiom was provided as part of the integer theory when we
interpreted type *u* as `int`.  It gives a way to construct a fresh
function `fun` from two functions:

- `base` gives the value of the function for inputs less than or equal to zero.
- `step` gives the value for positive *X* in terms of *X* and the value for *X*-1

A definition schema such as this requires that the defined function
symbol be fresh. With this schema, we can define a recursive function that
adds the non-negative numbers less than or equal to *X* like this:

```
    function sum(X:u) : u

    definition {
       sum(X:u) = 0 if X <= 0 else (X + sum(X-1))
    }
    proof {
       apply rec[u]
    }

```
Notice that we wrote the definition in curly brackets. This causes Ivy to 
treat it as an axiom schema, as opposed to a simple axiom.
We did this because the definition has a universally quantified variable
`X` under arithmetic operators, which puts it outside the decidable
fragment. Because this definition is a schema, when we want to use it,
we'll have to apply it explicitly,

In order to admit this definition, we applied the definition
schema `rec[u]`. Ivy infers the following substitution:

```
    #-  q=t, base(X) = 0, step(X,Y) = Y + X, fun(X) = sum(X)

```
This allows the recursive definition to be admitted, providing that
`sum` is fresh in the current context (i.e., we have not previously referred to
`sum` except to declare its type).


### Extended matching

Suppose we want to define a recursive function that takes an additional
parameter. For example, before summing, we want to divide all the
numbers by *N*. We can define such a function like this:

```
    function sumdiv(N:u,X:u) : u

    definition
        sumdiv(N,X) = 0 if X <= 0 else (X/N + sumdiv(N,X-1))
    proof
       apply rec[u]

```
In matching the recursion schema `rec[u]`, IVy will extend the
function symbols in the premises of `rec[u]` with an extra parameter *N* so that
schema becomes:

```
    #- axiom [rec[u]] = {
    #-     type q
    #-     function base(N:u,X:u) : q
    #-     function step(N:u,X:q,Y:u) : q
    #-     fresh function fun(N:u,X:u) : q
    #-     #---------------------------------------------------------
    #-     definition fun(N:u,X:u) = base(N,X) if X <= 0 else step(N,fun(N,X-1),X)
    #- }

```
The extended schema matches the definition, with the following assignment:

```
    #- q=t, base(N,X)=0, step(N,X,Y)=Y/N+X, fun(N,X) = sum2(N,X)

```
This is somewhat as if the functions were "curried", in which case the
free symbol `fun` would match the partially applied term `sumdiv N`.
Since Ivy's logic doesn't allow for partial application of functions,
extended matching provides an alternative. Notice that, 
to match the recursion schema, a function definition must be
recursive in its *last* parameter.

Induction
=========

The default tactic can't generally prove properties by induction. For
that IVy needs manual help. To prove a property by induction, we define
an invariant and prove it by instantiating an induction schema. Here is
an example of such a schema, that works for the non-negative integers:

```
    axiom [ind[u]] {
        relation p(X:u)
        theorem [base] {
            individual x:u
            property x <= 0 -> p(x)
        }
        theorem [step] {
            individual x:u
            property p(x) -> p(x+1)
        }
        #--------------------------
        property p(X)    
    }

```
Like the recursion schema `rec[u]`, the induction schema `ind[u]` is
part of the integer theory, and becomes available when we interpret
type `u` as `int`.

Suppose we want to prove that `sum(Y)` is always greater than or equal
to *Y*, that is:

```
    property [sumprop] sum(Y) >= Y

```
We can prove this by applying our induction schema. Here is a backward version of the proof:

```
    proof [sumprop] {
        apply ind[u] with X = Y
        proof [base] {
            instantiate sum with X = x
        }
        proof [step] {
            instantiate sum with X = x + 1
        }
    }

```
Applying `ind[u]` produces two sub-goals, a base case and an induction step:

```
    #- theorem [base] {
    #-    individual x:u
    #-    property x <= 0 -> sum(x) >= x
    #- }

    #- theorem [step] {
    #-     individual x:u
    #-     property [step] sum(x) >= x -> sum(x+1) >= x + 1
    #- }

```
The default tactic can't prove these goals because the definition of
`sum` is a schema that must explicitly instantiated. Fortunately, it
suffices to instantiate this schema just for the specific arguments
of `sum` in our subgoals. For the base case, we need to instantiate
the definition for `X`, while for the induction step, we need
`X+1`. The effect of the `instantiate` tactic is to add an instance of the definition of
`sum` to the proof context, so in particular, it will be used by the default tactic.
Notice that we referred to the definition of `sum` by the name
`sum`.  Alternatively, we can name the definition itself and refer
to it by this name.

After instantiating the definition of `sum`, our two subgoals look like this:

```
    #- theorem [prop5] {
    #-     property [def2] sum(Y + 1) = (0 if Y + 1 <= 0 else Y + 1 + sum(Y + 1 - 1))
    #-     property sum(Y) >= Y -> sum(Y + 1) >= Y + 1
    #- }


    #- theorem [prop4] {
    #-     property [def2] sum(Y) = (0 if Y <= 0 else Y + sum(Y - 1))
    #-     property Y:u <= 0 -> sum(Y) >= Y
    #- }


```
Because these goals are quantifier-free the default tactic can easily
handle them, so our proof is complete.

As an alternative, here is the slightly more verbose forward version of this proof:

```
    property [sumprop2] sum(Y) >= Y

    proof [sumprop2] {
        theorem [base] {
            individual x:u
            property x <= 0 -> sum(x) >= x
        } proof {
            instantiate sum with X = x
        }
        theorem [step] {
            individual x:u
            property sum(x) >= x -> sum(x+1) >= x + 1
        } proof {
            instantiate sum with X = x + 1
        }
        apply ind[u]
    }

```
This version is more clear than the backward version in that the base
case and inductive step of the proof are stated explicitly. For a complex
inductive hypothesis, this style might get a bit verbose. To reduce
the amount of text we need to type, we can introduce a function definition
inside a proof, like this:

```
    theorem [foo] {property sum(X) >= X}
    proof {
        function inv(X) = sum(X) >= X
        property inv(X) proof {
            theorem [base] {
                individual x:u
                property x <= 0 -> inv(x)
            } proof {
                instantiate sum with X = x
            }
            theorem [step] {
                individual x:u
                property inv(x) -> inv(x + 1)
            } proof {
                instantiate sum with X = x + 1
            }
            apply ind[u]
        }
    }


```

Naming
======

If we can prove that something exists, we can give it a name.  For
example, suppose that we can prove that, for every *X*, there exists a
*Y* such that `succ(X,Y)`. Then there exists a function that, given an
*X*, produces such a *Y*. We can define such a function called `next`
in the following way:

```
    relation succ(X:t,Y:t)
    axiom exists Y. succ(X,Y)

    property exists Y. succ(X,Y) named next(X)

```
Provided we can prove the property, and that `next` is fresh, we can
infer that, for all *X*, `succ(X,next(X))` holds. Defining a function
in this way, (that is, as a Skolem function) can be quite useful in
constructing a proof.  However, since proofs in Ivy are not generally
constructive, we have no way to *compute* the function `next`, so we
can't use it in extracted code.

Hierarchical proof development
------------------------------

As the proof context gets larger, it becomes increasingly difficult
for the automated prover to handle all of the judgments we have
admitted. This is especially true as combining facts or theories may
take us outside the automated prover's decidable fragment. For this
reason, we need some way to break the proof into manageable parts.
For this purpose, IVy provides a mechanism to structure the proof into
a collection of localized proofs called *isolates*.

Isolates
========

An isolate is a restricted proof context. An isolate can make parts of
its proof context available to other isolates and keep other parts
hidden. Moreover, isolates can contain isolates, allowing us to
structure a proof development hierarchically.

Suppose, for example, that we want to define a function *f*, but keep
the exact definition hidden, exposing only certain properties of the
function. We could write something like this:

```
    isolate t_theory = {

        implementation {
            interpret t -> int
            definition f(X) = X + 1
        }

        theorem [expanding] { 
            property f(X) > X
        }
        property [transitivity] X:t < Y & Y < Z -> X < Z        

    }

```
Any names declared within the isolate belong to its namespace. For
example, the names of the two properties above are
`t_theory.expanding` and `t_theory.transitivity`.

The isolate contains four declarations. The first, says the type `t`
is to be interpreted as the integers. This instantiates the theory
of the integers for type `t`, giving the usual meaning to operators
like `+` and `<`. The second defines *f* to be the integer successor
function.  These two declarations are contained an *implementation*
section. This means that the default tactic will use them only within
the isolate and not outside.

The remaining two statements are properties about *t* and *f* to
be proved. These properties are proved using only the context of the
isolate, without any judgments admitted outside.

Now suppose we want to prove an extra property using `t_theory`:

```
    isolate extra = {

        theorem [prop] {  
            property Y < Z -> Y < f(Z)
        }
        proof {
            instantiate t_theory.expanding with X = Z
        }
    }
    with t_theory

```
The 'with' clause says that the properties in `t_theory` should be
used by the default tactic within the isolate. In this case, the `transitivity` 
property will be used by default. This pattern is particularly useful when
we have a collection of properties of an abstract datatype that we wish to
use widely without explicitly instantiating them. 

Notice that the default tactic will not use the interpretation of *t* as the
integers and the definition of *f* as the successor function, since
these are in the implementation section of isolate `t_theory` and are therefore
hidden from other isolates. Similarly, theorem `expanding` is not used by default
because it is a schema. This is as intended, since including any of these facts would
put the verification condition outside the decidable fragment.

We used two typical techniques here to keep the verification
conditions decidable.  First, we hid the integer theory and the
definition of *f* inside an isolate, replacing them with some
abstract properties. Second, we eliminated a potential cycle in the
function graph by instantiating the quantifier implicit in theorem
`expanding`, resulting in a stratified proof goal.



Hierarchies of isolates
=======================

An isolate can in turn contain isolates. This allows us to structure a
proof hierarchically. For example, the above proof could have been
structured like this:

```
    isolate extra2 = {

        function f(X:t) : t

        isolate t_theory = {

            implementation {
                interpret t -> int
                definition f(X) = X + 1
            }

            theorem [expanding] { 
                property f(X) > X
            }
            property [transitivity] X:t < Y & Y < Z -> X < Z        

        }

        theorem [prop] {  
            property Y < Z -> Y < f(Z)
        }
        proof
            assume t_theory.expanding with X = Z

    }

```
The parent isolate `extra2` uses only the visible parts of the child isolate `t_theory`. 

Proof ordering and refinement
=============================

Thus far, proof developments have been presented in order. That is,
judgments occur in the file in the order in which they are admitted
to the proof context.

In practice, this strict ordering can be inconvenient. For example,
from the point of view of clear presentation, it may often be better
to state a theorem, and *then* develop a sequence of auxiliary
definitions and lemmas needed to prove it.  Moreover, when developing
an isolate, we would like to first state the visible judgments, then
give the hidden implementation.

To achieve this, we can use a *specification* section. The
declarations in this section are admitted logically *after* the other
declarations in the isolate.

As an example, we can rewrite the above proof development so that the
visible properties of the isolates occur textually at the beginning:



```
    isolate extra3 = {

        function f(X:t) : t

        specification {
            theorem [prop] {  
                property Y < Z -> Y < f(Z)
            }
            proof
                assume t_theory.expanding with X = Z
        }

        isolate t_theory = {

            specification {
                theorem [expanding] { 
                    property f(X) > X
                }
                property [transitivity] X:t < Y & Y < Z -> X < Z        
            }

            implementation {
                interpret t -> int
                definition f(X) = X + 1
            }
        }
    }
```
