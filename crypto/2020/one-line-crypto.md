# One Line Crypto

**CTF:** Crypto 2020  
**Category:** crypto  
**Original writeup:** <https://blog.cryptohack.org/cryptoctf2020> — by **blog.cryptohack.org**. All credit goes to the original author; this is an archived copy.

---

One Line Crypto

Challenge

A profile, a look, a voice, can capture a heart ♥ in no time at all.”

We’re given this very short snippet:

#!/usr/bin/python from Crypto.Util.number import * from secret import m , n , x , y , flag p , q = x ** ( m + 1 ) - ( x + 1 ) ** m , y ** ( n + 1 ) - ( y + 1 ) ** n assert isPrime ( p ) and isPrime ( q ) and p < q < p << 3 and len ( bin ( p * q )[ 2 :]) == 2048 enc = bytes_to_long ( flag ) print ( pow ( enc , 0x10001 , p * q ))

and the output of the print function, which consists of just a single, giant number. This is typical RSA encryption, with a slight twist: n is not given!

Solution

The prime selection for p and q is identical. Two secret numbers x and m are used, such that x^(m+1) - (x+1)^m is a prime. Calculating q uses differently named constants, but there’s no cross-dependencies between the parameters.

Knowing that p*q contains 2048 bits, counting from the MSB, puts some rather strict bounds on the parameters. Just playing around with random numbers, it quickly becomes clear that m / n can’t be very big.

The plan simply becomes:

Start off with a low bound like 500 or 1000 Brute-force all m and x less than the bound, such that x^(m+1) - (x+1)^m is max 2048 bits. When we got ourselves a small pool of candidate values, pair up two and two random values from the pool Check if their product is 2048 bits, and try to decrypt the ciphertext.

Implementation

#!/usr/bin/python3 from Crypto.Util.number import long_to_bytes from gmpy2 import invert , is_prime from tqdm import tqdm primes = [] for xy in tqdm ( range ( 500 )): for mn in range ( 500 ): prime = xy ** ( mn + 1 ) - ( xy + 1 ) ** mn if prime . bit_length () > 2048 : break if is_prime ( prime ): primes . append ( prime ) c = 14608474132952352328897080717325464308438322623319847428447933943202421270837793998477083014291941466731019653023483491235062655934244065705032549531016125948268383108879698723118735440224501070612559381488973867339949208410120554358243554988690125725017934324313420395669218392736333195595568629468510362825066512708008360268113724800748727389663826686526781051838485024304995256341660882888351454147057956887890382690983135114799585596506505555357140161761871724188274546128208872045878153092716215744912986603891814964771125466939491888724521626291403272010814738087901173244711311698792435222513388474103420001421 for i in range ( len ( primes )): for j in range ( i , len ( primes )): pq = primes [ i ] * primes [ j ] if len ( bin ( pq )[ 2 :]) == 2048 : try : d = invert ( 0x10001 , ( primes [ i ] - 1 ) * ( primes [ j ] - 1 )) dec = long_to_bytes ( pow ( c , d , pq )) if b "CCTF" in dec : print ( dec ) except ValueError : pass

Flag

CCTF{0N3_1!nE_CrYp7O_iN_202O}
