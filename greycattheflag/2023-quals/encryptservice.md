# EncryptService

**CTF:** GreyCatTheFlag 2023 Quals  
**Category:** crypto  
**Original writeup:** <https://hong5489.github.io/2023-05-22-greyctf2023/> — by **hong5489**. All credit goes to the original author; this is an archived copy.

---

EncryptService

Description

Attachment

After analyzed the source code, we can see it is using AES CTR mode to encrypt the flag:

def encrypt ( plaintext , iv ): hsh = sha256 ( iv ). digest ()[: 8 ] cipher = AES . new ( secret_key , AES . MODE_CTR , nonce = hsh ) ciphertext = cipher . encrypt ( plaintext ) return ciphertext . hex () ... ... print ( "Flag: " , encrypt ( FLAG . encode ( "ascii" ), os . urandom ( 1 )))

Also we are given ciphertext with 256 different nonce:

for i in range ( 256 ): ciphertext = encrypt ( plaintext , i . to_bytes ( 1 , 'big' )) print ( f "Ciphertext { i } : { ciphertext } " )

Analyse

Look at the wikipedia page of AES CTR mode, we can clearly see the encrypt and decrypt process:

Notice the encrypt and decrypt process are the same! It just XOR the plaintext/ciphertext with the AES encryption

And we are given ciphertext for all possible nonce, so we can just XOR all 256 ciphertext with the encrypted flag and see which return the flag format!

Solution

My solution in python script:

from pwn import * ciphertext = [] p = remote ( "34.124.157.94" , 10590 ) # Send 40 null bytes (flag is 40 characters long) p . sendlineafter ( ": " , "00" * 40 ) for i in range ( 256 ): p . recvuntil ( ": " ) ciphertext . append ( p . recvuntil ( '

' )[: - 1 ]) p . recvuntil ( "Flag: " ) flag = p . recvuntil ( '

' )[: - 1 ] # XOR all ciphertext with the encrypted flag for c in ciphertext : f = xor ( bytes . fromhex ( c . decode ()), bytes . fromhex ( flag . decode ())) if f . startswith ( b "grey{" ): print ( f ) # b'grey{0h_m4n_57r34m_c1ph3r_n07_50_53cur3}'

Flag

grey{0h_m4n_57r34m_c1ph3r_n07_50_53cur3}
