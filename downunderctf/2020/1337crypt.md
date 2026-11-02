# 1337crypt

**CTF:** DownUnderCTF 2020  
**Category:** crypto  
**Original writeup:** <https://jsur.in/posts/2020-09-20-downunderctf-2020-writeups> — by **jsur.in**. All credit goes to the original author; this is an archived copy.

---

1337crypt (500pts) - 3 solves

1337crypt.sage :

from Crypto . Util . number import getPrime , bytes_to_long from random import randint flag = open ( 'flag.txt' , 'rb' ) . read ( ) . strip ( ) p , q = getPrime ( 1337 ) , getPrime ( 1337 ) n = p * q D = ( 1 * 3 * 3 * 7 ) ^ ( 1 + 3 + 3 + 7 ) hint = int ( D * sqrt ( p ) + D * sqrt ( q ) ) x = randint ( 1337 , n ) while 1337 : lp = legendre_symbol ( x , p ) lq = legendre_symbol ( x , q ) if lp * lq > 0 and lp + lq < 0 : break x = randint ( 1337 , n ) m = map ( int , bin ( bytes_to_long ( flag ) ) [ 2 : ] ) c = [ ] for b in m : while 1337 : r = randint ( 1337 , n ) if gcd ( r , n ) == 1 : break c . append ( ( pow ( x , 1337 + b , n ) * pow ( r , 1337 + 1337 , n ) ) % n ) print ( f'hint = { hint } ' , f'D = { D } ' , f'n = { n } ' , f'c = { c } ' , sep = '

' )
