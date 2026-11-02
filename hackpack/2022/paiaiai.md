# paiaiai

**CTF:** HackPack 2022  
**Category:** crypto  
**Original writeup:** <https://www.sebven.com/ctf/2022/04/12/HackPackCTF2022-Paiaiai.html> — by **sebven.com**. All credit goes to the original author; this is an archived copy.

---

Cryptography – 469 pts (15 solves) – Chall author: Polymero (me)

Pai-ai-ai… My Paillier scheme seems to be broken and I stored my favourite flag in it. Please help me get it back, will you? Who could have guessed this would ever happen? … Me… I- I wrote it… yeah.

nc cha.hackpack.club 10997 or 20997

Files: paiaiai.py

This challenge was part of my guest appearance on HackPack CTF 2022.





Exploration

A broken Paillier scheme, ay? We will have to see about that, so let’s check out that netcat address.

| | __________ ___ _____ .___ ___ /\ ________ | \______ \ / / / _ \ | | \ \ / \ \_____ \ | | ___/ / / / /_\ \| | \ \ \/\/ _(__ < | | | ( ( / | \ | ) ) / \ | |____| \ \ \____|__ /___| / / /______ / | \__\ \/ /__/ \/ | | | MENU: | [E]ncrypt | [D]ecrypt | [Q]uit | | >>>

A simple encryption and decryption oracle by the looks of it. But is there a flag somewhere, or…?

| >>> e | | ENCRYPT: | [F]lag | [M]essage | | >>> f | | FLAG: 5034785281950343508744711185197053413579683684148346815055060643830096040661868151655540533876088575122326236411117844472809056644810603591168338896947065051838001190612137050286239757216637769428688458121795534498425150558799778262152211380500502315518078744021314932495226813697971002436193528611124439176499215601182718102181293884893725133193746104582566161413721747512035451101644872652626262911480381217826003138422858719320658061016453762663219079891957566134362778008467545382769109618240098365542282971245141679577428279057904116622344521602573879167018369528424811143805230006833934692544590370705590706749

Ah, there it is. How about we just try to decrypt the encrypted flag using the oracle. It’s worth a shot, no?

| >>> d | | CIP: int | > 5034785281950343508744711185197053413579683684148346815055060643830096040661868151655540533876088575122326236411117844472809056644810603591168338896947065051838001190612137050286239757216637769428688458121795534498425150558799778262152211380500502315518078744021314932495226813697971002436193528611124439176499215601182718102181293884893725133193746104582566161413721747512035451101644872652626262911480381217826003138422858719320658061016453762663219079891957566134362778008467545382769109618240098365542282971245141679577428279057904116622344521602573879167018369528424811143805230006833934692544590370705590706749 | | MSG: 12871600377284678436598137255971594702778717571805789843087607005526820834753398893936142416141847404877983796452226917450460827082602907623668019954000946671758045381893241895902842877012194250175399626949591728996536054349500689273433699378169394830786556638694244611253646771790397394644439916576778434039

It did not complain, so let’s see that juicy flag!

b'\x12Tk\xa2h?\xad\xd9\x7f\xf70\x17\xb0\x0e,L\t\x85@\x9aU\x146\xf1\x89\xcbG\x98\xe0\x90\x18\xc4\xfbz\xd0\xd7\xca\\\xe4\xcd\xcdk\xff)<\x92EI&\x7f\xec\x89\xb2K\xf2c\xe0\x80=\xf3\x8e\x1b\x00P\x80\xe3a\x00 x\xa3\xeeJ8\x1c/\xbb\x16A\x9c\xbd\xefK\xd8{\xc3\xff\x1b%\xbc\xd2W\xe7\xa2^\x11\x15v\xebB\xb2\x11\xcfA\x04Y\xf0\xcb\x03x\xbcpX\x90\xb6\xe8\xeb\xabF\xbaw\x82v\xc6\xd0\x7f5\xf7'

Oh… Mmh… Maybe somebody should go and tell them their Paillier scheme is broken. Oh, they already know? Right, right… Down into the source code we go! First stop, the main server loop. I added some comments to help guide you through the code.

# Just for you sanity assert len ( FLAG ) > 64 # Server oracle pai = Paiaiai () while True : try : print ( MENU ) choice = input ( "| >>> " ). lower (). strip () # Encrypt option if choice == 'e' : print ( "|

| ENCRYPT:" ) print ( "| [F]lag" ) print ( "| [M]essage" ) subchoice = input ( "|

| >>> " ). lower (). strip () # Encrypt the flag if subchoice == 'f' : enc_flag = pai . encrypt ( FLAG ) print ( "|

| FLAG:" , enc_flag ) # Encrypt input message elif subchoice == 'm' : msg = input ( "|

| MSG: str

| > " ) cip = pai . encrypt ( msg ) print ( "|

| CIP:" , cip ) # Decrypt option elif choice == 'd' : cip = input ( "|

| CIP: int

| > " ) msg = pai . decrypt ( int ( cip )) print ( "|

| MSG:" , msg ) # Exit option elif choice == 'q' : print ( "|

| Bai ~

|" ) break else : print ( "|

| Trai again ~

|" ) except KeyboardInterrupt : print ( "

|

| Bai ~

|" ) break except : print ( "|

| Aiaiai ~

|" )

Nothing unexpected so far, so let’s take a look at that Paiaiai class to get to know this Paillier-like scheme. Although at this point it should be noted that we seem not to have access to the Paillier public key, something to keep in mind.

class Paiaiai : """ My first Paillier implementation! So proud of it. ^ w ^ """ def __init__ ( self ): # Key generation p , q = [ getPrime ( 512 ) for _ in range ( 2 )] n = p * q # Public key self . pub = { 'n' : n , 'gp' : pow ( randbelow ( n ** 2 ), p , n ** 2 ), 'gq' : pow ( randbelow ( n ** 2 ), q , n ** 2 ) } # Private key self . priv = { 'la' : ( p - 1 ) * ( q - 1 ), 'mu' : inverse (( pow ( self . pub [ 'gp' ] * self . pub [ 'gq' ], ( p - 1 ) * ( q - 1 ), n ** 2 ) - 1 ) // n , n ) } def encrypt ( self , m : str ): m_int = int . from_bytes ( m . encode (), 'big' ) # Encrypting with one of two possible generators at random g = pow ([ self . pub [ 'gp' ], self . pub [ 'gq' ]][ randbelow ( 2 )], m_int , self . pub [ 'n' ] ** 2 ) r = pow ( randbelow ( self . pub [ 'n' ]), self . pub [ 'n' ], self . pub [ 'n' ] ** 2 ) return ( g * r ) % self . pub [ 'n' ] ** 2 def decrypt ( self , c : int ): # Decrypting with same private key regardless of generator choice cl = ( pow ( c , self . priv [ 'la' ], self . pub [ 'n' ] ** 2 ) - 1 ) // self . pub [ 'n' ] return ( cl * self . priv [ 'mu' ]) % self . pub [ 'n' ]

It appears this Paillier scheme uses two different generators, $g_p$ and $g_q$, which are both valid generators raised to the power of one of the two primes each. We see that during the encryption process only one of the generators is used and more specifically, chosen at random. However, during the decryption process the same private key is used regardless of generator choice. Furthermore, the private key is generated using the product of said generators instead. Obviously this scheme is not going to work… but can we help the author out? What a crypto noob hehe





Exploitation 1: Fuzzing

The initial exploitation to this faulty Paillier scheme I found through playing around with it a bit. By inspecting decrypted ciphertexts of a local copy I found the following interesting relationship

\[D_{\mu} \left( E_{g_{p}}(m) \right) \equiv \left\{ \begin{array}{c} 0 \mod{p} \\ m \mod{q} \end{array} \right.\]

and similarly for the other generator I found

\[D_{\mu} \left( E_{g_{q}}(m) \right) \equiv \left\{ \begin{array}{c} 0 \mod{q} \\ m \mod{p} \end{array} \right.\]

Frankly, this is exactly what we need. Not only does this allow us to recover the message knowing $p$ and $q$, it also allows us to recover $p$ and $q$ themselves! But first things first, let’s get both possible faulty decryptions of the flag.

# Get both possible faulty decrypted flag values flag_r = [] while len ( flag_r ) != 2 : k = get_decrypt ( get_flag ()) if k not in flag_r : flag_r += [ k ] flag_p , flag_q = flag_r

Next up is recovering the primes $p$ and $q$. We already know that getting a faulty decryption will yield a multiple of either $p$ or $q$ depending on the used generator. So if we can get two different messages encrypted with the same generator, we should be able to recover one of the primes from the greatest common devisor (GCD) of the faulty decryptions. Because we have no knowledge of the used modulus, we have to recover both primes this way.

# Try to recover prime factorisation of unknown public modulus using random faulty decryptions while True : rstr = [ '' . join ( random . sample ( ALP , 9 )) for _ in range ( 8 )] rdec = [ get_decrypt ( get_encrypt ( i )) for i in rstr ] p = max ([ GCD ( flag_p , i ) for i in rdec ]) q = max ([ GCD ( flag_q , i ) for i in rdec ]) for k in range ( 2 , 1_000_000 ): while not p % k : p //= k while not q % k : q //= k if isPrime ( p ) and isPrime ( q ): break

Having recovered the primes we can use the faulty decryptions of the flag to obtain the flag value modulo $p$ and $q$, respectively. Combining them through the chinese remainder theorem (CRT) we can easily recover the original flag modulo $pq$.

# Recover flag n = p * q flag_modp = flag_q % flag_p flag_modq = flag_p % flag_q flag , pub = crt ([ p , q ], [ flag_modp , flag_modq ]) FLAG = int ( flag ). to_bytes ( 128 , 'big' ). lstrip ( b " \x00 " ) print ( FLAG )

Ta-da!

flag{p41_41_41_1_d0nt_th1nk_th1s_1s_wh4t_p41ll13r_1nt3nd3d_3h}
