# mini_RSA

**CTF:** BackdoorCTF 2023  
**Category:** misc  
**Original writeup:** <https://sakibulalikhan.medium.com/backdoorctf-2023-mini-rsa-challenge-writeup-937984bd1e58> — by **sakibulalikhan.medium.com**. All credit goes to the original author; this is an archived copy.

---

Greetings, fellow cybersecurity enthusiasts and CTF players! In this writeup, we will dive into the solution of the “mini_RSA” challenge from the BackdoorCTF 2023.

Challenge Overview

Groot has encrypted a message using RSA with a key size of 512 bits and a public exponent (RSA_E) of 3. The challenge provides two files: script.py and output.txt . The former contains the RSA encryption script used by Groot, while the latter presents the output ciphertext (c) and the modulus (n).

Let’s explore the provided files and understand the challenge before revealing the solution.

The Encryption Script: script.py

# Import necessary modules

from Crypto.Util.number import getPrime, bytes_to_long, GCD

import random

import time



# Set the seed for randomness

random.seed(time.time())



# Define the flag to be encrypted (flag{REDACTED})

flag = b"flag{REDACTED}"



# Define RSA parameters

KEY_SIZE = 512

RSA_E = 3



# Function for fast modular exponentiation

def fast_exp(a, b, n):

output = 1

while b > 0:

if b & 1:

output = output * a % n

a = a * a % n

b >>= 1

return output



# Function to check RSA parameters

def check(p, q, n):

a_ = random.randint(1, 100)

b_ = random.randint(1, 100)

s = fast_exp(p, fast_exp(q, a_, (p - 1) * (q - 1)), n)

t = fast_exp(q, fast_exp(p, b_, (p - 1) * (q - 1)), n)

result = s + t

print(result)



# Function to generate RSA parameters

def gen_RSA_params(N, e):

p = getPrime(N)

q = getPrime(N)



while GCD(e, (p - 1) * (q - 1)) > 1:

p = getPrime(N)

q = getPrime(N)



n = p * q



check(p, q, n)

return (p, q, n)



# Generate RSA parameters and encrypt the flag

p, q, n = gen_RSA_params(KEY_SIZE, RSA_E)

m = bytes_to_long(flag)

c = pow(m, RSA_E, n)



# Print the ciphertext and modulus

print(c)

print(n)

This script generates random prime numbers p and q , checks some conditions, and then encrypts the flag using RSA. The check function is there to create a side channel but is not relevant to our solution.

The Output File: output.txt

s+t=24986288511406610689718446624210347240800254679541887917496550238025724025245366296475758347972917098615315083893786596239213463034880126152583583770452304

c=5926440800047066468184992240057621921188346083131741617482777221394411358243130401052973132050605103035491365016082149869814064434831123043357292949645845605278066636109516907741970960547141266810284132826982396956610111589

n=155735289132981544011017189391760271645447983310532929187034314934077442930131653227631280820261488048477635481834924391697025189196282777696908403230429985112108890167443195955327245288626689006734302524489187183667470192109923398146045404320502820234742450852031718895027266342435688387321102862096023537079

The output file provides the sum of two exponentiations s+t , the ciphertext c , and the modulus n .

My Solution Script: solve.py

# Import necessary modules

from Crypto.Util.number import bytes_to_long, long_to_bytes, getPrime, GCD

import random

import time



# Function for fast modular exponentiation

def fast_exp(a, b, n):

output = 1

while b > 0:

if b & 1:

output = output * a % n

a = a * a % n

b >>= 1

return output



# Function to check RSA parameters (not used in the solution)

def check(p, q, n):

a_ = random.randint(1, 100)

b_ = random.randint(1, 100)

s = fast_exp(p, fast_exp(q, a_, (p - 1) * (q - 1)), n)

t = fast_exp(q, fast_exp(p, b_, (p - 1) * (q - 1)), n)

result = s + t

print(result)



# Function to generate RSA parameters

def gen_RSA_params(N, e):

p = getPrime(N)

q = getPrime(N)



while GCD(e, (p - 1) * (q - 1)) > 1:

p = getPrime(N)

q = getPrime(N)



n = p * q



check(p, q, n)

return (p, q, n)



# Values from the output.txt

s_plus_t = 24986288511406610689718446624210347240800254679541887917496550238025724025245366296475758347972917098615315083893786596239213463034880126152583583770452304

c = 5926440800047066468184992240057621921188346083131741617482777221394411358243130401052973132050605103035491365016082149869814064434831123043357292949645845605278066636109516907741970960547141266810284132826982396956610111589

n = 155735289132981544011017189391760271645447983310532929187034314934077442930131653227631280820261488048477635481834924391697025189196282777696908403230429985112108890167443195955327245288626689006734302524489187183667470192109923398146045404320502820234742450852031718895027266342435688387321102862096023537079



# Generate RSA parameters

p, q, n = gen_RSA_params(512, 3)



# Calculate phi(n)

phi_n = (p - 1) * (q - 1)



# Calculate the modular inverse of e (RSA_E) modulo phi_n

d = pow(3, -1, phi



_n)



# Decrypt the ciphertext

m_decrypted = pow(c, d, n)



# Convert the decrypted message from long to bytes

flag_decrypted = long_to_bytes(m_decrypted)



# Print the decrypted flag

print(flag_decrypted.decode('utf-8'))

Explanation and Solution

The provided script uses RSA encryption with a small public exponent ( e = 3 ). To solve this challenge, we can use the provided ciphertext ( c ), modulus ( n ), and the knowledge that e = 3 to decrypt the message.

The solve.py script first generates new RSA parameters ( p , q , n ) using the gen_RSA_params function. However, the generated values are not used since we already have the correct values from output.txt .

The script then calculates phi(n) , the modular inverse of e (RSA_E) modulo phi(n) , and finally decrypts the ciphertext using modular exponentiation. The decrypted flag is then printed.

Conclusion

And there you have it! We successfully decrypted Groot’s message and unveiled the hidden flag. The mini_RSA challenge provided a beginner-friendly introduction to RSA encryption and the importance of securely generating key pairs.

Feel free to explore the provided scripts and experiment with different RSA concepts to deepen your understanding of cryptography. Happy hacking, and may the challenges keep coming!
