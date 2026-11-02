# they_were_eleven

**CTF:** BSidesAhmedabad 2021  
**Category:** crypto  
**Original writeup:** <https://mystiz.hk/posts/2021/2021-11-07-bsides-ahmedabad/> — by **mystiz.hk**. All credit goes to the original author; this is an archived copy.

---

They Were Eleven⌗

Challenge Summary⌗

Suppose that we have a 111-byte message $m$. Let $p_1, p_2, ..., p_{11}, q_1, q_2, ..., q_{11}$ be 512-bit primes and $r_1, r_2, ..., r_{11} \in [0, 2^{11})$.

We are given $n_i = p_i\cdot q_i$ and $c_i = m^{11}\cdot r_i\ \text{mod}\ n_i$ for $i = 1, 2, ..., 11$. The objective is to recover $m$.

Part I: Comparing with the broadcast attack⌗

It looked very similar to broadcast attack. $e = 11$ and we have 11 messages encrypted with different moduli. The only difference is the each ciphertext is multiplied with a 11-bit number. I think of Chinese remainder theorem that is used by the broadcast attack (hereby $N_k = (\prod{n_i}) / n_k$):

\[c = \left(\sum_{k=1}^{11} c_i \cdot N_k \cdot ({N_k}^{-1}\ \text{mod}\ n_k)\right)\ \text{mod}\ N.\qquad[\dagger]\]

On a second thought, $c$ is not meaningful at all. Although $c$ is not useful, the expression actually helped.

Part II: When in doubt, use lattice⌗

One thing that caught my attention is the size of $m$. It is 111-byte long, which is a bit short when compiled to the moduli. Does it mean that it is a "short solution" of a system of equations? The picture is clearer if we substitute $c_k = m^{11}\cdot r_k$ on $[\dagger]$:

\[c = \left(\sum_{k=1}^{11} m^{11} \cdot r_k \cdot N_k \cdot ({N_k}^{-1}\ \text{mod}\ n_k)\right)\ \text{mod}\ N.\]

Right, this is still useless. The only difference is those problematic $r_i$'s. If we remove those $r_k$'s our world would be much cleaner:

\[m^{11} = \left(\sum_{k=1}^{11} m^{11} \cdot N_k \cdot ({N_k}^{-1}\ \text{mod}\ n_k)\right)\ \text{mod}\ N.\]

In particular, $m^{11}$ is $9768 = 111 \cdot 8 \cdot 11$ bits long. Since $N$ is $11264 = 1024 \cdot 11$ bits long, $m^{11}$ is considered short. Maybe we can find short $r_1, r_2, ..., r_{11}$ such that

\[m^{11} = \left(\sum_{k=1}^{11} c_k \cdot N_k \cdot ({N_k}^{-1}\ \text{mod}\ n_k) \cdot {r_k}^{-1}\right)\ \text{mod}\ N.\]

I don't know how to formulate ${r_k}^{-1}$ in lattice for small $r_k$'s. Luckily we can rewrite the equation (denote $R = \prod r_i$ and $R_k = R / r_k$) by multiplying $R$ on both sides:

\[m^{11} R = \left(\sum_{k=1}^{11} c_k \cdot N_k \cdot ({N_k}^{-1}\ \text{mod}\ n_k) \cdot R_k\right)\ \text{mod}\ N.\]

Well, it doesn't sound lucky at all. Hear me out: The left hand side is now 9889 bits long. There are 11 unknowns on the right hand side, namely $R_1, R_2, ..., R_{11}$. Each of them is 88 bits long. There are total of 10857 bits, which is still smaller than 11264. Now we can try putting the whole thing in a lattice:

\[\left[\begin{matrix} -1 & 1 & 0 & ... & 0 \\ c_1 \cdot N_1 \cdot {N_1}^{-1} & 0 & 1 & ... & 0 \\ &&\vdots \\ c_{11} \cdot N_{11} \cdot {N_{11}}^{-1} & 0 & 0 & ... & 1 \\ N & 0 & 0 & ... & 0 \end{matrix}\right]\]

Note. I used ${N_{k}}^{-1}$ as a shorthand for ${N_{k}}^{-1}\ \text{mod}\ n_k$. That was too long.

It is expected that $(0, m^{11}R, R_1, R_2, ..., R_{11})$ is an vector spanned by the lattice. It could be found assigning appropriate weights to the vectors and perform LLL. I exhausted $r_1$ and computed $R := r_1R_1$. After that, we can derive $m$ from $m^{11}R$ and $R$. Finally, those integral $m$'s could be the flag.

Neko{th3_5tud3nts_0f_spac3_univ3rsity_sh0u1d_b3_wis3_1ik3_a_cat}
