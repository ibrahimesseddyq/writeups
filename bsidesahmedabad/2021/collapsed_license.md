# collapsed_license

**CTF:** BSidesAhmedabad 2021  
**Category:** rev  
**Original writeup:** <https://mystiz.hk/posts/2021/2021-11-07-bsides-ahmedabad/> — by **mystiz.hk**. All credit goes to the original author; this is an archived copy.

---

Collapsed License⌗

Challenge Summary⌗

We are given a binary executable compiled in C++ (luckily unstripped). It reads a "license file" and valiates it, and prints us the flag if the license file is valid.

Part I: What is a valid license?⌗

I understood the validation logic after spending some time reversing the binary. A valid license file should be 8192 bytes long (i.e., 65536 bits). It is converted into a $256 \times 256$ grid with each entry being either 0 or 1. There should be exactly 256 set bits, satisfying both conditions:

Each row (and each column) should contain exactly one set bit.

Suppose that $(x_1, y_1)$ and $(x_2, y_2)$ are two distinct set bits in the grid. If $(x_1', y_1')$ and $(x_2', y_2')$ are two set bits satisfying $x_1 - x_2 = x_1' - x_2'$ and $y_1 - y_2 = y_1' - y_2'$, then $(x_1', y_1') = (x_1, y_1)$ and $(x_2', y_2') = (x_2, y_2)$.

Let $(0, y_0), (1, y_1), ..., (63, y_{63})$ be the coordinates of the set bits. The below conditions are equivalent to the above ones:

$y_0, y_1, ..., y_{63}$ are all distinct, and

$y_1 - y_0, y_2 - y_1, ..., y_{63} - y_{62}$ are all distinct, and

$y_2 - y_0, y_3 - y_1, ..., y_{63} - y_{61}$ are all distinct, and

...

$y_{62} - y_0, y_{63} - y_1$ are all distinct.

What is the array called? According to ptr-yudai, the array $y_0, y_1, …, y_{n-1}$ is called a Costas array. We need to find such an array when $n = 256$.

Part II: A symbolic solution⌗

I wrote a script with Z3. It worked for $n$ is as small as 10-ish. When $n = 256$ my laptop crashed after running the program for 30 minutes.

from z3 import * n = 256 s = Solver() y = [Int( f 'y { i } ' ) for i in range(n)] for i in range(n): s . add(And(y[i] >= 0 , y[i] < n)) s . add(Distinct(y)) for d in range( 1 , n): s . add(Distinct([y[i] - y[i - d] for i in range(d, n)])) assert s . check() == sat md = s . model() y0 = [md . evaluate(y[i]) . as_long() for i in range(n)] grid = [[ 0 for _ in range( 256 )] for _ in range( 256 )] for x, y in enumerate(y0): grid[x][y] = 1 with open( 'test' , 'wb' ) as f: for i in range( 256 ): for j in range( 0 , 256 , 8 ): by = sum(b << k for k, b in enumerate(grid[i][j:j + 8 ])) by = bytes([by]) f . write(by)

Part III: The adrenaline rush when my laptop was crashed⌗

I am really excited able to think of the correct solution.

I was forced to work on paper and pen when my laptop was crashed. Suppose that $(0, y_0), (1, y_1), ..., (63, y_{63})$ are the coordinates of the set bits.

There are a number of failed attempts, like $y_k = \sum_{i=0}^{k} i$. At some point of time I think of primitive roots, which led me to the final solution.

Claim. Let $g$ be a primitive root under modulo 257. Below is a valid solution:

\[y_k = (g^k\ \text{mod}\ 257)-1,\]

for $k = 0, 1, 2, ..., 255$.

Proof.

Since g is a primitive root under modulo 257, $g^0, g^1, ..., g^{255}$ are all distinct. That said, $y_k$'s are all distinct.

Also, let $d > 0$. Suppose that $y_{i+d} - y_i = y_{j + d} - y_j$ for $0 \leq i, j \leq 255-d$.

\[\begin{aligned} & g^i (g^d - 1) \equiv y_{i+d} - y_i \equiv y_{j + d} - y_j \equiv g^j (g^d - 1)\ (\text{mod}\ 257)\\ & \Longrightarrow i = j \qquad\qquad\qquad\qquad\qquad\qquad\qquad\qquad\qquad\qquad\blacksquare \end{aligned}\]

I substituted $g = 3$ to created a license file. Sending the license to the server, we have the flag:
