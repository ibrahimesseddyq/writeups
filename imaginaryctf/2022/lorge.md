# Lorge

**CTF:** ImaginaryCTF 2022  
**Category:** crypto  
**Original writeup:** <https://imp.ress.me/blog/2022-07-19/imaginaryctf-2022-crypto/> — by **imp.ress.me**. All credit goes to the original author; this is an archived copy.

---

By Robin_Jadoul I guess smoll needed a revenge after all 😭 Attachments: lorge.py output.txt 29 solves, 433 points

From lorge.py , we see that p − 1 p-1 p−1 and q − 1 q-1 q−1 are 2 25 2^{25} 225 smooth. We first try to factor n n n using Pollard p-1 attack and generator g = 2 g=2 g=2, but at some point g g g goes from a coprime number to 1 1 1 straight away. This is because some largest prime factor x x x of p − 1 p-1 p−1 and q − 1 q-1 q−1 are the same. Since the initial run stopped at the largest prime factor x x x, we can instead start with generator g = 2 x m o d n g = 2^x \mod n g=2xmodn and re-try the attack which succeeds.

Exploit code python from tqdm.auto import tqdm from gmpy2 import mpz, next_prime, log, floor, powmod, gcd with open ( "lorge/output.txt" , "r" ) as f: raw = f.read() n, e, ct = [ int (i.partition( " = " )[ 2 ]) for i in raw.strip().split( "

" )] n = mpz(n) cap = mpz( 2 ** 25 ) def pollard_rho_step (g, q): r = mpz(floor(log(cap) / log(q))) z = q ** r return powmod(g, z, n) leak = [] while True : g = mpz( 2 ) cur = mpz( 2 ) for i in leak: g = pollard_rho_step(g, i) pbar = tqdm( total = int (cap)) pbar.update( int (cur)) while cur < cap: g = pollard_rho_step(g, cur) check = gcd(g - 1 , n) if check != 1 : break nx = next_prime(cur) delta = nx - cur pbar.update( int (delta)) cur = nx if g == 1 : print ( "leak common prime factor" , cur) leak.append(cur) else : break p = gcd(g - 1 , n) q = n // p assert (p * q == n) phi = (p - 1 ) * (q - 1 ) d = powmod(e, - 1 , phi) pln = powmod(ct, d, n) bytes .fromhex( hex ( int (pln))[ 2 :])

output 0%| | 0/33554432 [00:00<?, ?it/s] leak common prime factor 19071329 0%| | 0/33554432 [00:00<?, ?it/s] b'ictf{why_d1d_sm0ll_3v3n_sh0w_up_on_f4ct0rdb???_That_m4d3_m3_sad!}'
