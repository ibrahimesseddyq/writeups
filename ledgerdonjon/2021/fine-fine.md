# fine--fine-

**CTF:** LedgerDonjon 2021  
**Category:** crypto  
**Original writeup:** <https://eshard.com/posts/ledger-fine-fine> — by **eshard.com**. All credit goes to the original author; this is an archived copy.

---

This article presents the solution of the Fine Fine side-channel challenge of the 2021 Ledger Donjon CTF.

The statement of the challenge is the following:

We hooked an oscilloscope to a chip we found on the floor near the entrance of the Crypto-Party. It executes what looks like the public key computation on secp256r1 from some static secret, but it seems we can specify the input point though... And it's definitely a Montgomery Ladder. That secret in there must give us access to free cryptodrinks, surely?

Introduction

To be fair, I did not find the right attack path so quickly. I tried a lot of techniques like cross correlation, or trying to find some bit values as I know that the scalar contains "CTF{". But without success. After a discussion with my colleague Pierre-Yvan Liardet about the possible attack paths against:

a scalar multiplication

using Montgomery ladder

with point randomization

knowing that I can choose the point to multiply

he pointed me directly to an attack from Louis Goubin [Goubin2003] so called the "Refined attack", a chosen point attack. Mais oui mais c'est bien sûr! Fine fine -> Refined -> correct attack!

I'm sure that I'm on the right track! Let's implement it.

Attack theory

The refined attack is a chosen point method to attack scalar multiplication. It is compatible with Montgomery ladder, projection and point randomization.

The main idea is to obtain, at some step of the calculus a point with a null coordinate. If the implementation is not sufficiently robust, the operations involving should have a different signal. Indeed, regardless of the formulas used to perform the addition or the doubling, they will involve a lot of calculus with zero values, that can lead to differences in the power consumption compared to non-zero values.

First, a quick reminder about the Montgomery ladder algorithm. In the following, the registers and usually found in the literature are replaced by and to avoid double indexing.

Let the binary form of the scalar . The Montgomery ladder algorithm to perform the multiplication of the point by is the following:

A ← 0 B ← P for i from m downto 0 do if d_i = 0 then B ← point_add(A, B) A ← point_double(A) else A ← point_add(A, B) B ← point_double(B) return A

If we are looking at the operand of the double and add operations, we can see that:

the add operation always performs , regardless of the value of depending on the value of the bit , the double operation is performed either on if is 0 or on if is 1.

And knowing the bits we can build a special point such as or equal when is processed. And then the second observation can be used to distinguish bit values.

Compute (precursor of) special point

We want to find a point with a null coordinate.

The challenge uses the secp256r1 elliptic curve (we will use the tinyec package to work with this curve). As the curve equation is , a point is on the curve and should do the job. Then let's compute (how to calculate modular square roots).

First, we need to check that is a quadratic residue modulo the prime :

from tinyec . registry import get_curve curve = get_curve ( 'secp256r1' ) p = curve . field . p pow ( curve . b , ( p - 1 ) // 2 , p )

1

Yes it is!

The method to compute the square root depends on the modulo:

print ( 'p congruent to 3 mod 4: ' , p % 4 == 3 ) print ( 'p congruent to 5 mod 8: ' , p % 8 == 5 ) print ( 'p congruent to 1 mod 8: ' , p % 8 == 1 )

p congruent to 3 mod 4: True p congruent to 5 mod 8: False p congruent to 1 mod 8: False

We are in the easy case ! The square roots of can be computed as

root = pow ( curve . b , ( p + 1 ) // 4 , p ) assert pow ( root , 2 , p ) == curve . b root

46263761741508638697010950048709651021688891777877937875096931459006746039284

Let's now define the point :

from tinyec import ec p_star = ec . Point ( curve , 0 , root ) p_star

(0, 46263761741508638697010950048709651021688891777877937875096931459006746039284) on "secp256r1" => y^2 = x^3 + 115792089210356248762697446949407573530086143415290314195533631308867097853948x + 41058363725152142129326129780047268409114441015993725554835256314039467401291 (mod 115792089210356248762697446949407573530086143415290314195533631308867097853951)

But it is not sufficient to perform the attack. We want a "precursor" of , i.e. a point such as for a given scalar , . This point is easy to find: , where is the group order.

We can define a function to compute for a given scalar :

def generate_p_prime ( scalar ) : """Generate a point P' such as known_scalar x P = P*""" curve = get_curve ( 'secp256r1' ) p = curve . field . p root = pow ( curve . b , ( p + 1 ) // 4 , p ) p_star = ec . Point ( curve , 0 , root ) inv = ec . mod_inv ( scalar , curve . field . n ) p_prime = inv * p_star assert ( scalar * p_prime ) . x == 0 return p_prime # Check that we get p_start (having x==0) with k=3 p_prime = generate_p_prime ( 3 ) 3 * p_prime

(0, 46263761741508638697010950048709651021688891777877937875096931459006746039284) on "secp256r1" => y^2 = x^3 + 115792089210356248762697446949407573530086143415290314195533631308867097853948x + 41058363725152142129326129780047268409114441015993725554835256314039467401291 (mod 115792089210356248762697446949407573530086143415290314195533631308867097853951)

Observe the traces

It is now time to observe the traces. The challenge comes with a client.py file which provides a function send_getpubkey that asks the chip to perform the scalar multiplication for a given point and then return the recorded power trace. As a reminder, the goal is to find the secret scalar k used by the chip to perform point multiplication.

Let's try it with the curve base point (twice):

point2str = lambda point : point . x . to_bytes ( 32 , byteorder = 'big' ) . hex ( ) + point . y . to_bytes ( 32 , byteorder = 'big' ) . hex ( )

from client import send_getpubkey base_trace = send_getpubkey ( point2str ( curve . g ) ) base_trace_2 = send_getpubkey ( point2str ( curve . g ) )

import matplotlib . pyplot as plt plt . style . use ( './matplotlib.style' ) plt . plot ( base_trace ) plt . plot ( base_trace_2 ) plt . xlim ( 10_000 , 11_000 ) plt . title ( 'Zoom on returned traces for scalar mult on G' ) plt . show ( )

The trace is composed of 11,506 patterns separated by a sequence of zero values. And we can see that the two traces are different even if we requested the same point computation. This indicates the presence of the point randomization countermeasure, which is almost free when projective coordinates are used.

We can now observe what is happening when the point (resp ), i.e. the point such as (resp is processed.

import numpy as np traces = [ ] for scalar in range ( 2 , 6 ) : traces . append ( send_getpubkey ( point2str ( generate_p_prime ( scalar ) ) ) ) traces = np . vstack ( traces )

As shown with red arrows in the following figure, we can observe some new zero values obtained with for different values.

plt . plot ( traces . T [ : 2000 ] ) plt . xlim ( 1050 , 1550 ) plt . legend ( labels = [ f"$P'_ { i } $" for i in range ( 2 , 6 ) ] ) plt . title ( "Zoom on traces for $P'_2$ to $P'_5$" ) plt . arrow ( 1100 , 30000 , - 18 , - 25000 , color = 'red' , width = 0.5 , head_width = 6 , head_length = 4000 ) plt . arrow ( 1350 , 30000 , 18 , - 25000 , color = 'red' , width = 0.5 , head_width = 6 , head_length = 4000 ) plt . arrow ( 1510 , 30000 , 18 , - 25000 , color = 'red' , width = 0.5 , head_width = 6 , head_length = 4000 ) plt . show ( )

Attack the traces

These zero values can be used to decide if the bit is 0 or 1 with the following methodology:

Let the known first bits of the scalar . During the -th iteration of the scalar multiplication of , we have and . And then, as explained previously, the double operation is performed on either or depending on the current bit value.

To make appear at iteration , in either or , we can generate the traces for both and . By observing which one has the higher number of zero values, we will distinguish if the double operation is performed on or and then obtain the current bit value. To improve the process efficiency, we can use averaged traces.

The following function takes a known scalar as input and then returns the current bit processed is or 1:

def find_bit ( known_scalar ) : n = 2 mean_a = np . mean ( [ send_getpubkey ( point2str ( generate_p_prime ( known_scalar ) ) ) for _ in range ( n ) ] , 0 ) mean_b = np . mean ( [ send_getpubkey ( point2str ( generate_p_prime ( known_scalar + 1 ) ) ) for _ in range ( n ) ] , 0 ) return int ( ( mean_a == 0 ) . sum ( ) < ( mean_b == 0 ) . sum ( ) )

We can now automate the process to retrieve each of the 256 bits. To start, we can consider that the first bit is 1 and the process bit by bit:

from tqdm . notebook import tqdm d = '1' for _ in tqdm ( range ( 255 ) ) : di = find_bit ( int ( d , 2 ) ) d += str ( di ) d

0%| | 0/255 [00:00<?, ?it/s] '1011100010010111010101000100101110001100011101011100001110010101101100110100001101010100010001100111101101000111011010000110111101110011011101000111001101011111011000010110111001100100010111110100011101101111011101010110001001101001011011100111001101111101'

int ( d , 2 ) . to_bytes ( length = 32 , byteorder = 'big' )

b'\xb8\x97TK\x8cu\xc3\x95\xb3CTF{Ghosts_and_Goubins}'

It works! The flag is CTF{Ghosts_and_Goubins}.
