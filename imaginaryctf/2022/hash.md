# hash

**CTF:** ImaginaryCTF 2022  
**Category:** crypto  
**Original writeup:** <https://imp.ress.me/blog/2022-07-19/imaginaryctf-2022-crypto/> — by **imp.ress.me**. All credit goes to the original author; this is an archived copy.

---

By Eth007 My passwords are safe and secure with the use of sha42! nc hash.chal.imaginaryctf.org 1337 Attachments: hash.py jbox.txt 39 solves, 378 points

sha42 has poor confusion. Each byte of the hash output is effectively the xor of some subset of the initial input bytes, depending on the length of the input byte array. We can do a (dumb) symbolic run of the hash to identify which input bytes affect each output byte, then use these correlations with z3 to solve for a possible input which produces some hash.

Since the challenge service does not compare input lengths, we can fix a length of 30 and will always get (some) solution since we have 30 unknowns and 21 equations.

Exploit code python from collections import defaultdict from tqdm.auto import tqdm from z3 import * from pwn import * # original implementation config = [ [ int (a) for a in n.strip()] for n in open ( "hash/jbox.txt" ).readlines() ] def sha42 (s: bytes , rounds = 42 ): out = [ 0 ] * 21 for round in range (rounds): for c in range ( len (s)): if config[((c // 21 ) + round ) % len (config)][c % 21 ] == 1 : out[(c + round ) % 21 ] ^= s[c] return bytes (out).hex() # generate confusion relations confusion = defaultdict( list ) # confusion[hash_index] = [input_char_indexes] length = 30 for target in range (length): probe = [ 0 for _ in range (length)] probe[target] = 1 probe = bytes (probe) res = sha42(probe) res = bytes .fromhex(res) for idx, f in enumerate (res): if f == 1 : confusion[idx].append(target) # unhash solver def unhash (chal): solver = Solver() cs = [ BitVec( f "c { i } " , 8 ) for i in range (length) ] for sink in range ( 21 ): rhs = 0 for s in confusion[sink]: rhs = rhs ^ cs[s] solver.add(rhs == chal[sink]) solver.check() model = solver.model() res = bytes ([ model[s].as_long() for s in cs ]) return res # sanity check chal = sha42( b "some_16char_pass" ) chal = bytes .fromhex(chal) assert sha42(unhash(chal)) == chal.hex() # run conn = remote( "hash.chal.imaginaryctf.org" , 1337 ) for _ in tqdm( range ( 50 )): conn.recvuntil( b "sha42(password) = " ) chal = conn.recvline() chal = bytes .fromhex(chal.strip().decode()) soln = unhash(chal) conn.sendline(soln.hex().encode()) print (conn.recvall())

output [x] Opening connection to hash.chal.imaginaryctf.org on port 1337 [x] Opening connection to hash.chal.imaginaryctf.org on port 1337: Trying 34.90.15.213 [+] Opening connection to hash.chal.imaginaryctf.org on port 1337: Done 0%| | 0/50 [00:00<?, ?it/s] [x] Receiving all data [x] Receiving all data: 16B [x] Receiving all data: 95B [+] Receiving all data: Done (95B) [*] Closed connection to hash.chal.imaginaryctf.org port 1337 b'hex(password) = Correct!

Congrats! Your flag is: ictf{pls_d0nt_r0ll_y0ur_0wn_hashes_109b14d1}
