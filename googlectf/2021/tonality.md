# tonality

**CTF:** GoogleCTF 2021  
**Category:** crypto  
**Original writeup:** <https://jsur.in/posts/2021-07-19-google-ctf-2021-crypto-writeups> — by **jsur.in**. All credit goes to the original author; this is an archived copy.

---

Tonality

In this challenge, you have access to a signing oracle. Good luck! attachment tonality.2021.ctfcompetition.com 1337

We have a similar setup to Tiramisu; we are given the Go code running on the server and a client to interact with it. The goal of this task is to forge an ECDSA signature given the ability to "scale" the server's private key for one signing. Specifically, when a client connects to the server, the server generates a private key and sends the client its public key. The server asks the client for a scalar and then signs a message, scaling its private key by the client-provided scalar. It then challenges the client to forge a signature for another message.

The server uses secp256r1 so there are no insecure curve attacks. Let d d d be the server's private key and let h 1 , h 2 h_1, h_2 h1​,h2​ be the two messages. The server will sign h 1 h_1 h1​, and our goal is to sign h 2 h_2 h2​. Let t t t be the scalar we provide to the server to scale its private key when it signs the the first message. The server generates a random nonce k k k and computes r 1 = ( k G ) x r_1 = (kG)_x r1​=(kG)x​ and s 1 = k − 1 ( h 1 + t r 1 d ) s_1 = k^{-1}(h_1 + t r_1 d) s1​=k−1(h1​+tr1​d). We can control t t t, so set t = h 1 h 2 t = \frac{h_1}{h_2} t=h2​h1​​. Then,

s 1 = k − 1 ( h 1 + t r 1 d ) = t k − 1 ( h 1 t + r 1 d ) = h 1 k − 1 h 2 ( h 2 + r 1 d ) \begin{aligned} s_1 &= k^{-1}(h_1 + t r_1 d) \\ &= t k^{-1}\left (\frac{h_1}{t} + r_1 d \right ) \\ &= \frac{h_1 k^{-1}}{h_2} \left (h_2 + r_1 d \right ) \end{aligned} s 1 ​ ​ = k − 1 ( h 1 ​ + t r 1 ​ d ) = t k − 1 ( t h 1 ​ ​ + r 1 ​ d ) = h 2 ​ h 1 ​ k − 1 ​ ( h 2 ​ + r 1 ​ d ) ​

Now, let

s 2 = t − 1 s 1 = h 2 h 1 h 1 k − 1 h 2 ( h 2 + r 1 d ) = k − 1 ( h 2 + r 1 d ) \begin{aligned} s_2 &= t^{-1} s_1 \\ &= \frac{h_2}{h_1}\frac{h_1 k^{-1}}{h_2} \left ( h_2 + r_1 d \right ) \\ &= k^{-1} (h_2 + r_1 d) \end{aligned} s 2 ​ ​ = t − 1 s 1 ​ = h 1 ​ h 2 ​ ​ h 2 ​ h 1 ​ k − 1 ​ ( h 2 ​ + r 1 ​ d ) = k − 1 ( h 2 ​ + r 1 ​ d ) ​

So ( r 1 , s 2 ) (r_1, s_2) (r1​,s2​) is a valid signature for h 2 h_2 h2​.

from pwn import * from Crypto . Util . number import long_to_bytes , bytes_to_long from hashlib import sha1 from client import init , get_first_signature , send_second_signature p = 0xffffffff00000001000000000000000000000000ffffffffffffffffffffffff a = 0xffffffff00000001000000000000000000000000fffffffffffffffffffffffc b = 0x5ac635d8aa3a93e7b3ebbd55769886bc651d06b0cc53b0f63bce3c3e27d2604b E = EllipticCurve ( GF ( p ) , [ a , b ] ) n = 0xffffffff00000000ffffffffffffffffbce6faada7179e84f3b9cac2fc632551 h1 = bytes_to_long ( sha1 ( b'Server says 1+1=2' ) . digest ( ) ) h2 = bytes_to_long ( sha1 ( b'Server says 1+1=3' ) . digest ( ) ) t = h1 * inverse_mod ( h2 , n ) % n tube = init ( ) r1 , s1 = get_first_signature ( tube , t ) s2 = h2 * inverse_mod ( h1 , n ) * s1 % n send_second_signature ( tube , r1 , s2 )

Flag: CTF{TheySayTheEmptyCanRattlesTheMost}
