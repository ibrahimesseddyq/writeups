# Baby Crypt

**CTF:** CSAW 2017 Quals  
**Category:** crypto  
**Original writeup:** <https://teamrocketist.github.io/2017/09/18/Crypto-CSAW-CTF-2017-Baby-Crypt/> — by **teamrocketist**. All credit goes to the original author; this is an archived copy.

---

baby_crypt

The cookie is input + flag AES ECB encrypted with the sha256 of the flag as the key.



nc crypto.chal.csaw.io 1578



The first step that we took was to find out how many AES 16 byte blocks that the flag has.

To measure this we simply sent an empty request to the service.

1

2

3

DiogoMonteiro @ ~/baby_crypt -> nc crypto.chal.csaw.io 1578

Enter your username (no whitespace):

Your Cookie is: f9cc1330ae5830732a18d1a23211ffbce3725519adb9e6f10d658d87c80825ed



At this stage, and having a hex string with 64 chars, we know that the flag has 32 bytes (including any padding).

Therefore, the flag represents two AES 16 byte blocks.

From the challenge description we know that the encryption is performed using AES ECB, which is vulnerable to chosen plaintext attacks. As a matter of fact, we have control over the first (any) bytes of the plaintext.

With a chosen plaintext attack on AES, to find the 32 bytes of the flag, we need to brute force each one of its bytes, resulting in a search space with size 32 x 94 = 3008 chars, which is easy.

The chosen plaintext attack that we performed works as follows. The idea is to input 32 bytes where the last byte is the one that we are going to brute force.

We start by sending to the service the input = AAAAAAAAAAAAAAAX (16 chars) for each char X in the range of printable chars, and store the resulting ciphertext of the second block in a dictionary D1.

1

2

3

4

5

6

7

8

9

10

(X = byte to brute force, ! = byte of the secret, P = padding byte)



| AAAAAAAAAAAAAAAA | AAAAAAAAAAAAAAAX | !!!!!!!!!!!!!!!! | !!!!!!!!!!!!!!!! | for each X in 0x20 to 0x7E

| ---------------input ---------------| --------------secret ---------------|



D1 = {

'cipher(AAAAAAAAAAAAAAAa)': 'a',

'cipher(AAAAAAAAAAAAAAAb)': 'b',

...

}



To find the first character of the flag, we compute the ciphertext of the second block with input = AAAAAAAAAAAAAAA (15 chars) and lookup for that ciphertext in the dictionary D1.

The next round is similar.

For each byte X in the range of printable chars we send input = AAAAAAAAAAAAAAfX (16 chars, and notice the previously found “f” char) to the server and store the resulting ciphertext of the second block in a dictionary D2.

1

2

3

4

5

6

7

8

| AAAAAAAAAAAAAAAA | AAAAAAAAAAAAAAfX | !!!!!!!!!!!!!!!! | !!!!!!!!!!!!!!!P |

| ---------------input ---------------|



D2 = {

'cipher(AAAAAAAAAAAAAAfa)': 'a',

'cipher(AAAAAAAAAAAAAAfb)': 'b',

...

}



Giving input = AAAAAAAAAAAAAAf to the server and looking up in the dictionary D2, we find the second letter of the flag.

And so on.

The following code implements the attack.

1

2

3

4

5

6

7

8

9

10

11

12

13

14

15

16

17

18

19

20

21

22

23

24

25

26

27

28

from pwn import *



conn = remote( "crypto.chal.csaw.io" , 1578 )



def oracle (chosen) :

conn.send(chosen + "

" )

return conn.recvline().split( " " )[ -1 ].rstrip( "

" )



def second_block (blocks) :

return blocks[ 32 : 64 ]





found = ""

for j in range( 32 ):

d = {}

b = ( "A" * ( 31 -j)) + found



for i in range( 0x20 , 0x7E ):

chosen = b + chr(i)

blocks = oracle(chosen)

d[second_block(blocks)] = chr(i)

print "." ,



n = oracle( "A" * ( 31 -j))

found += d[second_block(n)]

print found



print found



After running it, we get the flag:

1

2

3

4

5

6

7

8

9

10

11

12

13

14

15

16

17

18

19

20

21

22

23

24

25

26

27

28

29

30

31

32

33

34

35

36

37

38

..............................................................................................

f

..............................................................................................

fl

..............................................................................................

fla

..............................................................................................

flag

..............................................................................................

flag{

..............................................................................................

flag{C

..............................................................................................

flag{Cr

..............................................................................................

flag{Cry

..............................................................................................

flag{Cryp

..............................................................................................



(truncated output)



flag{Crypt0_is_s0_h@rd_t0

..............................................................................................

flag{Crypt0_is_s0_h@rd_t0_

..............................................................................................

flag{Crypt0_is_s0_h@rd_t0_d

..............................................................................................

flag{Crypt0_is_s0_h@rd_t0_d0

..............................................................................................

flag{Crypt0_is_s0_h@rd_t0_d0.

..............................................................................................

flag{Crypt0_is_s0_h@rd_t0_d0..

..............................................................................................

flag{Crypt0_is_s0_h@rd_t0_d0...

..............................................................................................

flag{Crypt0_is_s0_h@rd_t0_d0...}

flag{Crypt0_is_s0_h@rd_t0_d0...}



Very fun challenge!
