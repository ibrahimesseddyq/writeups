# tiramisu

**CTF:** GoogleCTF 2021  
**Category:** crypto  
**Original writeup:** <https://jsur.in/posts/2021-07-19-google-ctf-2021-crypto-writeups> — by **jsur.in**. All credit goes to the original author; this is an archived copy.

---

Tiramisu

In this challenge, you establish a secure channel with the server. Good luck! attachment tiramisu.2021.ctfcompetition.com 1337

Initial Observations

We are given the code running on a server and some client code to interact with the server. The server is written in Go and performs key exchanges using ECDH. The server and client use protobuf to exchange data, but this is handled for us in the given client code so we can ignore it.

The interaction between the client and server is simple; when the client connects, the server sends its public key and the flag encrypted with its private key. The client then sends its public key and an encrypted and authenticated message. The server attempts to decrypt and verify the ciphertext, and if successful, encrypts the message and echoes it back to the client.

Upon analysing the code, we noticed:

server_main.go:118 : the server reads its private key and curve from a file; it is the same for every interaction (the public key and encrypted flag don't change even if we reconnect)

: the server reads its private key and curve from a file; it is the same for every interaction (the public key and encrypted flag don't change even if we reconnect) pb_util.go:25 : the server only supports curves secp224r1 and secp256r1 ; attempting to use any other curve causes the server to disconnect

: the server only supports curves and ; attempting to use any other curve causes the server to disconnect server.go:104 : the server ensures that the client provided public key point is on the curve specified by the client; this means our point must be on either secp224r1 or secp256r1

: the server ensures that the client provided public key point is on the curve specified by the client; this means our point must be on either or crypto/elliptic/elliptic.go:73 : the IsOnCurve check will pass even if the coordinates x and y lie outside of the range [0, p-1] (where p is the order of the finite field that the curve is defined over):

func ( curve * CurveParams ) IsOnCurve ( x , y * big . Int ) bool { y2 := new ( big . Int ) . Mul ( y , y ) y2 . Mod ( y2 , curve . P ) return curve . polynomial ( x ) . Cmp ( y2 ) == 0 }

server.go:111 : when computing its shared secret, the server reduces the x and y coordinates of the client's public key modulo its own curve's prime (i.e. modulo p224 )

: when computing its shared secret, the server reduces the and coordinates of the client's public key modulo its own curve's prime (i.e. modulo ) cipher.go:92 : if the client provided MAC does not match the expected match computed from the server's derived shared secret, the server disconnects

And perhaps most importantly, the goal is to recover the server's static private key, but we gain no information about it other than from the last observation; the MAC we compute matches the server's expected MAC if and only if we derive the same shared secret. This means, to gain any information about the server's private key, we'll need to use it as an oracle; guessing things we think might be correct, and verifying it with the server.

Attack Ideas

After seeing that the server checks that the client provided public key point is on the curve, it might seem that invalid curve attacks are out of the question. But it's highly suspicious that the server supports both secp224r1 and secp256r1 , despite only ever using secp224r1 for its own public key, and that it reduces the client's public key point modulo p224 regardless of the client's curve. It then continues to work with this point, scalar multiplying it by its private key. Note that only the curve parameters a a a and p p p are used in point addition; the curve's b b b is not relevant for addition, but affects the order of the curve. It turns out that this is enough to mount an invalid curve attack. Specifically, if we as the client, specify that our curve is secp256r1 and give a point on secp256r1 , once the server reduces the coordinates modulo p224 , it may no longer lie on secp256r1 nor secp224r1 .

The High-Level Overview

We will find points that lie on secp256r1 , but when reduced modulo p224 , have small order on the new curve that it lies on. We send this point as our public key, specifying curve secp256r1 so that we pass the IsOnCurve check. Then, because the point has small order (when reduced modulo p224 ), when the server derives the shared secret point by scalar multiplying our small order point with its private key, there will be only a small number of possible results. Specifically, there will be up to n different shared secret points the server may compute, where n is the order of the point (when reduced modulo p224 ). We can ascertain which is the correct value by multiplying this point by values up to n and using this as the shared secret point. We derive the secret key from it and send an authenticated message to the server. If the server echoes a message back, we've found the correct value. This gives us the value of the server's private key modulo n . By doing this many times with different points and coprime n , we can gather enough information to recover the entire private key by combining the results with the Chinese Remainder Theorem.

A Slightly More Detailed Overview

We denote the curve parameters of secp224r1 and secp256r1 with subscripts (e.g. p 256 , a 224 p_{256}, a_{224} p256​,a224​, and so on..)

Choose a random integer b b b and consider the curve E_b : E a 224 , b ( F p 224 ) E_{a_{224},b}(\mathbb{F}_{p_{224}}) E a 224 ​ , b ​ ( F p 224 ​ ​ ) Choose a random point G G G on the curve and compute its order; hopefully, and with reasonable probability, the order has a small prime factor n n n (small enough that we're happy to bruteforce on the remote server for: around 1000 to 60000 ) Let ( x , y ) = ∣ G ∣ n G (x, y) = \frac{|G|}{n}G ( x , y ) = n ∣ G ∣ ​ G . Then, ( x , y ) (x, y) ( x , y ) is a point on E_b with order n At the moment, ( x , y ) (x, y) ( x , y ) does not lie on secp256r1 , but we can "lift" it to ( X , Y ) (X, Y) ( X , Y ) such that it lies on secp256r1 when reduced modulo p256 , but equals ( x , y ) (x, y) ( x , y ) on E_b when reduced modulo p224 For i = 0..n : Connect to the server and send ( X , Y ) (X, Y) ( X , Y ) as the public key, specifying curve secp256r1 Compute a shared secret point by multiplying ( x , y ) (x, y) ( x , y ) by i (as a point on the curve E_b ) Use the key derivation function on this point to encrypt a message and compute a MAC. Send this to the server. If the server echoes a message back, we have recovered partial information about the server's private key. Repeat steps 1-5 for different values of b b b and coprime n n n until the product of all such n n n exceed p224 (the expected size of the private key) Use the Chinese Remainder Theorem to recover the private key fully

There are a few details missing. We will explore the full attack in the following sections.

Steps 1-4: Generating Good Curves and Points

Our goal is to find values ( X , Y ) (X, Y) (X,Y) such that reduction modulo p256 gives a valid point on secp256r1 , and reducing modulo p224 gives a point of small order on some insecure curve. We begin by choosing a random b < p 224 b \lt p_{224} b<p224​. We then construct the curve E_b given by E a 224 , b ( F p 224 ) E_{a_{224}, b}(\mathbb{F}_{p_{224}}) Ea224​,b​(Fp224​​). We choose a random point on E_b and compute its order. If a small prime factor n n n divides the order, we compute the point P = ( x , y ) = ∣ G ∣ n G P = (x, y) = \frac{|G|}{n} G P=(x,y)=n∣G∣​G which has order n n n. Now let

X = x m = p 224 p 256 t ≡ p 224 − 1 ( m o d p 256 ) s ≡ p 256 − 1 ( m o d p 224 ) L ≡ a 224 p 256 s + a 256 p 224 t ( m o d m ) C ≡ b p 256 s + b 256 p 224 t ( m o d m ) u ≡ X 3 + L X + C ( m o d m ) \begin{aligned} X &= x \\ m &= p_{224} p_{256} \\ t &\equiv p_{224}^{-1} \pmod{p_{256}} \\ s &\equiv p_{256}^{-1} \pmod{p_{224}} \\ L &\equiv a_{224} p_{256} s + a_{256} p_{224} t \pmod m \\ C &\equiv b p_{256} s + b_{256} p_{224} t \pmod m \\ u &\equiv X^3 + LX + C \pmod m \end{aligned} X m t s L C u ​ = x = p 224 ​ p 256 ​ ≡ p 224 − 1 ​ ( mod p 256 ​ ) ≡ p 256 − 1 ​ ( mod p 224 ​ ) ≡ a 224 ​ p 256 ​ s + a 256 ​ p 224 ​ t ( mod m ) ≡ b p 256 ​ s + b 256 ​ p 224 ​ t ( mod m ) ≡ X 3 + L X + C ( mod m ) ​

If u u u is a quadratic residue modulo p 224 p_{224} p224​ and modulo p 256 p_{256} p256​ then we can write u ≡ Y 2 ( m o d m ) u \equiv Y^2 \pmod m u≡Y2(modm). So

Y 2 ≡ X 3 + L X + C ( m o d m ) Y^2 \equiv X^3+ LX + C \pmod m Y 2 ≡ X 3 + L X + C ( mod m )

We can recover y y y by computing the square roots of u u u modulo p 224 p_{224} p224​ and modulo p 256 p_{256} p256​, and then combining the results with CRT.

Now, note that

L ≡ a 224 ( m o d p 224 ) C ≡ b ( m o d p 224 ) \begin{aligned} L &\equiv a_{224} \pmod{p_{224}} \\ C &\equiv b \pmod{p_{224}} \end{aligned} L C ​ ≡ a 224 ​ ( mod p 224 ​ ) ≡ b ( mod p 224 ​ ) ​

(because the p 224 p_{224} p224​ factors are killed by the mod p 224 p_{224} p224​, and p 256 s ≡ 1 ( m o d p 224 ) p_{256}s \equiv 1 \pmod{p_{224}} p256​s≡1(modp224​)).

Therefore,

Y 2 ≡ X 3 + a 224 X + b ( m o d p 224 ) Y^2\equiv X^3 + a_{224}X + b \pmod{p_{224}} Y 2 ≡ X 3 + a 224 ​ X + b ( mod p 224 ​ )

so the point, modulo p 224 p_{224} p224​, lies on E_b as we wanted!

Similarly, reducing modulo p 256 p_{256} p256​, we get

L ≡ a 256 ( m o d p 256 ) C ≡ b 256 ( m o d p 256 ) \begin{aligned} L &\equiv a_{256} \pmod{p_{256}} \\ C &\equiv b_{256} \pmod{p_{256}} \end{aligned} L C ​ ≡ a 256 ​ ( mod p 256 ​ ) ≡ b 256 ​ ( mod p 256 ​ ) ​

so as we hoped,

Y 2 ≡ X 3 + a 256 X + b 256 ( m o d p 256 ) Y^2 \equiv X^3 + a_{256}X + b_{256} \pmod{p_{256}} Y 2 ≡ X 3 + a 256 ​ X + b 256 ​ ( mod p 256 ​ )

So ( X , Y ) (X, Y) (X,Y) lies on secp256r1 when reduced modulo p 256 p_{256} p256​, and lies on E_b when reduced modulo p 224 p_{224} p224​.

The following Sage script finds such b b b and good curves/points:

def get_valid_br_point ( br ) : E = EllipticCurve ( GF ( p224 ) , [ a224 , br ] ) G = E . random_point ( ) n = G . order ( ) for z in GOOD_PRIMES : if n % z == 0 : break else : return None , None , None P = ( n // z ) * G m = p224 * p256 x = mod ( int ( P . xy ( ) [ 0 ] ) , m ) linear_coef = mod ( p256 * a224 * inverse_mod ( p256 , p224 ) + p224 * inverse_mod ( p224 , p256 ) * a256 , m ) const_coef = mod ( p256 * inverse_mod ( p256 , p224 ) * br + p224 * inverse_mod ( p224 , p256 ) * b256 , m ) ysqr = mod ( x ^ 3 + linear_coef * x + const_coef , m ) if kronecker ( ysqr , p224 ) == 1 and kronecker ( ysqr , p256 ) == 1 : ysp224 = Integer ( sqrt ( mod ( ysqr , p224 ) ) ) ysp256 = Integer ( pow ( ysqr , ( p256 + 1 ) // 4 , p256 ) ) y = mod ( CRT ( [ ysp224 , ysp256 ] , [ p224 , p256 ] ) , m ) P_on_E = E ( int ( x ) % p224 , int ( y ) % p224 ) return ( int ( x ) , int ( y ) ) , P_on_E , z return None , None , None p224 = 0xffffffffffffffffffffffffffffffff000000000000000000000001 a224 = 0xfffffffffffffffffffffffffffffffefffffffffffffffffffffffe p256 = 0xffffffff00000001000000000000000000000000ffffffffffffffffffffffff a256 = 0xffffffff00000001000000000000000000000000fffffffffffffffffffffffc b256 = 0x5ac635d8aa3a93e7b3ebbd55769886bc651d06b0cc53b0f63bce3c3e27d2604b GOOD_PRIMES = [ p for p in range ( 2 ^ 9 , 2 ^ 16 ) if Integer ( p ) . is_prime ( ) ] USED_FACTORS = [ ] while True : br = randint ( 1 , p224 ) point , P_on_E , z = get_valid_br_point ( br ) if point is None : continue else : x , y = point if z not in USED_FACTORS : print ( br , x , y , z ) USED_FACTORS . append ( z )

Step 5: Requesting the Oracle

We modified the run_session function in the provided client code to take integers x , y and shared_point . It runs the session as usual but uses the provided (x, y) as the public key, specifying secp256r1 as the curve. It derives the shared secret key from the provided shared_point and sends an encrypted and authenticated message. If it receives an echo reply, it returns True, otherwise, it returns False.

The following Sage function implements step 5.

def exhaust_shared_point ( XY , P_on_E , n ) : for k in range ( 1 , n ) : shared_point = ( k * P_on_E ) . xy ( ) [ 0 ] X , Y = XY res = run_session ( int ( X ) , int ( Y ) , int ( shared_point ) ) if res : return k

Because we used subgroups of sizes between 2^9 to 2^16 (why we used such large subgroups instead of smaller ones will be the topic of the next section), the bruteforce on the remote server would have taken a lot of time. To make it run faster, we ran the script on a GCP compute instance in Belgium, which is where the challenge server was located.

Steps 6-7: Recovering the Private Key

For step 6, we simply repeat step 5 with all of the different b b b and points we found in steps 1-4. Doing so gives us a set of n n n residues k 1 , … , k n k_1, \ldots, k_n k1​,…,kn​ and corresponding pairwise co-prime moduli m 1 , … , m n m_1, \ldots, m_n m1​,…,mn​ such that ∏ i = 1 n m i ≥ p 224 \prod_{i=1}^n m_i \geq p_{224} ∏i=1n​mi​≥p224​ and

k ≡ k 1 ( m o d m 1 ) k ≡ k 2 ( m o d m 2 ) ⋮ k ≡ k n ( m o d m n ) \begin{aligned} k &\equiv k_1 \pmod{m_1} \\ k &\equiv k_2 \pmod{m_2} \\ \vdots \\ k &\equiv k_n \pmod{m_n} \\ \end{aligned} k k ⋮ k ​ ≡ k 1 ​ ( mod m 1 ​ ) ≡ k 2 ​ ( mod m 2 ​ ) ≡ k n ​ ( mod m n ​ ) ​

where k k k is the server's private key. This can be solved using the Chinese Remainder Theorem. However, there is a subtlety we need to consider. Because the computation of the shared secret point disregards the y y y value of the result, there are actually two values for each k i k_i ki​ that satisfy k ≡ k i ( m o d m i ) k \equiv k_i \pmod{m_i} k≡ki​(modmi​). Specifically, if k ≡ k i ( m o d m i ) k \equiv k_i \pmod{m_i} k≡ki​(modmi​), then we also have k ≡ − k i ( m o d m i ) k \equiv -k_i \pmod{m_i} k≡−ki​(modmi​). Therefore, for each congruence we have in the system, we need to consider both values. This means we will need to perform an offline 2 n 2^n 2n search. For this reason, we chose to use larger subgroup sizes to ensure the offline part does not take too long.

The following Sage script prints a list of candidate private keys using data we obtained from the previous steps:

from itertools import product p = 0xffffffffffffffffffffffffffffffff000000000000000000000001 def generate_candidates ( K , M ) : candidate_K = [ ( k , m - k ) for k , m in zip ( K , M ) ] all_K = list ( product ( * candidate_K ) ) res = [ ] for K_ in all_K : res . append ( CRT ( list ( K_ ) , M ) % p ) return res K = [ 266 , 307 , 24 , 378 , 254 , 106 , 2231 , 1473 , 2700 , 127 , 743 , 121 , 2846 , 898 , 2734 , 3350 , 2668 , 2611 , 14559 ] M = [ 643 , 823 , 919 , 977 , 1013 , 1019 , 4507 , 5039 , 5479 , 5689 , 5839 , 6143 , 6451 , 12503 , 12611 , 17137 , 20789 , 29759 , 62633 ] candidates = generate_candidates ( K , M ) print ( '

' . join ( map ( str , candidates ) ) )

Finally, we adapted some of the provided Go code to perform decryption and find the flag:

package main import ( "crypto/aes" "crypto/cipher" "crypto/sha256" "fmt" "io" "io/ioutil" "math/big" "strings" "golang.org/x/crypto/hkdf" ) func DeriveKey ( secret , info [ ] byte ) ( [ ] byte , error ) { hash := sha256 . New hkdf := hkdf . New ( hash , secret , nil , info ) key := make ( [ ] byte , aes . BlockSize ) if _ , err := io . ReadFull ( hkdf , key ) ; err != nil { return nil , err } return key , nil } func main ( ) { channelCipherKdfInfo := [ ] byte ( "Flag Cipher v1.0" ) c_data := [ ] byte ( ">}\"B\352\"WgA\234*\014p\326b\\O6\374\250\217K\343\334U\374\252~\267\026\325\212J\3178M\354{q\231\201\310\351yyj`3_\224^\313\204P\200\323\233=" ) c_iv := [ ] byte ( "s@v\325g\340\t*\274\341\t\025\202UC}" ) dat , _ := ioutil . ReadFile ( "./candidate_keys" ) lines := strings . Split ( string ( dat ) , "

" ) for _ , line := range lines { secret := make ( [ ] byte , 28 ) D := new ( big . Int ) D . SetString ( line , 10 ) D . FillBytes ( secret ) key , _ := DeriveKey ( secret , channelCipherKdfInfo ) aes_cipher , _ := aes . NewCipher ( key ) stream := cipher . NewCTR ( aes_cipher , c_iv ) plaintext := make ( [ ] byte , len ( c_data ) ) stream . XORKeyStream ( plaintext , c_data ) if string ( plaintext ) [ : 3 ] == "CTF" { fmt . Println ( string ( plaintext ) ) } } }

Flag: CTF{ChocolateDoesNotAskSillyQuestionsChocolateUnderstands}
