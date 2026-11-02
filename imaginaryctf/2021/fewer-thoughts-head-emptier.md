# Fewer Thoughts, Head Emptier

**CTF:** ImaginaryCTF 2021  
**Category:** rev  
**Original writeup:** <https://github.com/DerBaer0/CTF-writeups/blob/main/2021-imaginaryctf/Misc-FewerToughs.md> — by **DerBaer0**. All credit goes to the original author; this is an archived copy.

---

## Reversing Brainfuck

Now, we get the numbers: `105 5664 5066 6888 5376 7742`. The first one is `i`, but the others are too large for ASCII. Now, I went onto reversing the tail part of the Brainfuck Code. I did this using a sheet of paper and stepping the online interpreter to visualize as much as possible what is happening. I start by picking out some important parts (and added line numbers for the explanation). `(H) marks the position of the head, _ a value not relevant for the current aspect`

#### The Doubler

```

1 [->+>++<<] # This is a loop that transforms: X 0 0 into 0 X 2*X

2 >>

3 [-<<+>>] # This loop moves a value two cells to the left: 0 _ X to X _ 0

# Together, they change X(H) 0 0 to 2*X X 0(H)

```

Followed by some moving stuff we look at later, follwed by a loop doing the important stuff:

```

1 [<

2 [->>-<<] # A _ B to 0 _ B-A

3 >-

4 [-<+<+>>]<<[->>+<<] # similar to 3-5: transform 0 0 X to 0 X 2*X

5 >>] # from X Y(HEAD) Z, to 0 Y Z-X, to 0 Y-1 Z-X, to Y 2*(Y-1) Z-X

6 >>

7 [>]

8 >-]

```

#### The Gauss Subtraction

```

1 [<

2 [->>-<<]

3 >-

4 [-<+<+>>]

5 <<

6 [->>+<<]

7 >>]

```

```

(0) initial setup

v (the Head)

0 X X D E (when the loop runs, both X have the same value)

--------------

(1) go left

v

0 X X D E

--------------

(2) subtraction

v

0 0 X D-X E

--------------

(3) go right, decrement

v

0 0 X-1 D-X E

--------------

(4) double

v

X-1 X-1 0 D-X E

--------------

(4) left, copy, right

v

0 X-1 X-1 D-X E

```

We are back at the initial state with a two differences:

D was decremented by X, X got reduced by 1 (in both positions). As this happens inside a loop, we continue substracting X-1 from D-X and reduce X-1 to X-2 and so forth. In The end, we subtracted all numbers between X and 1 from D. The sum of values between 1 and X is `(X + 1) * X / 2` (Gauss Sum), so D is reduced by this value in total.

#### The repeating Printer

```

<

[

[<]<[<]> # go to the leftmost non-zero cell (a bit complicated, because it needs to skip a zero cell in the middle)

. # print it

[>]>[>]< # go to the rightmost on-zero cell

-] # decrement and repeat

```

So, this function prints left leftmost most as many times as the rightmost cell indicates

#### The big loop

Putting our snippets together with some *glue* code, the big loop in the end reads as this (Note that there is a 0 between `B` and `K`. `B` is a normal character, and `K` is the number of characters in the flag. It is set similar to all other values, but there is an additional `>` beforehead to create the zero space.). In this case, it is assumed the flag only has two characters `A` and `B`.

```

Setup 0 0 A B 0 K(H) X 0 0

The Doubler 0 0 A B 0 K 2X X 0(H)

The repeating Printer 0 0 A B 0 K 2X 0(H) 0

Move to the far left 0 0 A(H) B 0 K 2X 0 0

Duplicate the value 0 A A(H) B 0 K 2X 0 0

The Gauss Subtraction 0 0 0(H) B-Gauss(A) 0 K 2X 0 0

Go right, decrement and Repeat 0 0 0(H) B-Gauss(A) 0 K-1(H) 2X 0 0

```

So, basically, we have a value `X` starting with 1 (directly before the big loop begins). We double `X`, print the current character `X` times, decrement the value of the second character by the gauss sum of the first character and go back to repeat by doubling `X`. This prints the first character once, the second character twice, the thid character 4 times and and so forth.

## The Script

```python

import sys

def get_num(ls):

pms = [0] * 4

for i in range(4):

pms[i] = (ls[i].count('+'), ls[i].count('-'))

val = max(pms[0]) * max(pms[1])

mul = 1

if ls[3][2] == '-':

mul = -1

val += mul * max(pms[2]) * max(pms[3])

return val

def gauss(x):

return (x+1) * x // 2

bf = open(sys.argv[1]).readlines() # open the (formatted! file)

ls = [x.strip() for x in bf[:128]] # take all characters (the value 124 is taken by looking at the source)

ls = [get_num(ls[i:i+4]) for i in range(0, len(ls), 4)] # compute all values

for i in range(1, len(ls)): # apply the gaussian subtraction

ls[i] = ls[i] - gauss(ls[i-1])

print("".join(map(chr, ls)))

```

The flag: `ictf{th3_l3s5_th0ugh+_+he_b3teR}`
