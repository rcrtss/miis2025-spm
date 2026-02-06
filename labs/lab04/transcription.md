# Lab 4 - LDPC Factor Graph Construction

This is the first of the two colab notebooks that form the project.

Here you will construct a **factor graph** for decoding a (regular)
**low-density parity-check (LDPC)** code. Although communication systems often
refer to messages and codewords, in this lab we model only the transmitted bits
$x$ and the channel outputs $y$. All other steps are either preprocessing
(encoding) or postprocessing (reading off the decoded bits).

As the block length $N$ increases, well-designed LDPC codes achieve better decoding performance, while the treewidth of the factor graph grows, making exact inference intractable and motivating approximate methods such as belief propagation.


## Learning goals

- Represent an LDPC code as a **factor graph** with:
  - binary **variable nodes** (bits),
  - **parity-check factors** (XOR constraints),
  - **channel factors** connecting each transmitted bit to its channel output.
- Validate the graph:
  - **structurally**: degrees, factor arities, correct wiring,
  - **semantically**: parity-check factors behave correctly on a known satisfying assignment.

You will be given:
- a generator for a *regular* parity-check matrix $H$ (with configurable density),
- a Binary Symmetric Channel (BSC) noise generator.

You will implement:
1. a parity-check factor constructor,
2. a channel factor constructor,
3. the factor-graph construction from $H$,
4. graph validation checks and visualization,
5. treewidth scaling analysis.

# Factor graphs for Low-Density Parity-Check (LDPC) decoding

## Variables and factorization

The task is to transmit $N$ bits over a noisy channel. LDPC codes introduce $M$
parity-check constraints to enable error correction.

The factor graph contains two types of binary variables:
- **transmitted bits**: $x_0,\dots,x_{N-1}$
- **channel outputs**: $y_0,\dots,y_{N-1}$

The joint distribution defined by the factor graph factorizes as
$$
p(x,y)
\;\propto\;
\prod_{m=1}^M \psi_m\bigl(x_{N(m)}\bigr)
\;\;
\prod_{n=0}^{N-1} \phi_n(x_n, y_n),
$$

where $N(m)$ denotes the set of transmitted-bit indices that appear in factor
$\psi_m$, and:
- $\psi_m$ are **parity-check factors** enforcing local constraints on transmitted bits,
- $\phi_n$ are **channel factors** modeling the noise affecting each bit.

## Parity-check factors $\psi_m(x_{N(m)})$

Each parity-check factor enforces an even-parity constraint on a small subset of
transmitted bits.

Formally, the factor is defined as
$$
\psi_m\bigl(x_{N(m)}\bigr)
\;=\;
\begin{cases}
1 & \text{if } \sum_{n \in N(m)} x_n \equiv 0 \pmod 2, \\
0 & \text{otherwise}.
\end{cases}
$$

Thus, each parity-check factor is a **local constraint** that involves only the
$k$ transmitted-bit variables connected to it.

## Channel factors $\phi_n(x_n,y_n)$

Each channel factor connects a transmitted bit $x_n$ to its corresponding channel
output $y_n$.

We model the channel as a **binary symmetric channel (BSC)** with flip probability
$f$, defining
$$
\phi_n(x_n,y_n)
=
p(y_n \mid x_n)
=
\begin{cases}
1-f & \text{if } y_n = x_n, \\
f   & \text{if } y_n \neq x_n.
\end{cases}
$$

Channel factors are independent across bit positions.

## Decoding as inference

In a communication scenario, the channel outputs
$y_0,\dots,y_{N-1}$ are observed.

Decoding consists of inferring the transmitted bits
$x_0,\dots,x_{N-1}$ given these observations, that is, computing the posterior
distribution $p(x \mid y)$.

In the next session, you will approximate this posterior using belief propagation on the same factor graph, after introducing evidence on the $y$ variables.

## Regular LDPC parity-check matrix generator

A **regular LDPC code** is specified by a sparse parity-check matrix $H \in \{0,1\}^{M \times N},$which defines the structure of the parity-check factors in the factor graph.

- Each row corresponds to one parity-check factor.
- Each column corresponds to one transmitted-bit variable.
- Each row has exactly $k$ ones (each check involves $k$ bits).
- Each column has exactly $j$ ones (each bit participates in $j$ checks).

For a given row $m$, the set
$$
N(m) = \{\, n : H_{m,n} = 1 \,\}
$$
is exactly the set of transmitted bits connected to parity-check factor $\psi_m$.

The following code generates a random **regular LDPC** parity-check matrix:
- each **column** has weight $j$,
- each **row** has weight $k$.

If generation fails (rare for small sizes), re-run the cell or adjust the parameters.

## Part 1: Choose code parameters and generate $H$

Pick modest sizes so you can visualize and test easily.
Suggested starting point:
- $N=24$, $j=3$, $k=6$  (then $M=\frac{N j}{k} = 12$)

## Part 2: Build factors

### 2.1 Parity-check factors $\psi_m(x_{N(m)})$

Implement `make_parity_check_factor` which receives a matrix $H$, a set of variables, and returns a factor defined over the transmitted-bit variables connected to that row.

It should be:
- value = 1 if the parity constraint is satisfied (even parity)
- value = 0 otherwise

Tips:
- With `DiscreteFactor`, you provide a `values` array of length $2^d$ (reshaped to `[2]*d`),
  where `d` is the number of variables in that factor.
- You can generate all assignments with `itertools.product([0,1], repeat=d)`.

$$
\begin{array}{ccc|c}
x_1 & x_2 & x_3 & \psi_{123} \\
\hline
0 & 0 & 0 & 1 \\
0 & 0 & 1 & 0 \\
0 & 1 & 0 & 0 \\
0 & 1 & 1 & 1 \\
1 & 0 & 0 & 0 \\
1 & 0 & 1 & 1 \\
1 & 1 & 0 & 1 \\
1 & 1 & 1 & 0 \\
\end{array}
$$

*Example of a 3-wise parity-check factor*

### 2.2 Channel factors $\phi_n(x_n,y_n)$

Implement `make_bsc_channel_factor(x_var, y_var, f)`,
a **pairwise factor** over `(x_var, y_var)` defined by a **binary symmetric channel (BSC)** with flip probability $f$.

## Part 3: Create the factor graph from $H$

Construct a `pgmpy` `FactorGraph`:

1. add variable nodes: `x0...x{N-1}`
2. add parity-check factors (one per row of H) and connect them
3. (optional today, required later) add likelihood factors and connect them

Implement:
- `build_ldpc_factor_graph(H, y=None, f=None)`

## Part 4: Validating the factor graph

We must verify that the factor graph correctly represents the intended LDPC model.

Your task in this part is to **think about validations that make
sense**, and to implement some of them.

Examples of questions you may consider include:
### Structural checks:

- Does the number and type of factors match the specification of the LDPC code?
- Are parity-check factors connected to the correct subsets of transmitted bits?
- Do variable nodes have the expected degrees (for a regular code)?
- Do parity-check factors behave correctly on a known satisfying assignment?
### Semantic checks:
For small $N$, you can test the parity-check factors exhaustively.

For each assignment $x \in \{0,1\}^N$ (there are $2^N$ of them), compute the so-called *syndrome*
$$
s = Hx \pmod 2.
$$
The parity-check constraints are satisfied if and only if $s=0$, so the product of
all parity-check factors should evaluate to
$$
\prod_{m=1}^M \psi_m(x_{N(m)}) =
\begin{cases}
1 & \text{if } Hx \equiv 0 \pmod 2,\\
0 & \text{otherwise.}
\end{cases}
$$

For this semantic test you can ignore the $y$ variables and the channel factors.

## Part 5: Visualization

Implement code that visualizes a factor graph.

You can use `networkx` and `matplotlib.pyplot`, and all the functions from that library such as `nx.draw_networkx_nodes`

## Part 6: Treewidth scaling with block length $N$

To study how inference complexity scales with the block length $N$, we analyze the **treewidth** of the LDPC code.

We first construct the **primal graph** on the transmitted bits $x_0,\dots,x_{N-1}$: this is an undirected graph in which two bits are connected by an edge if they appear together in any parity-check factor. It is analogous to moralization, but applied to a factor graph rather than a Bayesian network.

Since computing treewidth exactly is NP-hard, we estimate it using a
**min-fill heuristic**, which provides an upper bound on the treewidth.

Use `networkx.algorithms.approximation.treewidth_min_fill_in` to estimate the
treewidth of the primal graph for increasing values of $N$, and plot the result. When do you think inference will start being intractable?
