# Filter_Collector

**CTF:** LexingtonInformaticsTournament 2021  
**Category:** crypto  
**Original writeup:** <https://github.com/AC01010/CTF-Writeups/blob/master/LITCTF2021/filter-collector.md> — by **AC01010**. All credit goes to the original author; this is an archived copy.

---

**Filter Collector**

**303 points, 9 solves**

It is clear that we cannot recover enough information to recover the entire flag in one connection, so we must figure out a way to recover a portion of the flag. Given that we are able to input a modulus for all computations to be computed in, the obvious decision is to use CRT to recover the flag.

However, we first have to figure out how to recover the flag, which has been partitioned; to do this, we must somehow find the sum of every 7th integer in the polynomial.

Say that we have 7 elements of the multiplicative group of integers; we know that for any prime modulus, the group is cyclic. We also know that the flag partitions in the polynomial are in terms x^m, where m is any multiple of 7.

Thus, we search for a cyclic group of order 7 in the finite field of any arbitrary modulus, as long as it is prime (this is a requirement for CRT). We input into the connection all 6 non-one elements of the group, and collect the output. Define the elements G_i as b^i for all positive integral i less than 7.

Let us define the polynomial to be expressed as a_i * x^i for all positive integral i less than 200. Then, the 6 outputs are:

a_0 + a_1 * G_1 + a_2 * G_2 + a_3 * G_3 + a_4 + G_4 + a_5 * G_5 + a_6 * G_6 ...

a_0 + a_1 * G_2 + a_2 * G_3 + a_3 * G_4 + a_4 + G_5 + a_5 * G_6 + a_6 * G_1 ...

a_0 + a_1 * G_3+ a_2 * G_4 + a_3 * G_5 + a_4 + G_6 + a_5 * G_1 + a_6 * G_2 ...

a_0 + a_1 * G_4 + a_2 * G_5 + a_3 * G_6 + a_4 + G_1 + a_5 * G_2 + a_6 * G_3 ...

a_0 + a_1 * G_5 + a_2 * G_6 + a_3 * G_1 + a_4 + G_2 + a_5 * G_3 + a_6 * G_4 ...

a_0 + a_1 * G_6 + a_2 * G_1 + a_3 * G_2 + a_4 + G_3 + a_5 * G_4 + a_6 * G_5 ...

= 6 * a_0 + a_1 * Σ G_(1-6) + a_2 * Σ G_(1-6) + a_3* Σ G_(1-6) + a_4* Σ G_(1-6) + a_5 * Σ G_(1-6) + a_6 * Σ G_(1-6) ...

We then input 1 into the program; let this output be c. 6c = 6 * a_0 + 6 * a_1 + 6 * a_2 + 6 * a_3 + 6 * a_4 + 6 * a_5 + 6 * a_6 ... . Subtractng 6c from the sum of the previous 6 outputs, we arrive at:

a_1 * Σ G_(1-6)-6 + a_2 * Σ G_(1-6)-6 + a_3* Σ G_(1-6)-6 + a_4* Σ G_(1-6)-6 + a_5 * Σ G_(1-6)-6 + a_6 * Σ G_(1-6)-6 ...

We know each element of G, and we can simply compute Σ G_(1-6)-6. We can simply divide by multiplying this sum by inverse_mod(Σ G_(1-6)-6, m) to recover a_1+a_2+a_3+a_4+a_5+a_6. Subtracting this from c, we are able to recover the sum of every 7th partition (the flag), modulo a chosen modulus.

To recover the flag, we need to repeat this process enough times. Doing this, we are left with a list of residues and moduli. Using CRT, we are left with the flag.

Here's my full solve script:

```python

import sympy

from sympy.ntheory.modular import crt

from pwn import *

used = []

list_primes = []

for j in range(10000,12000): # find cyclic groups of order 7

if sympy.isprime(j):

for i in range(2,j):

if pow(i,7,j)==1 and j not in used:

l2 = []

sum = 0

for p in range(1,7):

l2.append(pow(i,p,j))

list_primes.append((l2,j))

used.append(j)

residues = []

modulos = []

for prime in list_primes:

r = remote("filtercollector.litctf.live",1337)

r.recvuntil("favorite mod: ")

r.sendline(str(prime[1]))

res = 0

def get_number(inp):

r.recvuntil("Input the a number: ")

r.sendline(str(inp))

r.recvuntil("filter is ")

return(int(r.recvline().strip()))

for i in prime[0]:

res+=get_number(i)

sum = get_number(1)

res-=(sum*6)

d = pow(prime[1]-7,-1,prime[1]) #do the computation to recover sum of every 7th partition

b = (res*d)%prime[1]

r = (sum-b)%prime[1]

residues.append(r)

modulos.append(prime[1])

print(bytes.fromhex(hex(crt(modulos,residues)[0])[2:]).decode()) #CRT and dec -> ascii

```
