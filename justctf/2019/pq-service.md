# p&q Service

**CTF:** justCTF 2019  
**Category:** crypto  
**Original writeup:** <https://github.com/hyperreality/ctf-writeups/tree/master/2019-justctf> — by **hyperreality**. All credit goes to the original author; this is an archived copy.

---

### p&q service

```

Let's play the game. You give me p and q along with the calculated cipher and I will try to guess the e and decipher the text. Just encrypt the justGiveTheFlag!! word and I will give you the flag! Hopefully...

```

This was a cool RSA-like challenge that asked us to exploit an oddity of multiplicative modular groups.

We input primes p and q, and a ciphertext to the server. It validates the primes are strong, calculates random but RSA-conforming e and d, decrypts our ciphertext, and checks that our plaintext exactly matches 'justGiveTheFlag!!' after removing padding.

At first it seems impossible to produce a specific ciphertext when we don't know the public or private exponent, however there are a few things here working in our favour:

1. The padding function just looks at the last byte of the plaintext and removes the corresponding number of bytes from the message.

1. We are in total control of the modulus, and can take advantage of the weirdness of modular math :D

Our first idea was to use the fact that `(n-1) ^ e % n` cycles between just two values, `n-1` when `e` is odd, and `1` when `e` is even. RSA requires odd exponents anyway, so after applying the exponent, `n-1` is unchanged modulo n.

We then search for two primes which when multiplied give us an n of the form `'justGiveTheFlag!!' + JUNK + NUM_BYTES_TO_UNPAD+1`. We subtract one from this, submit that as the ciphertext along with the p and q that we found, and the server will strip off the junk bytes leaving us with just the message as plaintext. Great.

However, this approach runs into a small but insurmountable issue. Due to the primes needing to be ~256 bits, we have to use exactly 47 bytes of junk and a final byte of 0x30 (48) to get a modulus and ciphertext which not only unpad the correct amount but are also within the allowed bitsize of 508-512. However, 0x30 is even and it's thus impossible for two large primes to multiply to a number that ends with that byte. If we set it to 0x29, then our plaintext would be unpadded one byte too few, and decrypt to 'justGiveTheFlag!!\\x01' - close, but no cigar.

The real solution which we didn't discover until later, is to use a slightly more sophisticated nuance of modular math, which is that `p^e % (p*(p-1)) = p`

To gain an intution for how this works, here's an example with `p = 19`. As the exponent on p increases, the difference between `p^e` and the modulus remains a multiple of 19, so the exponent once again doesn't actually matter.

```

e p^e p*(p-1) p^e - p*(p-1) p^e % (p*k(p-1))

1 19 342 -323 = -17*19 19

2 361 342 19 = 1*19 19

3 6859 342 6517 = 343*19 19

4 130321 342 129979 = 6481*19 19

```

It's easiest to see this with the 19^2 case. Clearly 19\*19 mod 19\*18 is 19, and by further incrementing the exponent, we are just successively multiplying the LHS by 19 which doesn't change the output mod 19.

So `p*(p-1)` as modulus seems to work out nicely, however in this challenge we need q to be a prime, so we can't simply use `p-1` for q as it would be a even number.

If we use a factor of `p-1`, such as `(p-1)/2` instead of `p-1`, then our modulus becomes `p*(p-1)/2`. In the case of p = 19, this equals 171, and 19\*19 - 171 is 190 rather than 19. Here we are off by 10 multiples of 19 instead of 1, which is of course exactly the same modulo 19\*9. So we can select q as a factor of `p-1`, say `(p-1)/2`, and keep trying values until it is prime also. There's plenty such primes and they are very useful elsewhere in cryptography, see [safe primes](https://en.wikipedia.org/wiki/Safe_prime).

So putting it all together, we:

- construct a message that is `'justGiveTheFlag!!' + JUNK + NUM_BYTES_TO_UNPAD`

- bruteforce the junk bytes until we find a prime p that in byte form matches the beginning and end of the message

- ensure that (p-1)/2 is also prime so we can use it as q.

- submit p and q, and p as the ciphertext, to the server

The decryption happens, the ciphertext remains identical due to the laws above, and then the unpad function helpfully removes the junk for us, leaving 'justGiveTheFlag!!'
