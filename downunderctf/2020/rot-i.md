# ROT-I

**CTF:** DownUnderCTF 2020  
**Category:** crypto  
**Original writeup:** <https://jsur.in/posts/2020-09-20-downunderctf-2020-writeups#rot-i> — by **jsur.in**. All credit goes to the original author; this is an archived copy.

---

rot-i (100pts) - 447 solves

ROT13 is boring!

Ypw'zj zwufpp hwu txadjkcq dtbtyu kqkwxrbvu! Mbz cjzg kv IAJBO{ndldie_al_aqk_jjrnsxee}. Xzi utj gnn olkd qgq ftk ykaqe uei mbz ocrt qi ynlu, etrm mff'n wij bf wlny mjcj :).

Solution

The title hints at the well known Caesar Shift Cipher. If we try to ROT13 or use substitution cipher solvers on the ciphertext, we don't get any useful results. There are a couple things that we can observe to help us proceed. The first is that the title is called "rot-i". This hints at each element being rotated by a different amount. Punctuation has also been preserved, so we can use this to guess what the plaintext might be. Specifically, the first word looks like it might say "You're" or "You've", and indeed if we look a bit closer, Y = Y + 0 , p = o + 1 , w = u + 2 , and so on... It looks like each letter is shifted by i positions, where i is its index in the message.

Solve script:

from string import ascii_lowercase , ascii_uppercase c = "Ypw'zj zwufpp hwu txadjkcq dtbtyu kqkwxrbvu! Mbz cjzg kv IAJBO{ndldie_al_aqk_jjrnsxee}. Xzi utj gnn olkd qgq ftk ykaqe uei mbz ocrt qi ynlu, etrm mff'n wij bf wlny mjcj :)." def decrypt ( ct ) : out = '' for i , c in enumerate ( ct ) : if c in ascii_lowercase : alph = ascii_lowercase elif c in ascii_uppercase : alph = ascii_uppercase else : out += c continue out += alph [ ( alph . index ( c ) - i ) % len ( alph ) ] return out print ( decrypt ( c ) )

Which gives:

You've solved the beginner crypto challenge! The flag is DUCTF{crypto_is_fun_kjqlptzy}. Now get out some pen and paper for the rest of them, they won't all be this easy :).
