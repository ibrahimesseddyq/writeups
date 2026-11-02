# yahtzee

**CTF:** redpwn 2021  
**Category:** crypto  
**Original writeup:** <https://github.com/vndctv/ctf/blob/master/2021/redpwn/crypto/yahtzee/README.md> — by **vndctv**. All credit goes to the original author; this is an archived copy.

---

# crypto/yahtzee Writeup

*Challenge Description: Pseudo-random number generators are weak! I use only true RNGs, like rolling a set of dice!*

## Code Analysis

Since we are provided with the source code, we can go through the code and spot any potential weak points that may be succeptible to attack.

When a client connects, they are asked for input. Any input other than `quit` will return a ciphertext to the user:

```

============================================================================

= Welcome to the yahtzee message encryption service. =

= We use top-of-the-line TRUE random number generators... dice in a cup! =

============================================================================

Would you like some samples?

yes pls

Ciphertext: 8790ca3f1656d7f0afc8f678814f2f4d8d7480513f2cf52d752...[truncated]

Would you like some more samples, or are you ready to 'quit'?

```

To generate the message, the server picks a random quote from a file, then inserts the flag string into the quote at a random location:

```python

def random_message():

NUM_QUOTES = 25

quote_idx = randint(0,NUM_QUOTES-1)

with open('quotes.txt','r') as f:

for idx, line in enumerate(f):

if idx == quote_idx:

quote = line.strip().split()

break

quote.insert(randint(0, len(quote)), flag)

return ' '.join(quote)

```

The server then encrypts the message using AES counter mode and sends it to the client:

```python

def encrypt(message, key, true_rng):

nonce = true_rng.next()

cipher = AES.new(key, AES.MODE_CTR, nonce = long_to_bytes(nonce))

return cipher.encrypt(message)

```

## Crypto Implementation

Let's take a look under the hood at the server's crypto implementation, in particular, the nonce that comes from the `TrueRNG` class method `.next()`:

```python

class TrueRNG:

@staticmethod

def die():

return randint(1, 6)

@staticmethod

def yahtzee(N):

dice = [TrueRNG.die() for n in range(N)]

return sum(dice)

def __init__(self, num_dice):

self.rolls = num_dice

def next(self):

return TrueRNG.yahtzee(self.rolls)

```

The `TrueRNG.next()` method calls `TrueRNG.yahtzee(N)` which returns the sum of *N* dice. In the case of the server's implementation, `true_rng` is only set to use two dice. The maximum value it can ever return is 12, and because the sum of two dice can never amount to anything less than 2, there are only 10 possible nonce values.

## Plan of Attack

I'll preface this with saying that I'm not a "crypto guy" by any means, so my terminology and technique may be awful, and a lot of this (as I suspect the case to be with most of this type of cryptanalysis) was slow manual work adjusting my code.

Anyway... Given that we can generate multiple different outputs with a 10% chance of a nonce being reused, and we know a bit of plaintext, it may be possible to XOR all permutations together and "drag for cribs."

```python

#!/usr/bin/python3

from pwn import xor

from binascii import unhexlify

class CipherPair:

def __init__(self, ciphertexts):

self.ciphertexts = ciphertexts

self.xored = xor(self.ciphertexts[0], self.ciphertexts[1])

self.results = drag_crib(b'flag{', self.xored)

def gen_permuations(ciphertexts):

perms = []

for _ in range(len(ciphertexts) - 1):

key = ciphertexts.pop()

perms += [CipherPair([key, ciphertext]) for ciphertext in ciphertexts]

return perms

def drag_crib(crib, ciphertext):

results = []

for i in range(len(crib)):

key = crib[i:] + crib[:i]

results.append(xor(key, ciphertext))



return results

with open('ciphertexts.txt', 'r') as f:

ciphertexts = [unhexlify(line.rstrip()) for line in f]

perms = gen_permuations(ciphertexts)

for perm in perms:

idx = perms.index(perm)

if idx in [44, 55, 90,]:

print(f'CipherPair {perms.index(perm)}')

for result in perm.results:

print(f'{str(result)}')

print()

```

## Dragging for Cribs

The first crib we will drag for is `flag{`, as we know every single flag in the CTF starts with this string. It's not much to work with, but with some luck, it might be enough to start with.

When I first ran this crib against my XOR permutations, I was able to identify some, albeit short, plaintext in some of the cipher pairs. In my case, pairs 44, 55, and 90:

```

CipherPair 44

b"Defin?p[^*$k\x1e|u#n'fTgP[t|a>%\x7fg;?\\&h\x7f%ao/ a pr9`KS;wwX3~z>1}VkVS'asp2rq5lX(se8 (m4bi~8s/fh>}p,fpq+\xc2O#\x8c\xb1\r"

b'Nh`us5}]B7.f\x18`h)c!zIm]]hak3#cz12Z:uu(gs2*l&lo3mMO&}z^/cp37aKa[U;|y}4nl?a^4no5&4p>oob%y"`t#w}*zm{&\xc4S>\x86\xbc\x0b'

b"Cn|hy8{A_=#`\x04}b$e=gC`[Aukf5?~p<4F'\x7fx.{n8'j:qe>kQR,p|B2i}5+|Al]I&vt{(sf2gB)db3:)z3is\x7f/t$|i)z{6ggv \xd8N4\x8b\xba\x17"

b'Erabt>g\\U0%|\x19wo"y mNfG\\\x7ff`)"t}:([-r~2fd5!v\'{h8wLX!v`_8d{)6vLjAT,{rg5yk4{_#id/\'#w5unu"r8ac$|g+mjp<\xc5D9\x8d\xa6

'

b'Yokor"zVX69a\x13zi>d*`HzZVr`|4(y{&5Q tb/li3=k-vn$jFU\'j}U5bg4<{Jv\\^!}nz?tm(fU.ox2-.q)hdx$n%kn"`z!`ll!\xcfI?\x91\xbb\x00'

CipherPair 55

b'We mu%p\x1eR.;yPpv#}=zWxJ]!zgmqtmu?]*j\x7f%`f/1d5\x7f)jgzwl/lr46y cir5qSC;6v]v`->\x1b4Ee\x03_te`q5baolm`+ba owvh<\\`e\xd6]'

b"]h&qh/}\x18N31tVlk)p;fJrG[=gm`whp\x7f2[6wu(fz2;i3c4`j|kq%at(+s-euo?|U_&<{[j}'3\x1d(Xo\x0eYhxj|3~|eak|6hl&sj|e:@}o\xdb["

b'Pn:lb"{\x04S9<rJqa$v\'{@\x7fAG m`fkuzr4G+}x.zg86o/~>ml`v{(gh5!~+yhe2zIB,1}Gww*5\x015Rb\x08Eurgz/cvhgwa<ej:n`qc&]wb\xddG'

b'Vr\'fo$g\x19Y4:nW{l"j:qMy]Z*`fzv\x7fwt(Z!p~2gm50s2t3kp}|v.{u?,x7dbh4fTH!7aZ}z,)\x1c?_d\x14X\x7f\x7faf2i{n{jk1cv\'dmw\x7f;Wzd\xc1Z'

b"Jo-ki8z\x13T2&s]vj>w0|Ke@P'fzg|rqh5P,vb/m`3,n8y5wmwqp2f\x7f2*d*non({^E'+|Pp|04\x162Yx\tRry}{8d}rf`f7\x7fk-ikkb1Z|x\xdcP"

CipherPair 90

b"You m7}\x1eR*w|W`rsn=}J|FZt|t>(xw;*I.v'lm}{ft.b)hzq$i*b`<&w\x13U:8qFx|ju1q]f\x07\x13\x1b)ny|p>v4gj}uw(.h=+l.v>aj|z4|m6\xab"

b'Sbs<p=p\x18N7}qQ|oyc;aWvK\\ha~3.dj1\'O2k-akafly(~4bww8t of ;}\x1eS&%{K~`w\x7f<wA{\r\x1e\x1d5ssqv"k>jlah}%(t !a(j#kgzf)v`0\xb7'

b"^do!z0v\x04S=pwMaete'|]{M@uks52y`oqk%~-iz=1p\x18O;/vMb}}r:k\\q\x00\x18\x01(y~wj?a3lp|bp#4i*,g4w)faf{#{f,\xaa"

b'Xxr+w6j\x19Y0vkPkhry:vP}Q]\x7ffu)/sm:=N%l&{jvagc)i3imv/s+ug7<v\x04R1"pQ\x7fwpt&vV|\x06\x04\x1c"txkw5l5pmvov?)c\'*{)}$`}{q.}z1\xa0'

b'Dex&q*w\x13T6jvZfnnd0{VaLWr`i4%~k& D(j:f`{g{~#d5up|"u7hm::j\x19X<$lLuzvh;|[z\x1a\x19\x16/rdv}8j)mg{ij"#n!6f#p"|`q|(ag;\xad'

```

Let's change our crib to `Definite` and observe the output. Below I've isolated pair 44 to demonstrate that we're getting closer to a flag:

```

CipherPair 44

b"flag{0h_}4$n\x11rz F#gFoUNvCb4-vu)6y$up'i|1\x02h'~g6xOp%wrW=qy\x165|DcSF%^pz:{c'e}*nj:(;s\x16knp-|7bK }u#h\x7fr\x03\xc6N1\x84\xb4\x18"

b'Gon`|-y~\\7+i\x16ok\x01g hAhH_Wba;*qh8\x17X\'zw tm\x10#k(y`+inQ&xuP `X76sCdNW\x04\x7fsu=|~6D\\)am=5*R7haw*a&Cj#rr$unS"\xc5A6\x83\xa9\t'

b'D`iga<X__8,n\x0b~J d/oFuY~van<-ly\x196[(}p=eL1 d/~}:HOR)\x7frM1Ay49tDy_v%||r:ao\x17e_&fj $\x0bs4gfp7p\x07bi,uu9dOr!\xcaF1\x9e\xb8('

b"Kgnzp\x1dy\\P?+s\x1a_k#k(h[dx_uni;0}X85T/zm,Dm2/c(cl\x1biL].xo\\\x10`z;>sYh~W&s{u'pN6fP!aw1\x05*p;`am&Q&af+rh(Enq.\xcdA,\x8f\x99\t"

b'L`skQ<zSW86b;~h,l/uJEY\\zin&!\\y;:S(g|\ren=(d5rM:jCZ)e~}1cu<9nHI_T)t|h6Qo5iW&|f\x10$)\x7f<g||\x07p%na,oy\tdm~)\xca\\=\xae\xb8

'

b"K}bJp?uTP%'C\x1a}g+k2dkdZS}ns7\x00}z4=T5v],fa:/y$Sl9eD]4t_\\2lr;$\x7fih\\[.say\x17pl:nP;mG1'&x;zm]&s*if1~X(gby.\xd7M\x1c\x8f\xbb\x05"

b'VlCks0rSM4\x06b\x19r`,v#EJgUTzsb\x16!~u3:I$W|/if=2h\x05ro6bC@%U~_=ku&5^HkS\\)npX6sc=iM*Lf2(!\x7f&kL|%|-n{ _y+he~3\xc6l=\x8c\xb4\x02'

b'GMbh|7uN\\\x15\'a\x16ug1g\x02dIhRSgbC7"qr4\'X\x05v\x7f na #I$q`1e^Q\x04t}P:lh7\x14\x7fKdT[4\x7fQy5|d:t\\\x0bme=/&b7Jm\x7f*{*sj\x01~z$obc"\xe7M>\x83\xb3\x05'

```

It's not easy grinding out three characters at a time, so I took a few minutes to see if I could identify any other chunks of plaintext that might be Googleable/guessable strings. As most of this was a manual process for me, I'll cut to the chase.

Through all the manual grinding, I noticed what looked like a part of the phrase "a product of my circumstances". Using this crib gave me a nice large chunk of the flag in pair 44:

```

CipherPair 44

[...]

b"A}'asyqC\x190bw\rtj0a2!@g\x1cWj'fr4js9&^53v/ e-flag{0h_W41t_ther1:]\x7fUV5ya<<s*>y\x19.(s&.+c1z(v%5.~/$;l?nob$\xd7\x087\x8c\xfd\x01"

[...]

```

Our new known flag plaintext is: `flag{0h_W41t_ther`, which gives us enough of pair 44 to Google for the full quote:

```

CipherPair 44

b'Definiteness of p mNfG

{XPfulC);X3uu(gsd.R\x16#8+S^\\%c}U5bgb8Ez9\x0bF\x1fn}y!sf2gB\x7f`\\\x03u~b\rz`|1s/fh>+t\x12V?&3\xfc\\0\x8f\xaf

'

```

Full quote identified! ***Definiteness of purpose is the starting point of all achievement.*** Using this as the crib reveals the flag at the start of cipher pair 44:

`b'flag{0h_W41t_ther3s_nO_3ntr0py} I am not[...]`
