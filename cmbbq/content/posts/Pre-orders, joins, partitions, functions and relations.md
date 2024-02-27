+++
title = "Cat01: Pre-orders, joins, partitions, functions and relations"
date = "2024-02-26"
tags = ["en", "math"]
description = "The write-up gives an order-theoretic warm-up for the full-fledged category theory."
showFullContent = false
+++

Starting from `sets` and `subsets`, we can define the `relation` between s$A$ and $B$ as a subset $R \in A\times B$. 

Every `function` is a relation, satisfying 2 properties: 
1. $\forall a \in A$, there exists $b \in B$, such that $(a,b)\in \mathbb{R}$
2. $\forall a, b_1, b_2$, if $(a,b_1) \in R$ and $(a, b_2) \in \mathbb{R}$, then $b_1 = b_2$.

> Order, equivalence, tolerance are all relations.

A function $f: A\rightarrowtail B$ is called `injection` if $\forall a_1, a_2, b$, if $(a_1, b), (a_2, b) \in R$, then $a_1 = a_2$.

A function $f: A\twoheadrightarrow B$ is called `surjection` if $\forall b \in B$, there exists an $a \in A$, such that $f(a) = b$.

A `partition` on $A$ is just a surjection on some other set $P$: $A \twoheadrightarrow P$.

We can order partitions: $A \twoheadrightarrow P_1$, $A \twoheadrightarrow P_2$. We say $P_1 \leqslant P_2$, if there is a function $P_1 \rightarrow P_2$ making the diagram commute: $A \twoheadrightarrow P_1 \rightarrow P_2$.

So $A \twoheadrightarrow A$ is the minimum partition, and the $A \twoheadrightarrow \underline{1}$ is the maximum partition.

An `pre-order` is 
1. a set $S$
2. a relation $\preceq \: \subseteq S \times S$[^1] 

satisfying 2 properties:
1. $\forall S' \in S$, $S' \preceq S$.
2. $\forall S_1$, $S_2$, $S_3$, if $S_1 \preceq S_2$ and $S_2 \preceq S_3$, then $S_1 \preceq S_3$.

Order creates `join`. Joining $A$ and $B$, denoted by $A\vee B$, results in the least partition bigger than both $A$ and $B$. That is $A \preceq (A\vee B)$ and $A \preceq (A\vee B)$. And for any C, if $A \preceq C$ and $B \preceq C$ then $(A\vee B) \preceq C$, 

> So order is more fundamental than joining.

[^1]: Here we use the symbol $\preceq$ instead of $R$ as it implies a partial order, and the infix notation $S_1 \preceq S_2$ looks more natural than $(S_1,S_2) \in R$.