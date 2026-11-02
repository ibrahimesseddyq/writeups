# story

**CTF:** GoogleCTF 2021  
**Category:** crypto  
**Original writeup:** <https://jsur.in/posts/2021-07-19-google-ctf-2021-crypto-writeups> — by **jsur.in**. All credit goes to the original author; this is an archived copy.

---

Story

Please, tell me a beautiful story. attachment story.2021.ctfcompetition.com 1337

The server asks us for a string of printable characters and computes some unknown CRC on it. It gives us the value of the CRC and then "randomises" our input in some way. The CRC and randomise functions are not given. It then gives us the randomised values and challenges us to provide a second input that consists of the same characters as the first input, only differing by capitalisation, which has a CRC value that is the same as the randomised values. We suspect that the randomise function simply flips the case (lowercase/uppercase) of characters in its input and computes the CRC of that.

The CRC function we are interested in produces CRCs of 112 112 112 bits. When we tested against the server, we were able to pass the expectedValues != null check by providing strings of length 256 . So our approach will be to consider inputs of length 256 that consist of a single letter with different capitalisations in differnt positions. We will represent these as vectors of size 256 over F 2 \mathbb{F}_2 F2​, where a lowercase character is represented by 0 0 0 and an uppercase character is represented by 1 1 1. We also represent the CRC values as vectors of size 112 over F 2 \mathbb{F}_2 F2​, which we convert by simply looking at the binary string of the CRC value.

A known property of CRCs is that they are affine; a C R C \mathrm{CRC} CRC that takes an n n n-bit input and produces an m m m-bit output can be represented by a matrix M M M of size m × n m \times n m×n and a vector c 0 c_0 c0​ where c 0 = C R C ( 0 ) c_0 = \mathrm{CRC}(0) c0​=CRC(0). Specifically, if X X X is an n n n-bit input, then C R C ( X ) = M X + c 0 \mathrm{CRC}(X) = MX + c_0 CRC(X)=MX+c0​. We can write M M M in terms of where the CRC maps the basis vectors of F 2 n \mathbb{F}_2^n F2n​ (viewed as a vector space). So

M = ( ∣ ∣ ∣ c 1 c 2 ⋯ c n ∣ ∣ ∣ ) M = \begin{pmatrix} | & | & & | \\ c_1 & c_2 & \cdots & c_n \\ | & | & & | \end{pmatrix} M = ⎝ ⎛ ​ ∣ c 1 ​ ∣ ​ ∣ c 2 ​ ∣ ​ ⋯ ​ ∣ c n ​ ∣ ​ ⎠ ⎞ ​

where c i = C R C ( [ 0 , … , 0 , 1 , 0 , … , 0 ] ) c_i = \mathrm{CRC}([0, \ldots, 0, 1, 0, \ldots, 0]) ci​=CRC([0,…,0,1,0,…,0]) (the 1 1 1 is in the i i ith position).

We can find the c i c_i ci​ by asking the server for the CRC value of a 256 length string of a s where the i i ith character is capitalised.

Once we have M M M and c 0 c_0 c0​, finding preimages, X X X, for a given target CRC, T T T, is easy:

T = M X + c 0 ⟹ M X = T + c 0 \begin{aligned} T &= MX + c_0 \\ \implies MX &= T + c_0 \end{aligned} T ⟹ MX ​ = MX + c 0 ​ = T + c 0 ​ ​

We simply need to solve the system of linear equations and any solution will pass the check.

import os os . environ [ 'PWNLIB_NOTERM' ] = 'True' from pwn import * from tqdm import tqdm def get_corpus ( N ) : corpus = { } S = b'a' * 256 for i in tqdm ( range ( N ) ) : conn = remote ( 'story.2021.ctfcompetition.com' , 1337 , level = 'error' ) conn . recvline ( ) conn . recvline ( ) conn . sendline ( S ) o = conn . recvline ( ) . decode ( ) crc16 , crc32 , crc64 = [ int ( x , 16 ) for x in o . split ( '[' ) [ 1 ] . strip ( '].

' ) . split ( ', ' ) ] corpus [ S ] = ( crc16 , crc32 , crc64 ) conn . close ( ) S = b'a' * i + b'A' + b'a' * ( 255 - i ) return corpus corpus = get_corpus ( 257 ) def bytes_to_Fvec ( b ) : return vector ( F , [ bool ( c == 97 ) for c in b ] ) def num_to_Fvec ( n , nbits ) : return vector ( F , map ( int , bin ( n ) [ 2 : ] . zfill ( nbits ) ) ) def crc_to_vec ( crc ) : return num_to_Fvec ( crc [ 0 ] | crc [ 1 ] << 16 | crc [ 2 ] << 48 , 112 ) F = GF ( 2 ) c0 = crc_to_vec ( corpus [ b'a' * 256 ] ) eqs = [ crc_to_vec ( corpus [ s ] ) for s in list ( corpus ) [ 1 : ] ] M = Matrix ( F , eqs ) . transpose ( ) conn = remote ( 'story.2021.ctfcompetition.com' , 1337 ) conn . recvline ( ) conn . recvline ( ) conn . sendline ( 'a' * 256 ) conn . recvline ( ) o = conn . recvline ( ) . decode ( ) target_crcs = [ int ( x , 16 ) for x in o . split ( '[' ) [ 1 ] . strip ( '].

' ) . split ( ', ' ) ] target = crc_to_vec ( target_crcs ) sol = M . solve_right ( target + c0 ) sol = '' . join ( [ 'aA' [ i ] for i in sol ] ) conn . sendline ( sol ) print ( conn . recvline ( ) . decode ( ) )

Flag: CTF{eb64749d08bd99b681f2bc75aa65eab35a80310f7426f6872ba869d244e37135}
