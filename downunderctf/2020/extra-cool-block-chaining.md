# Extra Cool Block Chaining

**CTF:** DownUnderCTF 2020  
**Category:** crypto  
**Original writeup:** <https://jsur.in/posts/2020-09-20-downunderctf-2020-writeups> — by **jsur.in**. All credit goes to the original author; this is an archived copy.

---

Extra Cool Block Chaining (422pts) - 52 solves

Everyone knows ECB is broken because it lacks diffusion. That's why I've come up with my own variant that uses IVs and chaining and all that cool stuff! It solves all the problems ECB had... I think nc chal.duc.tf 30201

server.py :

from Crypto . Cipher import AES from Crypto . Util . Padding import pad , unpad from Crypto . Util . strxor import strxor from os import urandom flag = open ( './flag.txt' , 'rb' ) . read ( ) . strip ( ) KEY = urandom ( 16 ) IV = urandom ( 16 ) def encrypt ( msg , key , iv ) : msg = pad ( msg , 16 ) blocks = [ msg [ i : i + 16 ] for i in range ( 0 , len ( msg ) , 16 ) ] out = b'' for i , block in enumerate ( blocks ) : cipher = AES . new ( key , AES . MODE_ECB ) enc = cipher . encrypt ( block ) if i > 0 : enc = strxor ( enc , out [ - 16 : ] ) out += enc return strxor ( out , iv * ( i + 1 ) ) def decrypt ( ct , key , iv ) : blocks = [ ct [ i : i + 16 ] for i in range ( 0 , len ( ct ) , 16 ) ] out = b'' for i , block in enumerate ( blocks ) : dec = strxor ( block , iv ) if i > 0 : dec = strxor ( dec , ct [ ( i - 1 ) * 16 : i * 16 ] ) cipher = AES . new ( key , AES . MODE_ECB ) dec = cipher . decrypt ( dec ) out += dec return out flag_enc = encrypt ( flag , KEY , IV ) . hex ( ) print ( 'Welcome! You get 1 block of encryption and 1 block of decryption.' ) print ( 'Here is the ciphertext for some message you might like to read:' , flag_enc ) try : pt = bytes . fromhex ( input ( 'Enter plaintext to encrypt (hex): ' ) ) pt = pt [ : 16 ] enc = encrypt ( pt , KEY , IV ) print ( enc . hex ( ) ) except : print ( 'Invalid plaintext! :(' ) exit ( ) try : ct = bytes . fromhex ( input ( 'Enter ciphertext to decrypt (hex): ' ) ) ct = ct [ : 16 ] dec = decrypt ( ct , KEY , IV ) print ( dec . hex ( ) ) except : print ( 'Invalid ciphertext! :(' ) exit ( ) print ( 'Goodbye! :)' )

Solution

The server implements some weird mode of operation that looks something like this:

encryption

decryption

Note that the decryption function is actually broken, I made an implementation mistake (forgot an XOR, oops) and didn't realise until someone pointed it out during the CTF. This doesn't affect the solution or the challenge at all though, since you are only given the first block of ciphertext, and even with the implementation error, the first block of ciphertext decrypts properly. This is what it should have looked like:

We can easily recover the first 16 bytes of the flag by asking for the decryption of the first 16 bytes of the ciphertext we receive. We can't do the same for the rest of the blocks though.

We quickly notice that if we can recover the IV, we can decrypt any block we want by asking for the decryption of C_{i-1} xor IV xor C_i .

We turn to the encryption part of the service to see how we can recover the IV. Notice how after a plaintext block passes through the block cipher, it is XORed with the encryption of the previous block (before the IV is applied). So, if we could somehow get the encryption of two identical blocks, the second block would be 0 after the "chaining" part (XORing with the previous encrypted plaintext block), and so after XORing with the IV, the result would be just the IV.

But we're only allowed to send one block of plaintext to be encrypted! Luckily, the server pads our message before encrypting it, as it is necessary for the message to be a multiple of the block length. PCKS#7 padding is being used, so if we send a full block of 16s ( \x10\x10\x10\x10\x10\x10\x10\x10\x10\x10\x10\x10\x10\x10\x10\x10 ), it will be padded out to two full blocks of 16s. This gives us our two identical plaintext blocks! It follows that the IV will be given to us in the 2nd ciphertext block.

We can then recover the rest of the flag as described above.

Solve script:

from pwn import remote , xor from Crypto . Util . Padding import unpad def connect ( ) : return remote ( '0.0.0.0' , 1337 ) def recv ( conn ) : o = conn . recvline ( ) . decode ( ) print ( '[<]' , o ) return o def send ( conn , data ) : print ( '[>]' , data ) conn . sendline ( data ) flag = b'' conn = connect ( ) recv ( conn ) ciphertext = recv ( conn ) . split ( 'read: ' ) [ 1 ] num_blocks = len ( ciphertext ) // 32 send ( conn , 'aa' * 16 ) recv ( conn ) send ( conn , ciphertext [ : 32 ] ) flag += bytes . fromhex ( recv ( conn ) . split ( ': ' ) [ 1 ] ) conn . close ( ) for i in range ( 1 , num_blocks ) : conn = connect ( ) recv ( conn ) ciphertext = bytes . fromhex ( recv ( conn ) . split ( 'read: ' ) [ 1 ] ) send ( conn , ( b'\x10' * 16 ) . hex ( ) ) IV = bytes . fromhex ( recv ( conn ) . strip ( ) [ - 32 : ] ) to_decrypt = xor ( xor ( IV , ciphertext [ ( i - 1 ) * 16 : i * 16 ] ) , ciphertext [ i * 16 : ( i + 1 ) * 16 ] ) send ( conn , to_decrypt . hex ( ) ) flag += bytes . fromhex ( recv ( conn ) . split ( ': ' ) [ 1 ] ) conn . close ( ) print ( '[*]' , unpad ( flag , 16 ) . decode ( ) )

Flag: DUCTF{4dD1nG_r4nd0M_4rR0ws_4ND_x0RS_h3r3_4nD_th3R3_U5u4Lly_H3lps_Bu7_n0T_7H1s_t1m3_i7_s33ms!!}
