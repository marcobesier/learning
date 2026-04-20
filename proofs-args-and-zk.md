Below, you'll find solutions to exercises of Justin Thaler's excellent book, _Proofs, Arguments and Zero-Knowledge_.

## Chapter 3: Definitions and Technical Preliminaries

### Exercise 3.1

Let $A, B, C$ be $n \times n$ matrices over a field $\mathbb{F}$.
In Section 2.2, we presented a randomized algorithm for checking that $C = A \cdot B$.
The algorithm picked a random field element $r$, let $x = \left( r, r^2,...,r^n \right)$, and output EQUAL if $Cx = A \cdot (Bx)$, and output NOT-EQUAL otherwise.
Suppose instead that each entry of the vector $x$ is chosen independently and uniformly at random from $\mathbb{F}$. 
Show that:

- If $C_{ij} = (AB)_{ij}$ for all $i = 1,...,n, j = 1,...,n$, then the algorithm outputs EQUAL for every possible choice of $x$.

- If there is even one $(i,j) \in [n] \times [n]$ such that $C_{ij} \neq (AB)_{ij}$, then the algorithm outputs NOT-EQUAL with probability at least $1 - 1/|\mathbb{F}|$.

#### Solution

First, let $x = (x_1,...,x_n)\in \mathbb{F}^n$, where each $x_i$ is chosen independently and uniformly at random from $\mathbb{F}$.
Furthermore, let $D = A \cdot B$, so that our goal is to determine whether the $\textit{claimed}$ product matrix $C$ actually equals the $\textit{true}$ product matrix $D$.

The first property (completeness) is easy to see:
if $C=D$, then clearly $Cx=Dx$ for any $x\in \mathbb{F}^n$, so the algorithm will output EQUAL for every choice of $x$.

To see that the second property (soundness) holds, let us define the matrix $E := C-D$.
Furthermore, suppose $C\neq D$, i.e., $E\neq 0$.
Since $E$ is non-zero, we have that $\text{rank}(E) \geq 1$.
By the [rank-nullity theorem](https://en.wikipedia.org/wiki/Rank%E2%80%93nullity_theorem), we then have that $\text{dim(ker}(E)) \leq n-1$.
Since our vector coordinates $x_i$ are sampled independently and uniformly at random from $\mathbb{F}$, the soundness error is given by 

$$
\text{Pr}_{x\in \mathbb{F}^n}[Ex=0] = \frac{|\text{ker}(E)|}{|\mathbb{F}^n|}\leq \frac{|\mathbb{F}^{n-1}|}{|\mathbb{F}^{n}|} = \frac{1}{|\mathbb{F}|}
$$

In other words, if there is even one $(i,j) \in [n] \times [n]$ such that $C_{ij} \neq (AB)_{ij}$, then the algorithm outputs NOT-EQUAL with probability at least $1 - 1/|\mathbb{F}|$.

### Exercise 3.2

In Section 2.1, we described a communication protocol of logarithmic cost for determining whether Alice's and Bob's input vectors are equal.
Specifically, Alice and Bob interpreted their inputs as degree-$n$ univariate polynomials $p_a$ and $p_b$, chose a random $r \in \mathbb{F}$ with $|\mathbb{F}|>>n$, and compared $p_a(r)$ to $p_b(r)$.
Give a different communication protocol in which Alice and Bob interpret their inputs as multilinear rather than univariate polynomials over $\mathbb{F}$.
How large should $\mathbb{F}$ be to ensure that the probability Bob outputs the wrong answer is at most $1/n$?
What is the communication cost in bits of this protocol?

#### Solution

For the protocol to work, we assume (without loss of generality) that Alice's and Bob's vectors are both of length $2^l$.
Whenever that's not the case, Alice and Bob agree to pad their vectors with zeros until the length becomes the next power of two available.

With that assumption, Alice's vector $a = (a_0,a_1,...,a_{2^l-1})\in \mathbb{F}^{2^l}$ can be viewed as defining a function $f_a: \{0,1\}^l \rightarrow \mathbb{F}$.
Let $\tilde{f}_a:\mathbb{F}^l\rightarrow \mathbb{F}$ denote the corresponding multilinear extension.

Similarly, Bob's vector $b = (b_0,b_1,...,b_{2^l-1})\in \mathbb{F}^{2^l}$ can be viewed as defining a function $f_b: \{0,1\}^l \rightarrow \mathbb{F}$ with a corresponding multilinear extension $\tilde{f}_b:\mathbb{F}^l\rightarrow \mathbb{F}$.

The steps of the protocol are as follows:

1. Alice and Bob agree to encode their vectors into multilinear extensions as described above.
2. Alice chooses a random point $r\in \mathbb{F}^l$.
3. Alice sends $r$ and $\tilde{f}_a(r)$ to Bob.
4. Bob computes $\tilde{f}_b(r)$ and checks if it's equal to $\tilde{f}_a(r)$.

Now, how large should $\mathbb{F}$ be to ensure that the probability Bob outputs the wrong answer is at most $1/n$?
To answer this question, note that if $f_a\neq f_b$, then $\tilde{f}_a - \tilde{f}_b$ is a non-zero multilinear polynomial of degree $\leq l$.
By the [Schwartz-Zippel Lemma](https://en.wikipedia.org/wiki/Schwartz%E2%80%93Zippel_lemma), the probability that $\tilde{f}_a(r)=\tilde{f}_b(r)$ for a randomly chosen $r\in \mathbb{F}^l$ is at most:

$$
\frac{\text{deg}\left(\tilde{f}_a-\tilde{f}_b\right)}{|\mathbb{F}|} \leq \frac{l}{|\mathbb{F}|}
$$

To ensure the error is $\leq 1/n$, set:

$$
\frac{l}{|\mathbb{F}|} \leq \frac{1}{n} \Rightarrow |\mathbb{F}| \geq l\cdot n = l\cdot 2^l
$$

Lastly, to compute the protocol's total communication cost in bits, recall that the number of bits needed to represent an element of $\mathbb{F}$ is

$$
\left \lceil{\text{log}_2|\mathbb{F}|}\right \rceil
$$

because that's the number of bits needed to represent integers up to $|\mathbb{F}|-1$.

During the protocol, Alice sends:

- The vector $r\in \mathbb{F}^l$, i.e., $l\cdot \left \lceil{\text{log}_2|\mathbb{F}|}\right \rceil$ bits.
- The value $\tilde{f}_a(r)\in \mathbb{F}$, i.e., $\left \lceil{\text{log}_2|\mathbb{F}|}\right \rceil$ bits.

Therefore, the protocol's total communication cost in bits is $(l+1)\cdot \left \lceil{\text{log}_2|\mathbb{F}|}\right \rceil$ bits.

To see how efficient this is, let's consider an example.
Suppose $n=1024=2^{10}$, i.e., $l=10$. 
To ensure the soundness error is $\leq 1/n$, we choose $p \geq l\cdot n = 10\cdot 1024 = 10240$.
With this choice, we have $\text{log}_2p \approx 14$.
The total communication cost in bits is, therefore, given by:

$$
(10+1)\cdot 14 = 154 \text{ bits}
$$

Compared to naively sending the whole vector (1024 field elements = 14,336 bits), our protocol gives an exponential saving.


### Exercise 3.3

Let $p = 11$. Consider the function $f:\{0,1\}^2\rightarrow\mathbb{F}_p$ given by $f(0,0)=3, f(0,1)=4, f(1,0)=1$, and $f(1,1)=2$.
Write out an explicit expression for the multilinear extension $\tilde{f}$ of $f$. 
What is $\tilde{f}(2,4)$?

Now consider the function $f: \{0,1\}^3\rightarrow \mathbb{F}_p$ given by $f(0,0,0)=1, f(0,1,0)=2, f(1,0,0)=3, f(1,1,0)=4, f(0,0,1)=5, f(0,1,1)=6, f(1,0,1)=7, f(1,1,1)=8$.
What is $\tilde{f}(2,4,6)$?
How many field multiplications did you perform during the calculation?
Can you work through the calculation of $\tilde{f}(2,4,6)$ that uses "just" 20 multipliactions operations?
Hint: see Lemma 3.8.

#### Solution

Let $p = 11$. Consider the function $f:\{0,1\}^2\rightarrow\mathbb{F}_p$ given by $f(0,0)=3, f(0,1)=4, f(1,0)=1$, and $f(1,1)=2$.
Using Lemma 3.6, we can compute $\tilde{f}$ via:

$$
\begin{aligned}
\tilde{f}(x_1, x_2) &= \sum_{w\in \{0,1\}^2}f(w)\cdot \prod_{i=1}^2 (x_i w_i + (1-x_i)(1-w_i))\\
&= 3(1-x_1)(1-x_2) + 4(1-x_1)x_2 + x_1(1-x_2) + 2x_1x_2
\end{aligned}
$$

Consequently, we have $\tilde{f}(2,4) = 3$.

Now consider the function $f: \{0,1\}^3\rightarrow \mathbb{F}_p$ given by $f(0,0,0)=1, f(0,1,0)=2, f(1,0,0)=3, f(1,1,0)=4, f(0,0,1)=5, f(0,1,1)=6, f(1,0,1)=7, f(1,1,1)=8$.
Using Lemma 3.6, we can compute $\tilde{f}$ via

$$
\tilde{f}(x_1, x_2, x_3) = \sum_{w\in \{0,1\}^3}\tilde{f}_w(x_1,x_2,x_3)
$$

where:

$$
\begin{aligned}
\tilde{f}_{(0,0,0)}(x_1, x_2, x_3) &= 1\cdot (1-x_1)\cdot (1-x_2)\cdot (1-x_3) \\
\tilde{f}_{(0,1,0)}(x_1, x_2, x_3) &= 2\cdot(1-x_1)\cdot x_2\cdot(1-x_3) \\
\tilde{f}_{(1,0,0)}(x_1, x_2, x_3) &= 3\cdot x_1\cdot (1-x_2)\cdot (1-x_3) \\
\tilde{f}_{(1,1,0)}(x_1, x_2, x_3) &= 4\cdot x_1\cdot x_2\cdot (1-x_3) \\
\tilde{f}_{(0,0,1)}(x_1, x_2, x_3) &= 5\cdot (1-x_1)\cdot (1-x_2)\cdot x_3 \\
\tilde{f}_{(0,1,1)}(x_1, x_2, x_3) &= 6\cdot (1-x_1)\cdot x_2\cdot x_3 \\
\tilde{f}_{(1,0,1)}(x_1, x_2, x_3) &= 7\cdot x_1\cdot (1-x_2)\cdot x_3 \\
\tilde{f}_{(1,1,1)}(x_1, x_2, x_3) &= 8\cdot x_1\cdot x_2\cdot x_3 \\
\end{aligned}
$$

Consequently, we can compute $\tilde{f}(2,4,6) = 33 \text{ mod } 11 = 0$ using a total of 24 multiplications.
Using the memoization procedure from Lemma 3.8, we can reduce the number of required multiplications to 20.
To see that, notice that each of the following four multiplications appears twice in the above expressions for the $\tilde{f}_w$.

- $(1-x_1)\cdot (1-x_2)$
- $(1-x_1)\cdot x_2$
- $x_1\cdot (1-x_2)$
- $x_1\cdot x_2$

Therefore, it suffices to compute each of them only once instead of evaluating each instance at $(2,4,6)$ independently.
As a result, we only need a total of $20 = 24 - 4$ multiplications to compute $\tilde{f}(2,4,6)$.


### Exercise 3.4

Fix some prime $p$ of your choosing.
Write a Python program that takes as input an array of lenght $2^l$ specifying all evaluations of a function $f: \{0,1\}^l \rightarrow \mathbb{F}_p$ and a vector $x\in \mathbb{F}_p^l$, and outputs $\tilde{f}(x)$.

### Solution

Recall that the multilinear extension $\tilde{f}: \mathbb{F}_p^l\rightarrow \mathbb{F}_p$ of a function $f: \{0,1\}^l\rightarrow\mathbb{F}_p$ is defined as

$$
\tilde{f}(x) = \sum_{w\in \{0,1\}^l}f(w)\cdot \chi_w(x)
$$

where:

$$
\chi_w(x)=\prod_{i=1}^l[(1-x_i)\text{ if }w_i=0\text{, else }x_i]
$$

We can, therefore, implement the multilinear extension in Python as follows:

```python
from itertools import product

def eval_multilinear_extension(f_vals, x, p):
    """
    Evaluate the multilinear extension of a function f: {0,1}^l -> F_p at point x in F_p^l.

    Inputs:
      - f_vals: List of 2^l field elements IN LEX ORDER(!) (w in {0,1}^l)
      - x: List of length l, point in F_p^l
      - p: Prime field modulus

    Output:
      - tilde_f(x) in F_p
    """
    l = len(x)
    result = 0
    for idx, w in enumerate(product([0, 1], repeat=l)):
        chi = 1
        for i in range(l):
            chi *= (1 - x[i]) if w[i] == 0 else x[i]
            chi %= p
        result += f_vals[idx] * chi
        result %= p
    return result
```

Notice that the function assumes that the input list `f_vals` is ordered according to [lexicoraphic order on bitstrings](https://www.geeksforgeeks.org/lexicographic-rank-of-a-binary-string/).
That is, the input values $f(w)$ must correspond to $w\in \{0,1\}^l$ ordered like:

$$
(0,0,...,0),(0,0,...,1),...,(1,...,1,0),(1,...,1,1)
$$

This ordering is generated by `product([0, 1], repeat=l)` in the above code and must match exactly.
If this assumption is violated, the output will be incorrect.

As an example, here's how to correctly reproduce $\tilde{f}(2,4,6)$ from the previous exercise:

```python
# Input values
p = 11
# indices map to:
# (0,0,0), (0,0,1), (0,1,0), (0,1,1), (1,0,0), (1,0,1), (1,1,0), (1,1,1)
f_vals = [1, 5, 2, 6, 3, 7, 4, 8]
x = [2, 4, 6]

# Output: 0 (= 33%11)
print(eval_multilinear_extension(f_vals, x, p))
```

If we had passed the `f_vals` in a way that would result in non-lexicographic order of the values $w\in \{0,1\}^l$ instead, our function would have given a wrong result:

```python
# Input values
p = 11
# indices map to:
# (0,0,0), (0,1,0), (1,0,0), (1,1,0), (0,0,1), (0,1,1), (1,0,1), (1,1,1)
# Note: this order is NOT LEXICOGRAPHIC
f_vals = [1, 2, 3, 4, 5, 6, 7, 8]
x = [2, 4, 6]

# Output: 1
print(eval_multilinear_extension(f_vals, x, p))
```
