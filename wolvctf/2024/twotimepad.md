# TwoTimePad

**CTF:** WolvCTF 2024  
**Category:** crypto  
**Original writeup:** <https://nopedawn.github.io/posts/ctfs/2024/wolv-ctf-2024/#crypto-twotimepad> — by **nopedawn**. All credit goes to the original author; this is an archived copy.

---

Overview Author: cogsworth64

Description: One-time pads are perfectly information-theoretically secure, so I should be safe, right?

chall.py

eFlag.bmp

eWolverine.bmp

Second challenge crypto beginner is a classic example of the misuse a one-time pad (OTP). Called TwoTimePad is a hint towards this. The key should be random, secret, and never reused. However in this challenge, the same key is used to encrypt two different encrypted files ( eWolverine.bmp & eFlag.bmp ).

chall.py 1 from Crypto.Random import random , get_random_bytes 2 3 BLOCK_SIZE = 16 4 5 with ( open ( './genFiles/wolverine.bmp' , 'rb' )) as f : 6 wolverine = f . read () 7 with ( open ( './genFiles/flag.bmp' , 'rb' )) as f : 8 flag = f . read () 9 10 w = open ( 'eWolverine.bmp' , 'wb' ) 11 f = open ( 'eFlag.bmp' , 'wb' ) 12 13 f . write ( flag [: 55 ]) 14 w . write ( wolverine [: 55 ]) 15 16 for i in range ( 55 , len ( wolverine ), BLOCK_SIZE ): 17 KEY = get_random_bytes ( BLOCK_SIZE ) 18 w . write ( bytes ( a ^ b for a , b in zip ( wolverine [ i : i + BLOCK_SIZE ], KEY ))) 19 f . write ( bytes ( a ^ b for a , b in zip ( flag [ i : i + BLOCK_SIZE ], KEY )))

The headers of the BMP files contain important information such as the size, color depth, compression method, etc. In the provided Python script, the headers of the original images are preserved. However, if we XOR the two encrypted images together, the headers are also XOR-ed.

Then, we need to XOR the two encrypted images together. Since the same key was used for both, this will effectively cancel out the key, leaving with the XOR of the two original images.

solver.py 1 HEADER_SIZE = 55 # Size of the BMP header 2 3 with open ( 'eWolverine.bmp' , 'rb' ) as f : 4 eWolverine = f . read () 5 with open ( 'eFlag.bmp' , 'rb' ) as f : 6 eFlag = f . read () 7 8 header = eWolverine [: HEADER_SIZE ] 9 10 xor = header + bytes ( a ^ b for a , b in zip ( eWolverine [ HEADER_SIZE :], eFlag [ HEADER_SIZE :])) 11 12 with open ( 'out.bmp' , 'wb' ) as f : 13 f . write ( xor )

FLAG wctf{D0NT_R3CYCLE_K3Y5}
