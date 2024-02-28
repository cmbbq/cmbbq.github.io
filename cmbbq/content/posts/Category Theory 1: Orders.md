+++
title = "Category Theory 1: Orders"
date = "2024-02-26"
tags = ["en", "math"]
description = "The 1st write-up gives an order-theoretic warm-up for the full-fledged category theory. It covers preorders, meets/joins, monotone maps and Galois connections."
showFullContent = false
+++

## Preorders 
Starting from `sets` and `subsets`, we can define the `relation` between s$A$ and $B$ as a `subset` $R \in A\times B$. 

Every `function` is a relation, satisfying 2 properties: 
1. $\forall a \in A$, there exists $b \in B$, such that $(a,b)\in \mathbb{R}$
2. $\forall a, b_1, b_2$, if $(a,b_1) \in R$ and $(a, b_2) \in \mathbb{R}$, then $b_1 = b_2$.

> Order, equivalence, tolerance are all relations.

A `function` $f: A\rightarrowtail B$ is called `injection` if $\forall a_1, a_2, b$, if $(a_1, b), (a_2, b) \in R$, then $a_1 = a_2$.

A `function` $f: A\twoheadrightarrow B$ is called `surjection` if $\forall b \in B$, there exists an $a \in A$, such that $f(a) = b$.

A `function` $f: A \overset{\cong} \rightarrowtail B$ is called `bijection` if it is both surjective and injective.

An `identity function` on a set $X$, denoted $id_X$. It is the bijective function $id_X(x) = x$.

A `partition` on $A$ is just a surjection on some other set $P$: $A \twoheadrightarrow P$.

We can order `partitions`: $A \twoheadrightarrow P_1$, $A \twoheadrightarrow P_2$. We say $P_1 \leqslant P_2$, if there is a `function` $P_1 \rightarrow P_2$ making the diagram commute: $A \twoheadrightarrow P_1 \rightarrow P_2$.

So $A \twoheadrightarrow A$ is the minimum `partition`, and the $A \twoheadrightarrow \underline{1}$ is the maximum `partition`.

An `preorder` is 
1. a set $S$, and
2. a relation $≤ \: \subseteq S \times S$[^1] 

satisfying 2 properties:
1. $S ≤ S$. [^2]
2. $\forall S_1$, $S_2$, $S_3$, if $S_1 ≤ S_2$ and $S_2 ≤ S_3$, then $S_1 ≤ S_3$. [^3]

> A `preorder` is a `category` with at most one `morphism` between any two objects. Something slightly more complicated is that a `preorder` is a `Bool-enriched category`. 

## Meets and Joins 
Order creates `meets` and `joins`. 

Joining $A$ and $B$, denoted by $A\vee B$, results in the least `partition` bigger than both $A$ and $B$. That is $A ≤ (A\vee B)$ and $A ≤ (A\vee B)$. And for any C, if $A ≤ C$ and $B ≤ C$ then $(A\vee B) ≤ C$. 

Put it formally. Let $(P, ≤)$ be a `preorder`, and let $A \subseteq P$ be a `subset`. We say that an element $p \in P$ is a `meet` of $A$ if
1. $\forall a\in A$, we have $p≤ a$[^4], and 
2. for all $q$ such that $q≤ a$ for all $a\in A$, we have that $q≤ p$[^5].

We write $P = \wedge A =  \underset{a\in A}{\wedge} a = a_1 \wedge a_2 \wedge ... \wedge a_n$ 

Similarly, we say that $p$ is a `join` of $A$ if
1. $\forall a\in A$, we have $a≤ p$[^6], and
2. for all $q$ such that $a≤ q$ for all $a\in A$, we have that $p≤ q$[^7].

In this case, $P = \vee A =  \underset{a\in A}{\vee} a = a_1 \vee a_2 \vee ... \vee a_n$ 

We next discuss examples.

**Example 1: Truth Tables of the Booleans $\mathbb{B}$=\{T,F\}$.**

For example, the pairwise `meets` table of $\{T,F\}(F \leq T)$ happens to be the truth table for `AND` in elementary logic. A similar binary join computation will generate the truth table for `OR`.

**Example 2: Power Sets, $(P(x), ≤)$.**

Let's pick a particular set, let $X = \{\square, \times, \heartsuit\}$, then we consider its power sets:

![power_sets](https://cmbbq.github.io/img/power_sets.png)

In this case, it's clear that $\wedge$ = intersection, $\vee$ = union.

**Example 3: $(\mathbb{N}, |), a ≤ b$, iff $a|b$**

![divisible](https://cmbbq.github.io/img/divisible.png)

1 divides everything, so we start from 1. Here we have $\wedge$ = gcd, $\vee$ = lcm.

**Example 4: There may be more than one `meet`/`join`.**
![hasse_diagram](https://cmbbq.github.io/img/hasse_diagram.png)

This hasse diagram specifies a `preorder` where both $c$ and $d$ are `meets` of $A$. We have $c≤ d$ and $d≤ c$, so $c \cong d$, that is $c$ and $d$ are `isomorphic`, which will be covered later; we generally do not run into trouble if we pretend they are equal though.

**Example 5: `Meets` or `joins` may not exist.**

Clearly things don't always have a lower bound or a upper bound at all. There might be multiple lower bounds/upper bounds but these lower/upper bounds are non-comparable.

Throughout these examples, many familiar things can be characterized by this quite simple universal property, be it gcd/lcm, max/min, limits/colimits, or itersection/union.

The idea that things can be characterized by universal properties, implies we are closer to category theory now.

## Monotone Maps
`Preorders` themselves can be related to one another. A `monotone map` is a structure-preserving map for `preorders`.

A `monotone map` between `preorders` $(A, ≤_A)$ and $(B, ≤_B)$ is a `function` $f : A \rightarrow B$ such that, for all elements $x, y ∈ A$, if $x ≤_A y$ then $f (x) ≤_B f(y)$.
![monotone_map](https://cmbbq.github.io/img/monotone_map.png)

Let $\mathbb{B}$ be the preorders of booleans and $\mathbb{N}$ be the preorder of natural numbers. The map $\mathbb{B} \rightarrow \mathbb{N}$ sending false to 17 and true to 24 is a `monotone map`, because it preserves order.

![b2n](https://cmbbq.github.io/img/b2n.png)

A `monotone map` $f: P \rightarrow Q$ preserves `joins` if $\forall p,p' \in P, f(p\vee p') = f(p) \vee f(p')$

For any `preorder` $(P, ≤_P)$, the identity function is monotone.
If $(Q, ≤_Q)$ and $(R, ≤_R)$ are preorders and $f : P → Q$ and $g : Q → R$ are monotone, then $(f ; g): P → R$ is also monotone.

Let $(P, ≤_P)$ and $(Q, ≤_Q)$ be preorders. A `monotone function` $f : P → Q$ is called an `isomorphism` if there exists a `monotone function` $g : Q → P$ such that $f;g = id_P$ and $g;f = id_Q$. This means that for any $p ∈ P$ and $q ∈ Q$, we have $p=g(f(p))$ and $q 
=f(g(q))$.

We refer to $g$ as the inverse of $f$ , and vice versa: $f$ is the inverse of $g$.

If there is an `isomorphism` $P → Q$, we say that $P$ and $Q$ are isomorphic.

> An isomorphism between preorders is basically just a relabeling of the elements.

## Galois Connections
`Galois connections` between `preorders` $P$ and $Q$ is a pair of `monotone maps` $f : P → Q$ and $g : Q → P$ such that $f(p) ≤ q iff p ≤ g(q).$ 

We say that $f$ is the left `adjoint` and g is the right `adjoint` of the `Galois connection`.

> The theory of `Galois connections` is a special case of a more general theory, the theory of `adjunctions`.

**Example 1: $P = Q = \underline{3}$**
![togc](https://cmbbq.github.io/img/Galois_connections.png)

In this case $P$ and $Q$ are total orders, $f$ is left adjoint to $g$ as long as arrows do not cross.

**Example 2: $\mathbb{Z} \xrightarrow[f]{3\times\square} \mathbb{R}$, $\mathbb{R} \xrightarrow[g]{\lfloor\square/3\rfloor} \mathbb{Z}$**

It means we have $5 \xrightarrow[f]{3\times\square}15$, $13.3 \xrightarrow[g]{\lfloor\square/3\rfloor}4$. 

Since $3n ≤ x$ iff $n ≤ \lfloor x/3\rfloor$, $f$ is left adjoint to $g$.

In the end of the day, we can claim that a `monotone map` $f$ is a left/right `adjoint` iff it preserves `joins`/`meets`.




[^1]: Here we use the symbol $≤$ instead of $R$ as it implies a preorder, and the infix notation $S_1 ≤ S_2$ looks more natural than $(S_1,S_2) \in R$.
[^2]: reflexivity
[^3]: transitivity
[^4]: $p$ is the lower bound of $A$
[^5]: $p$ is the greatest lower bound
[^6]: $p$ is the upper bound of $A$
[^7]: $p$ is the least lower bound
