# PellesRotor-SupportedArithmetic

**CTF:** MidnightSun 2022 Quals  
**Category:** crypto  
**Original writeup:** <https://github.com/death-of-rats/CTF/tree/master/2022/midnightsun/pelle-rotor> — by **death-of-rats**. All credit goes to the original author; this is an archived copy.

---

# Pelle's rotor supported arithmetic

## Challenge

```

#!/usr/bin/python3

from sys import stdin, stdout, exit

#from flag import FLAG

from secrets import randbelow

from gmpy import next_prime

from Crypto.Util.number import bytes_to_long, long_to_bytes

FLAG = bytes_to_long(b'mf{testowa_flaga_costam_costam}')

p = int(next_prime(randbelow(2**512)))

q = int(next_prime(randbelow(2**512)))

n = p * q

e = 65537

phi = (p - 1)*(q - 1)

d = int(pow(e, -1, phi))

d_len = len(str(d))

print("encrypted flag", pow(FLAG, 3331646268016923629, n))

stdout.flush()

ctr = 0

def oracle(c, i):

global ctr

if ctr > 10 * d_len // 9:

print("Come on, that was already way too generous...")

return

ctr += 1

rotor = lambda d, i: int(str(d)[i % d_len:] + str(d)[:i % d_len])

return int(pow(c, rotor(d, i), n))

banner = lambda: stdout.write("""

Pelle's Rotor Supported Arithmetic Oracle

1) Query the oracle with a ciphertext and rotation value.

2) Exit.

""")

banner()

stdout.flush()

choices = {

1: oracle,

2: exit

}

while True:

try:

choice = stdin.readline()

print("c:")

stdout.flush()

cipher = stdin.readline()

print("rot:")

stdout.flush()

rotation = stdin.readline()

print(choices.get(int(choice))(int(cipher), int(rotation)))

stdout.flush()

except Exception as e:

stdout.write("%s

" % e)

stdout.flush()

exit()

```

## TL;TR;

Encrypting at least 2 sets of messages: $m_3 = m_2\ m_1$, we can find $N$: `k*N = gcd(c1_1*c1_2-c1_3, c2_1*c2_2-c2_3)`, and get ride of `k` by dividing the result by small primes.

Having $N$ we retrive $d$ digit by digit:

$$ msg^{10\ d_r} = \frac{ c_{i+1}+kN}{msg^{digit}} $$

$$ c_i^{10} = msg^{digit\ 10^{digitCount}}\ msg^{10\ d_r} $$

## Solution

I solved this challenge after the CTF.

The script allows us the opportunity to encrypt our messages using d, the modular multiplicative inverse of e. We can encrypt more times than d has digits. This gives us a spare to calculate N.

How to find N?

$$ c_1\ c_2 = m_1^d\ m_2^d = (m_1\ m_2)^d = c_3 $$

For modulo operation:

$$ (m_1^d)\ mod\ N\ (m_2^d)\ mod\ N = (m_1\ m_2)^d\ mod\ N + kN $$

$$ c_1\ c_2 - c_3 = kN $$

If we choose good messages so that $c_1\ c_2 > N$.

Prepare 2 or 3 sets of messages. Using `gcd(k1*N, k2*N)` we should get N. But sometimes `gcd` still returns muiltiple of N. For assurance, one may try to divide it by small primes.

So we have N.

Next d.

"Size" (number of digits) of $d$ cannot be greater than the number of digits of N. We can send the same message to encryption with i = [0... digits count for N] and compare results. When it will be equal to cipher for i=0. And we have the "size" of $d$.

Next, what are adjacent ciphers look like:

$$ d_{i} = digit\ 10^{digitCount-1} + d_r$$

$$ d_{i+1} = 10\ d_r + digit $$

$$ c_{i} = msg^{d_{i}} = msg^{digit\ 10^{digitCount-1}}\ msg^{d_r} $$

$$ c_{i+1} = msg^{d_{i+1}} = msg^{10\ d_r}\ msg^{digit} $$

$msg^{digit}$ for small msg is so small that iterating we can find $k$ that would fullfill condition:

$$ c_{i+1} + kN\ mod\ msg^{digit} == 0 $$

and next:

$$ msg^{10\ d_r} = \frac{ c_{i+1}+kN}{msg^{digit}} $$

Using $c_1$ to 10th power (**all modulo N**) we can compare cipher with out calculation for choosen $digit$:

$$ c_i^{10} = msg^{digit\ 10^{digitCount}}\ msg^{10\ d_r} $$

So I check all digits, which one meets the condition.

Havind $d$ we can calculate $\phi = d\ e-1$ and retrive the flag:

```

fi = e*d-1

D = pow(flag_power, -1, fi)

M = pow(flag_cipher, D, N)

print(f"{long_to_bytes(M)}")

```

[sol.py](https://github.com/death-of-rats/CTF/blob/master/2022/midnightsun/pelle-rotor/sol.py)
