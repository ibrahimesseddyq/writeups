# impECCable

**CTF:** DownUnderCTF 2020  
**Category:** crypto  
**Original writeup:** <https://jsur.in/posts/2020-09-20-downunderctf-2020-writeups> — by **jsur.in**. All credit goes to the original author; this is an archived copy.

---

impECCable (500pts) - 3 solves

Can you crack this flawless ECC signature scheme? nc chal.duc.tf 30203

import ecdsa import random import hashlib Curve = ecdsa . NIST384p G = Curve . generator n = Curve . order flag = open ( 'flag.txt' , 'r' ) . read ( ) . strip ( ) auth_msg = b'I know alll of your secrets!' def inv ( z , n = n ) : return pow ( z , - 1 , n ) def gen_keypair ( ) : d = random . randint ( 1 , n - 1 ) Q = d * G return d , Q def sign ( msg , d ) : x = int ( hashlib . sha1 ( int . to_bytes ( d , 48 , byteorder = 'big' ) ) . hexdigest ( ) , 16 ) % 2 ** 25 while True : k1 = ( random . getrandbits ( 340 ) << 25 ) + x k2 = ( random . getrandbits ( 340 ) << 25 ) + x r1 = ( k1 * G ) . x ( ) r2 = ( k2 * G ) . y ( ) if r1 != 0 or r2 != 0 : break h = int ( hashlib . sha384 ( msg ) . hexdigest ( ) , 16 ) s = inv ( k1 ) * ( h * r1 - r2 * d ) % n return ( r1 , r2 , s ) def verify ( msg , Q , sig ) : if any ( x < 1 or x >= n for x in sig ) : return False r1 , r2 , s = sig h = int ( hashlib . sha384 ( msg ) . hexdigest ( ) , 16 ) v1 = h * r1 * inv ( s ) v2 = r2 * inv ( s ) x1 = ( v1 * G + ( - v2 % n ) * Q ) . x ( ) return ( x1 - r1 ) % n == 0 def menu ( ) : m = '''Here are your options: [S]ign a message [V]erify a signature [P]ublic Key [Q]uit''' print ( m ) choice = input ( ) [ 0 ] . lower ( ) if choice == 's' : print ( 'Enter your message (hex):' ) msg = bytes . fromhex ( input ( ) ) if len ( msg ) >= 8 : print ( 'Message too long!' ) exit ( ) sig = sign ( msg , d ) print ( ' ' . join ( map ( str , sig ) ) ) elif choice == 'v' : print ( 'Enter your message (hex):' ) msg = bytes . fromhex ( input ( ) ) print ( 'Enter your signature:' ) sig = [ int ( x ) for x in input ( ) . split ( ) ] if verify ( msg , Q , sig ) : if msg == auth_msg : print ( 'Hello there authenticated user! Here is your flag:' , flag ) exit ( ) else : print ( 'Verified!' ) else : print ( 'Invalid Signature!' ) elif choice == 'p' : print ( Q . x ( ) , Q . y ( ) ) else : print ( 'Oh ok then... Bye!' ) exit ( ) d , Q = gen_keypair ( ) print ( 'Welcome to my impECCable signing service.' ) for _ in range ( 11 ) : menu ( )

Solution

ECDSA-like Signature Scheme

The server implements an ECDSA-like signature scheme which is summarised below.

With public parameters ( p , E ( F p , a , b ) , G , n , H ) (p, E(\mathbb{F}_p, a, b), G, n, H) (p,E(Fp​,a,b),G,n,H) ( n n n is the curve order, H H H is the hash function)

Key Generation: The signer chooses a private key d ∈ [ 1 , n − 1 ] d \in [1, n-1] d∈[1,n−1] and computes their public key Q = d G Q = dG Q=dG

Signing a message m m m:

Calculate h = H ( m ) h = H(m) h = H ( m ) Select random k 1 , k 2 ∈ [ 1 , n − 1 ] k_1, k_2 \in [1, n-1] k 1 ​ , k 2 ​ ∈ [ 1 , n − 1 ] Calculate r 1 = ( k 1 G ) x r_1 = (k_1 G)_x r 1 ​ = ( k 1 ​ G ) x ​ , r 2 = ( k 2 G ) y r_2 = (k_2 G)_y r 2 ​ = ( k 2 ​ G ) y ​ . If r 1 = r 2 = 0 r_1 = r_2 = 0 r 1 ​ = r 2 ​ = 0 , go back to step 2 Calculate s ≡ k − 1 ( h r 1 − r 2 d ) ( m o d n ) s \equiv k^{-1} (hr_1 - r_2d) \pmod n s ≡ k − 1 ( h r 1 ​ − r 2 ​ d ) ( mod n ) The signature is ( r 1 , r 2 , s ) (r_1, r_2, s) ( r 1 ​ , r 2 ​ , s )

Verifying a message m m m and signature ( r 1 , r 2 , s ) (r_1, r_2, s) (r1​,r2​,s):

Ensure r 1 , r 2 , s ∈ [ 1 , n − 1 ] r_1, r_2, s \in [1, n-1] r 1 ​ , r 2 ​ , s ∈ [ 1 , n − 1 ] . Return invalid if not Calculate h = H ( m ) h = H(m) h = H ( m ) Calculate v 1 = h r 1 s − 1 v_1 = hr_1 s^{-1} v 1 ​ = h r 1 ​ s − 1 and v 2 = r 2 s − 1 v_2 = r_2 s^{-1} v 2 ​ = r 2 ​ s − 1 Calculate ( x 1 , y 1 ) = v 1 G − v 2 Q (x_1, y_1) = v_1 G - v_2 Q ( x 1 ​ , y 1 ​ ) = v 1 ​ G − v 2 ​ Q The signature is valid if x 1 ≡ r 1 ( m o d n ) x_1 \equiv r_1 \pmod n x 1 ​ ≡ r 1 ​ ( mod n ) .

The signature scheme itself is probably secure, but an implementation flaw allows us to recover the private key given a few signatures. Specifically, the nonce k 1 k_1 k1​ is biased and all nonces have only 340 bits of randomness, with a shared suffix. This kind of attack has been covered in the literature (https://eprint.iacr.org/2019/023.pdf). The attack works by picturing the situation as a hidden number problem instance, which can be solved using lattice-based techniques.

Hidden Number Problem

A (simplified) version of the hidden number problem can be stated as follows.

Let p p p be a prime and let d ∈ F p d \in \mathbb{F}_p d∈Fp​ be an unknown integer. Recover d d d given pairs of integers { ( t i , a i ) } i = 1 m \{ (t_i, a_i) \}_{i=1}^m {(ti​,ai​)}i=1m​ such that

k i − t i d − a i ≡ 0 ( m o d p ) k_i - t_i d - a_i \equiv 0 \pmod p k i ​ − t i ​ d − a i ​ ≡ 0 ( mod p )

and ∣ k i ∣ < B |k_i| < B ∣ki​∣<B for some B < p B < p B<p. The k i k_i ki​ are unknown.

This problem can be easily solved given a sufficient number of pairs and small enough B B B.

Consider the lattice generated by the rows of the matrix

M = [ p 0 0 ⋯ 0 0 0 p 0 ⋯ 0 0 ⋮ ⋱ ⋮ 0 0 ⋯ p 0 0 t 1 t 2 ⋯ t m B / p 0 a 1 a 2 ⋯ a m 0 B ] M = \begin{bmatrix} p & 0 & 0 & \cdots & 0 & 0 \\ 0 & p & 0 & \cdots & 0 & 0 \\ \vdots & & \ddots & & & \vdots \\ 0 & 0 & \cdots & p & 0 & 0 \\ t_1 & t_2 & \cdots & t_m & B/p & 0 \\ a_1 & a_2 & \cdots & a_m & 0 & B \end{bmatrix} M = ⎣ ⎡ ​ p 0 ⋮ 0 t 1 ​ a 1 ​ ​ 0 p 0 t 2 ​ a 2 ​ ​ 0 0 ⋱ ⋯ ⋯ ⋯ ​ ⋯ ⋯ p t m ​ a m ​ ​ 0 0 0 B / p 0 ​ 0 0 ⋮ 0 0 B ​ ⎦ ⎤ ​

We'll call the first m m m rows of this matrix p p p-rows. Notice that if we multiply the second-to-last row by d d d, add the last row to it, and subtract appropriate multiples of the p p p-rows, we get the vector

v k = ( k 1 , k 2 , … , k m , B d / p , B ) v_k = (k_1, k_2, \ldots, k_m, Bd/p, B) v k ​ = ( k 1 ​ , k 2 ​ , … , k m ​ , B d / p , B )

which is a short vector since the k i k_i ki​ are small, so can be recovered by solving SVP!

Finding the HNP Instance

Our goal is to turn the signatures from the service into an instance of the HNP. It should be quite obvious that the hidden number is the private key d d d, and the unknown k i k_i ki​'s are the biased nonces. In fact, if we look at how the verification algorithm works, we see that it almost tells us how to write the equations. Let

t i = − 2 − 25 r i 2 s i − 1 a i = 2 − 25 h i r i 1 s i − 1 t_i = -2^{-25}r_{i2} s_i^{-1} \qquad a_i = 2^{-25}h_ir_{i1} s_i^{-1} t i ​ = − 2 − 25 r i 2 ​ s i − 1 ​ a i ​ = 2 − 25 h i ​ r i 1 ​ s i − 1 ​

Each nonce has 340 bits of randomness, plus 25 bits that we don't know (but is a common suffix for all nonces). It turns out that just knowing that the top 19 bits are all 0 isn't quite enough to recover the private key as we only have a limited number of samples. We'll need to optimise a bit. Suppose we are given m m m signatures by the server. What we can do is eliminate the unknown common suffix by taking the m − 1 m-1 m−1 points of data given by { t i ′ = t 0 − t i } i = 1 m \{t_i' = t_0 - t_i\}_{i=1}^m {ti′​=t0​−ti​}i=1m​ and { a i ′ = a 0 − a i } i = 1 m \{a_i' = a_0 - a_i\}_{i=1}^m {ai′​=a0​−ai​}i=1m​ as our inputs to the hidden number problem.

Write the nonce k i k_i ki​ as k i = p i 2 25 + c k_i = p_i 2^{25} + c ki​=pi​225+c where p i p_i pi​ is the 340 random bits, and c c c is the unknown common suffix.

To help make the algebra a bit less messy, let b i = ( h i r i 1 − r i 2 d ) − 1 b_i = (h_i r_{i1} - r_{i2}d)^{-1} bi​=(hi​ri1​−ri2​d)−1.

Then,

t i ′ ≡ t 0 − t i ≡ − 2 − 25 ( r 02 k 0 b 0 − r i 2 k i b i ) ( m o d n ) \begin{aligned} t_i' &\equiv t_0 - t_i \\ &\equiv -2^{-25} (r_{02} k_0 b_0 - r_{i2} k_i b_i) \pmod n \end{aligned} t i ′ ​ ​ ≡ t 0 ​ − t i ​ ≡ − 2 − 25 ( r 02 ​ k 0 ​ b 0 ​ − r i 2 ​ k i ​ b i ​ ) ( mod n ) ​

and

a i ′ ≡ a 0 − a i ≡ 2 − 25 ( h 0 r 01 k 0 b 0 − h i r i 1 k i b i ) ( m o d n ) \begin{aligned} a_i' &\equiv a_0 - a_i \\ &\equiv 2^{-25} (h_0 r_{01} k_0 b_0 - h_i r_{i1} k_i b_i) \pmod n \end{aligned} a i ′ ​ ​ ≡ a 0 ​ − a i ​ ≡ 2 − 25 ( h 0 ​ r 01 ​ k 0 ​ b 0 ​ − h i ​ r i 1 ​ k i ​ b i ​ ) ( mod n ) ​

So,

t i ′ d + a i ′ ≡ 2 − 25 ( − d ( r 02 k 0 b 0 − r i 2 k i b i ) + h 0 r 01 k 0 b 0 − h i r i 1 k i b i ) ≡ 2 − 25 ( h 0 r 01 k 0 b 0 − r 02 k 0 b 0 d − ( h i r i 1 k i b i − r i 2 k i b i d ) ) ≡ 2 − 25 ( k 0 b 0 ( h 0 r 01 − r 02 d ) − k i b i ( h i r i 1 − r i 2 d ) ) ≡ 2 − 25 ( k 0 b 0 b 0 − 1 − k i b i b i − 1 ) ≡ 2 − 25 ( k 0 − k i ) ≡ 2 − 25 ( p 0 2 25 + c − ( p i 2 25 + c ) ) ≡ p 0 − p i ( m o d n ) \begin{aligned} t_i' d + a_i' &\equiv 2^{-25}(-d(r_{02} k_0 b_0 - r_{i2} k_i b_i) + h_0 r_{01} k_0 b_0 - h_i r_{i1} k_i b_i) \\ &\equiv 2^{-25}(h_0 r_{01} k_0 b_0 - r_{02} k_0 b_0 d -(h_i r_{i1} k_i b_i - r_{i2} k_i b_i d)) \\ &\equiv 2^{-25}(k_0 b_0 (h_0 r_{01} - r_{02} d) - k_i b_i (h_i r_{i1} - r_{i2} d)) \\ &\equiv 2^{-25} (k_0 b_0 b_0^{-1} - k_i b_i b_i^{-1}) \\ &\equiv 2^{-25} (k_0 - k_i) \\ &\equiv 2^{-25} (p_0 2^{25} + c - (p_i 2^{25} + c)) \\ &\equiv p_0 - p_i \pmod n \end{aligned} t i ′ ​ d + a i ′ ​ ​ ≡ 2 − 25 ( − d ( r 02 ​ k 0 ​ b 0 ​ − r i 2 ​ k i ​ b i ​ ) + h 0 ​ r 01 ​ k 0 ​ b 0 ​ − h i ​ r i 1 ​ k i ​ b i ​ ) ≡ 2 − 25 ( h 0 ​ r 01 ​ k 0 ​ b 0 ​ − r 02 ​ k 0 ​ b 0 ​ d − ( h i ​ r i 1 ​ k i ​ b i ​ − r i 2 ​ k i ​ b i ​ d )) ≡ 2 − 25 ( k 0 ​ b 0 ​ ( h 0 ​ r 01 ​ − r 02 ​ d ) − k i ​ b i ​ ( h i ​ r i 1 ​ − r i 2 ​ d )) ≡ 2 − 25 ( k 0 ​ b 0 ​ b 0 − 1 ​ − k i ​ b i ​ b i − 1 ​ ) ≡ 2 − 25 ( k 0 ​ − k i ​ ) ≡ 2 − 25 ( p 0 ​ 2 25 + c − ( p i ​ 2 25 + c )) ≡ p 0 ​ − p i ​ ( mod n ) ​

Therefore...

( p 0 − p i ) − t i ′ d − a i ′ ≡ 0 ( m o d n ) (p_0 - p_i) - t_i' d - a_i' \equiv 0 \pmod n ( p 0 ​ − p i ​ ) − t i ′ ​ d − a i ′ ​ ≡ 0 ( mod n )

and this is precisely the hidden number problem setting described above! Even better, it turns out that we have enough signatures to recover the private key with this optimisation.

All that's left is to throw the matrix at LLL, recover the private key, sign the auth message and grab the flag!

Solve script:

import os os . environ [ 'PWNLIB_NOTERM' ] = 'True' from pwn import remote , process import ecdsa import hashlib Curve = ecdsa . NIST384p G = Curve . generator n = Curve . order def connect ( ) : return remote ( '0.0.0.0' , 1337 , level = 'error' ) def recv ( conn ) : o = conn . recvline ( ) . decode ( ) print ( '[<]' , o ) return o def send ( conn , d ) : print ( '[>]' , d ) conn . sendline ( d ) def read_menu ( conn ) : [ recv ( conn ) for _ in range ( 5 ) ] def get_sig ( conn , msg ) : read_menu ( conn ) send ( conn , 's' ) recv ( conn ) send ( conn , msg . hex ( ) ) sig = [ int ( x ) for x in recv ( conn ) . split ( ' ' ) ] return sig def verify ( conn , msg , sig ) : read_menu ( conn ) send ( conn , 'v' ) recv ( conn ) send ( conn , msg . hex ( ) ) recv ( conn ) send ( conn , ' ' . join ( str ( x ) for x in sig ) ) return recv ( conn ) def recover_privkey ( n , hashes , sigs , B , lo ) : Zn = Zmod ( n ) N = len ( sigs ) - 1 M = [ [ 0 ] * i + [ n ] + [ 0 ] * ( N - i - 1 + 2 ) for i in range ( N ) ] Ts = [ int ( Zn ( - r2 / ( ( 2 ^ lo ) * Zn ( s ) ) ) ) for r1 , r2 , s in sigs ] M . append ( [ int ( Zn ( Ts [ 0 ] - t ) ) for t in Ts [ 1 : ] ] + [ B / n , 0 ] ) As = [ int ( Zn ( h * r1 / ( ( 2 ^ lo ) * Zn ( s ) ) ) ) for h , ( r1 , r2 , s ) in zip ( hashes , sigs ) ] M . append ( [ int ( Zn ( As [ 0 ] - a ) ) for a in As [ 1 : ] ] + [ 0 , B ] ) M = Matrix ( M ) M = M . LLL ( ) for row in M : if row [ - 1 ] == B : return Zn ( row [ - 2 ] * n / B ) conn = connect ( ) recv ( conn ) hashes = [ ] sigs = [ ] for m in range ( 10 ) : msg = os . urandom ( 6 ) hashes . append ( int ( hashlib . sha384 ( msg ) . hexdigest ( ) , 16 ) ) sigs . append ( get_sig ( conn , msg ) ) d = recover_privkey ( int ( n ) , hashes , sigs , 2 ^ 340 , 25 ) print ( '[+] recovered d:' , d ) auth_msg = b'I know alll of your secrets!' h = int ( hashlib . sha384 ( auth_msg ) . hexdigest ( ) , 16 ) k1 , k2 = 133 , 337 r1 = ( k1 * G ) . x ( ) r2 = ( k2 * G ) . y ( ) s = inverse_mod ( k1 , n ) * ( Integer ( h * r1 ) - r2 * d ) % n verify ( conn , auth_msg , ( r1 , r2 , s ) ) conn . close ( )

Flag: DUCTF{y0u_f0unD_7h3_h1dD3n_numB3r5!}
