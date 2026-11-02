# Chemistry_Lab

**CTF:** TheCyberJawaraInternational 2024  
**Category:** crypto  
**Original writeup:** <https://gist.github.com/Lydxn/26679096f7675abc3eb30b7fbfc07541#chemistry-lab-1-solve> — by **Lydxn**. All credit goes to the original author; this is an archived copy.

---

Chemistry Lab (1 solve)

Checkout my new chemistry experiment on using Deoxyribonucleic acid (DNA) to encrypt images securely :) nc 68.183.177.211 4444

This challenge is somewhat overwhelming in the amount of matrix/string transformations that it tries to perform, but it is actually quite straightfoward if we keep track of everything that is going on.

There are several global values that stay constant throughout the entire program:

height/width : the height and width of flag.png .

: the height and width of . dna_encoded_flag : the DNA-encoded representation of flag.png 's RGB pixel matrix.

: the DNA-encoded representation of 's RGB pixel matrix. formula : a random height by width matrix of bytes.

: a random by matrix of bytes. dna_encoded_formula : the DNA-encoded representation of formula .

: the DNA-encoded representation of . lx/ly : a random permutation of size width and height , respectively.

All of these are unknown, except for height and width which can be easily extracted from the protocol.

The program converts often between two "representations", the matrix form and its corresponding DNA form, which is a string of ATCG s. The DNA form is mostly used to obtain a nice, printable form of the matrix, so we can mostly ignore it and focus on the matrix transformations.

We have three options upon connecting to the server:

Get encrypted formula

Get encrypted input

Get encrypted flag

The last option is only available upon guessing formula , so the goal will be to somehow find formula using the other two protocols.

Get encrypted formula returns formula , but it is encrypted by the following procedure:

The rows and columns are shuffled using the indices of the permutations lx and ly .

and . The matrix is XORed with a key_matrix that is randomly generated each time.

that is randomly generated each time. The DNA-encoded representation of the matrix is returned.

Get encrypted input works pretty much the same, except formula is replaced by our own input. I left a few details out, like how the matrices can sometimes be of different sizes and how that affects the result, but for simplicity we can put those thoughts to the side, since it doesn't change the overall algorithm.

Notice that when we call Get encrypted input with a matrix of all zeros, the row/column shuffling doesn't do anything, which eliminates one element of unknown. That leaves the XOR, which returns the XOR of key_matrix and our input. But our input is all zeros, so we have just leaked key_matrix itself! But what can we do with key_matrix ? It's a one-time value, so it appears to give no useful information.

The trick is that the RNG used to generate key_matrix (and in fact all of the other randomly-generated values) is random.randbytes() . It is well-known that CPython's random is insecure as a PRNG. There are tools that can determinstically crack the PRNG after 624 known outputs. And key_matrix holds way more than 624 outputs, so we can easily determine all of the subsequent and previous randomly-generated values. This includes formula , lx and ly .

I borrowed this library to crack the Mersenne Twister state and retrieve all of those values. This is actually more tricky than it seems, due to the fact that lx and ly are generated using random.randint() , which internally calls the following function:

def randbelow ( n ): k = n . bit_length () # don't use (n-1) here because n can be 1 r = getrandbits ( k ) # 0 <= r < 2**k while r >= n : r = getrandbits ( k ) return r

As we can see, this function is only able to grab k bits at a time, and will throw away that output if it is above the bound. In other words, there are skipped states. lx and ly are generated right after formula is generated, so the number of skipped states in between is unknown to us. Fortunately, we can "brute-force" the number of skipped states by rewinding the state back x times (where x is in some range) and starting the RNG from there, checking if the recovered values match what we expect by using them to re-generate the encrypted formula.

We don't know exactly what x is, but we can bound it to some small range using some statistical analysis. Once we know for certain what the PRNG state is a the beginning of the program, we can easily recover the encrypted flag, since we know all of the relevant secret parameters.

Below is my full solve script:

from pwn import * from chall import * from PIL import Image from mersenne_twister import MT19937 import numpy as np import random def get_encrypted_formula (): io . sendline ( b'1' ) io . recvuntil ( b'Encrypted formula:
