+++
title = "CAT02: Resources"
date = "2024-06-14"
tags = ["en", "math"]
description = "The 2nd CAT write-up is about the categorical formalism of resources, and transforming one set of resources into another. It covers monoidal preorders, wiring diagrams, monoidal monotone maps, and V-categories."
showFullContent = false
+++

## Symmetric Monoidal Preorders

**Def 2.1:** A `symmetric monoidal structure` on a preorder $(X,≤)$ consists of two constituents: (i) an element $I \in X$, called the `monoidal unit`, (ii) and a function $\otimes: X \times X \rightarrow X$, called the `monoidal product`. These constituents must satisfy the following properties:
- monotonicity: $\forall x_1, x_2, y_1, y_2 \in X$, if $x_1 \le y_1$ and $x_2 \le y_2$, then $x_1 \otimes x_2 \le y_1 \otimes y_2$.
- unitality: $\forall x \in X$, $I \otimes x = x$ and $x \otimes I = x$ holds.
- associativity: $\forall x,y,z \in X$, $(x\otimes y)\otimes z = x \otimes (y\otimes z)$.
- symmetry: $\forall x,y \in X, x\otimes y = y\otimes x$.[^1]

**Def 2.2:** A preorder equipped with a symmetric monoidal structure, $(X,\le,I,\otimes)$, is called a `symmetric monoidal preorder`.

**Example 2.1(The Booleans):** $\mathbb{B} = \{true, false\}$ with $false < true$ is the simplest nontrivial preorder. We can define the monoidal unit be true and the monoidal product be $\wedge$(AND). Then we have a monoidal preorder which we denote $Bool := (\mathbb{B}, \le ,true, \wedge )$.

## Wiring Diagrams
`Wiring diagrams` are visual representations for building new releationships from old. In a preorder without a monoidal structure, the relations are chained in series.
![wiring_diagrams_1](https://cmbbq.github.io/img/wiring_diagrams_1.png)

With a symmetric monoidal structure, relations could be arranged in parallel as well.
![wiring_diagrams_2](https://cmbbq.github.io/img/wiring_diagrams_2.png)
The whole wiring diagram above says "if $t\le v, w+u\le x+z, v+x\le y$, then $t+u\le y+z$".

We could draw two wires in parallel to represent the monoidal product of two labels.
![wiring_diagrams_3](https://cmbbq.github.io/img/wiring_diagrams_3.png)
The validity of the box above corresponds to $x_1\otimes x_2 \le y_1 \otimes y_2 \otimes y_3$.

TBD

[^1]: To be a bit more rigorous, it's often useful to replace $=$ with $\cong$ throughout **Def 2.1**.