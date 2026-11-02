# stream

**CTF:** ImaginaryCTF 2022  
**Category:** crypto  
**Original writeup:** <https://imp.ress.me/blog/2022-07-19/imaginaryctf-2022-crypto/> — by **imp.ress.me**. All credit goes to the original author; this is an archived copy.

---

By Eth007 HELP! I encrypted my files with this program I downloaded from the internet. Can you recover my 42-byte flag? Attachments: stream out.txt 37 solves, 390 points

Open stream binary with ghidra and grok.

Decompile of stream binary

The binary encrypts text with this procedure:

Read an input file specified in argv[1]

Process 8 bytes of input at a time as a 64 bit unsigned integer

Xor the input integer with a 64 bit unsigned integer key specified in argv[2]

Square the key

Repeat until all of the input file is processed

Write the output to a file specified in argv[3]

What happens to input files that need padding to the nearest 8 bytes? We can run the script locally with a dummy key to test.

bash $ python -c "print('\x00' * 42, end='')" > zeros.txt $ ./stream zeros.txt 7 zeros.enc.txt $ hexdump -C -v zeros.txt 00000000 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 | ................ | 00000010 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 | ................ | 00000020 00 00 00 00 00 00 00 00 00 00 | .......... | 0000002a $ hexdump -C -v zeros.enc.txt 00000000 07 00 00 00 00 00 00 00 31 00 00 00 00 00 00 00 | ....... .1. ...... | 00000010 61 09 00 00 00 00 00 00 c1 f6 57 00 00 00 00 00 | a.........W..... | 00000020 81 7d 05 a5 39 1e 00 00 01 3b 9b 6e 58 33 3c 30 | . } . .9. ... ; .nX3<0 | 00000030 $ python -c "print(hex(pow(7, 2 ** 5, 2 ** 64)))" 0x303c33586e913b01

It seems like the trailing bits are padded with zeros before xor. Since the flag is 42 bytes there will be 6 zero bytes of padding. These tail 6 bytes of enc then leaks the high 6 bytes of key during the final iteration.

Assuming the flag starts with ictf{ , we can use this to leak the low 5 bytes of key during the first iteration. The key * key operation is done on a unsigned long long , which acts like multiplication modulo 2**64 . As a result of this nice modulo, the k k k lower bits of key in later iterations only depends on the same k k k lower bits of key in previous iterations. We can manually square key from first iteration to get the low 5 bytes of key in the final iteration. This way, we have leaked key .

We can recover key of earlier iteration using iterative square root modulo prime power of key from later iterations. The exploit uses z3 to calculate this square root. We can speed this up by early pruning if some key produces a non-printable flag.

Exploit code python import struct from z3 import * from tqdm.auto import tqdm from string import printable from itertools import product # for some reason sage is slow, so z3 to the rescue def sqrt_mod_2_64 (x): x = int (x) res = [] a = BitVec( "a" , 64 ) while True : s = Solver() s.add(a * a == x) for soln in res: s.add(a != soln) if str (s.check()) == "unsat" : break soln = s.model()[a].as_long() res.append(soln) return res with open ( "stream/out.txt" , "rb" ) as f: enc_raw = f.read() enc_raw, len (enc_raw) enc = struct.unpack( "<" + "Q" * 6 , enc_raw) key_low_first = (struct.unpack( "<Q" , b "ictf{ \x00\x00\x00 " )[ 0 ] ^ enc[ 0 ]) & 0x ffffffffff key_low_final = pow (key_low_first, 2 ** 5 , 2 ** 64 ) & 0x ffffffffff print ( "final key low 5 bits" .ljust( 22 ), hex (key_low_final).rjust( 18 )) key_high_final = enc[ 5 ] & 0x ffffffffffff0000 print ( "final key high 6 bits" .ljust( 22 ), hex (key_high_final)) key_final = key_low_final | key_high_final print ( "final key" .ljust( 22 ), hex (key_final)) charset = set (printable.encode()) def search (idx, key, data = "" ): pln_chunk = bytes .fromhex( hex ((enc[idx] ^ key) + 2 ** 64 )[ 3 :]) if idx == 5 : pln_chunk = pln_chunk[ - 2 :] if not all ([i in charset for i in pln_chunk]): return None data = pln_chunk[:: - 1 ].decode() + data print (idx, data.strip()) if idx == 0 : return prev_keys = sqrt_mod_2_64(key) for prev_key in prev_keys: search(idx - 1 , prev_key, data) search( 5 , key_final)

output final key low 5 bits 0x241d06d81 final key high 6 bits 0x2297b40241d00000 final key 0x2297b40241d06d81 5 } 4 901bf2e4} 3 ystream_901bf2e4} 2 ed_my_keystream_901bf2e4} 1 _rec0vered_my_keystream_901bf2e4} 0 ictf{y0u_rec0vered_my_keystream_901bf2e4}
