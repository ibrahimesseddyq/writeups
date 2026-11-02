# nooombers

**CTF:** DEFCON 2021 Quals  
**Category:** crypto  
**Original writeup:** <https://github.com/petitnau/writeups/tree/master/defcon2021quals/nooombers> — by **petitnau**. All credit goes to the original author; this is an archived copy.

---

# nooombers

> I really do not understand what this is about, but my cousin told me that once he managed to got a flag out of this. I was able to copy from my cousin's shell parts of his interactions with the server. You can find them in interaction1.txt and interaction2.txt. Unfortunately, they are incomplete.

> nooombers.challenges.ooo 8765

> Files:

> interaction1.txt f6aeae3c35be2f8a2d098d076e4bf7ac9f5ece24ddc88c9e7fb43016e049b89a

> interaction2.txt 6fd5facfefdbd20b82d651520edf56713ef585bad1b78788db06c4208d2da321

Solved by: Srdnlen

+ Roberto Pettinau (@petitnau)

+ Emanuele Cuncu (@Geladen)

+ M C (@zoopr)

+ Erik Piersante (@Piier)

# Analyzing the files

We are presented with two files and a service.

The files interaction1.txt and interaction2.txt contain two separate conversations with the server. Let's start looking at interaction1.txt, while also working with the service.

Since everything is in chinese and seems to talk about fish, we tried to replace each symbol or string of symbols with an identifier. It seems like every conversation begins with a triple chinese symbol that we'll call `>`. After the initial `>>>`, the service gives us 11 other chinese symbols, followed by another `>`. It then waits for an input.

Each connection we make has different symbols, so it looks like the characters are in some way randomized, even though they are the same throughout a single connection.

If we give in input anything that's not one of the first 10 symbols (we'll call the first ten `0-9`, and the eleventh `-`), it closes the connection. Depending on the picked option, the services asks us for 0, 1, 2 or 3 inputs, and then spits out a list of chinese strings.

Let's take one part of interaction1 as an example:

```

>>>

0

1

2

3

4

5

6

7

8

9

-

> 0

0 A

```

Here the server asks us which action we want to perform, we chose 0. The server spits out symbol 0 followed by a long string of chinese symbols "鱷鳘鰬鰛鳘鰑鳌鲯鳗鲳鱃鰢鲣鲴鲌鰊鲤鲜鰩鰽鱡鱒鳮鳖鰹鱖鳃鱘鰑鰊鰄鰆鱉鳼鱨鳢鱑鲀鲒鳆鰢鲔鰕鳤鳾鲳鲜鲉鱚鳞鲑鱔鳽鱄鳪鲥鰻鳲鲷鲉鱍鲚鲻鱲鰩鲜鰨鰶鰛鱪鱔鱭鳹鳤鳘鲻鱬鲈鲼鲭鲨鳨鱎鳯鳕鰷鲓鰣鲺鲜鳖鱻鱌鰫鱶鲓鰴鳶鳙鱽鳰鲼鲒鲵鱜鱸鰄鱒鰻鱱鲊鳧鲔鰢鰛鰲鳛鲵鲯鱤鰺鰮鱷鰐鱠鲓鲯鲷鲯鲙鳥鱝鰸鰚鱈鲒鲲鱱鳣鰂鳇鲫鳃鳾" (which we'll call A).

```

>>>

0

1

2

3

4

5

6

7

8

9

-

> 1

> A

1 A B C

```

We then try to use option 1, giving it as a parameter the string given by option 0 (A). It returns symbol 1 followed by our input, and then two strings, B and C. String B seems to be some sort of separator between input and output since it's present in option 1 2 3 4, but we'll see later that that is not the case.

Let's look at one last operation

```

>>>

0

1

2

3

4

5

6

7

8

9

-

> 3

> A

> C

3 A C B D

```

Here the user asks for option 3, and gives it two inputs: A and C (these are of course the same A and C as the earlier example, since they are both part of one big interaction with the server).

The server then answers with `3 A C B D`: the option we asked, our two inputs, a "separator", and an output D.

After a bunch of operations, the server seems to give us a warning `Too much! Disabling one option...`, and disables option 9.

Let's now look at interaction2.txt.

```

>>>

0

1

2

3

4

5

6

7

8

9

-

> 9

> A

> B

2 B C D

4 E D C F

4 A D C G

6 H F I J

6 K G I L

5 J L I M

0 N

1 N C O

3 M N C P

3 P O C A

9 E A B A

Correct signature! This is the flag:

```

The user asks for option 9, and gives it two inputs "A" and "B" (of course these are different strings than the A and B of interaction1, since every interaction changes symbols and symbol strings).

The server spits out a bunch of lines, each resembling the outputs of various operations in interaction1.txt. So it seems like option 9 performs a bunch of operations on our inputs:

+ `2 B C D`: option 2 on input B returning string D

+ `4 E D C F`: option 4 on inputs E and D returning string F

+ ...

+ `9 E A B A`: maybe option 9 on inputs E, A and B returning A?.

This last line is interesting because it seems to have "A" as an output and as an input.

We try connecting to the server and asking it to use option 9.

```

>>>

0

1

2

3

4

5

6

7

8

9

-

> 9

> A

> B

2 B C D

4 E D C F

4 A D C G

6 H F I J

6 K G I L

5 J L I M

0 N

1 N C O

3 M N C P

3 P O C Q

9 E A B Q

```

Here the server returns us with `9 E A B Q`, and doesn't give us the flag. We hypothesize that the server wants some A and B such that "`9 E A B A`". But what do these options do? Let's go back to interaction1.txt

```

0

1

2

3

4

5

6

7

8

9

-

> 3

> A

> C

3 A C B D

```

```

>>>

0

1

2

3

4

5

6

7

8

9

-

> 3

> D

> F

3 D F B F

```

Let's look at these two interactions with the server that use option 3. In the first block, the user performs option 3 with input A and C, and the servers gives back output D.

In the second block, the user performs option 3 with input D and F, and the server gives back F, the second input.

Why did 3 give back a new string in one case, and one of the input in the other? Maybe D is some kind of identity element and this is all some kind of algebra? Let's find out.

# Working with the server

We built a wrapper in python to facilitate working with the server, since copying and pasting strings of chinese isn't much fun.

We can call the function `op(OPERATIONNUMBER, PARAMETERS)`, which will return the last string of chinese the server writes (what we think is the operation output).

This way we can easily combine operations with calls like `a = op(0)` and `op(3, a, op(1, a))`. The wrapper also tells us when it encounter as an output something it has seen before, for example if we wrote

```

op(3, b, op(3, op(1, a), a))

```

our script will return

```

0() = a

0() = b

1(a) = c

3(1(a), a) = d

3(b, 3(1(a), a)) = b

!!! 3(b, 3(1(a), a)) = b <-> 0() = b

```

telling us that `op(3, b, op(3, op(1, a), a))` is equal to `b` itself

Next chapter lists all our findings

# Operations

## OP0 [ random number in a pool of 7 ]

> This operation seems to always give one number out of 7 random numbers

---

## OP1 [ -x ]

> This operation acts like the **additive inversion** (with op3 being the addition), probably **modulo N**

### Identities

**<span>[!]</span> mentions operation 3 (operation and identity element)**

```python

# 131aa = 31aa : -((-a)+a) = -0 = 0

assert op(1, op(3, op(1, a), a)) == op(3, op(1, a), a)

```

```python

# 11a = a : -(-a) = a

assert op(1, op(1, a)) == a

```

---

## OP2 [ 1/x ]

> This operation acts like the **multiplicative inversion** (with op4 being the multiplication), probably **modulo N**

### Identities

**<span>[!]</span> mentions operation 4 (operation and identity element)**

```python

# 242aa = 42aa : 1/(a*(1/a)) = 1/1 = 1

assert op(2, op(4, op(2, a), a)) == op(4, op(2, a), a)

```

```python

# 22a = a : 1/(1/a) = a

assert op(2, op(2, a)) == a

```

**<span>[!]</span> mentions operation 3 (operation and identity element)**

```python

# 231aa = 31aa : 1/((-a)+a)) = 1/0 = 0

assert op(2, op(3, op(1, a), a)) == op(3, op(1, a), a)

```

---

## OP3 [ a+b, 0 identity element, -x inverse ]

> This operation acts like the **addition**, probably **modulo N**

### Commutativity

```python

# 3ab = 3ba : (a + b) = (b + a)

assert op(3, a, b) == op(3, b, a)

```

### Associativity

```python

# 3a3bc = 33abc = 3abc : a + (b + c) = (a + b) + c

assert op(3, a, op(3, b, c)) == op(3, op(3, a, b), c)

```

### Identity element

```python

# 3b31aa = b : b + ((-a) + a) = b + 0 = b

assert op(3, b, op(3, op(1, a), a)) == b

# together with other permutations (because of commutativity)

```

### Inverse

```python

# 31a3ab = b : (-a) + (a + b) = b

assert op(3, op(1, a), op(3, a, b)) == b

# together with other permutations (because of commutativity)

```

---

## OP4 [ a*b, 1 identity element, 1/x inverse, 0 absorbing element ]

> This operation acts like the **multiplication**, probably **modulo N**

### Commutativity

```python

# 4ab = 4ba : ab = ba

assert op(4, a, b) == op(4, b, a)

```

### Associativity

```python

# 4a4bc = 44abc = 4abc : a*(b*c) = (a*b)*c

assert op(4, a, op(4, b, c)) == op(4, op(4, a, b), c)

```

### Identity element

```python

# 4b42aa = b : b * (1/a * a) = b * 1 = b

assert op(4, b, op(4, op(2, a), a)) == b

# together with other permutations (because of commutativity)

```

, **<span>[!]</span> mentions operation 5 (operation and identity element)**

```python

# 4b52aa = b : b * (1/a × a) = b

assert op(4, b, op(5, op(2, a), a)) == b

# together with other permutations (because of commutativity)

```

### Inverse

```python

# 42a4ab = b : 1/a * (a * b) = 1 * b = b

assert op(4, op(2, a), op(4, a, b)) == b

# together with other permutations (because of commutativity)

```

### Absorbing element

```python

# 5a3b1b = 3b1b : a * (b + (-b)) = a*0 = 0

assert op(4, a, op(3, b, op(1, b))) == op(3, b, op(1, b))

# together with other permutations (because of commutativity)

```

---

## Notes on OP3 and OP4

### Distributivity of OP4 on OP3

```python

# 4a3bc == 34ab4ac : a * (b + c) = ab + ac

assert op(4, a, op(3, b, c)) == op(3, op(4, a, b), op(4, a, c))

```

### Confirmations on op3 and op4 of being sum and multiplication

```python

# 1 + 1 + 1 + 1 = 4 = (1 + 1) * (1 + 1)

n1 = op(4, op(2, a), a)

assert op(3, n1, op(3, n1, op(3, n1, n1))) == op(4, op(3, n1, n1), op(3, n1, n1))

```

```python

# a * 2 = a + a

assert op(4, a, n2) == op(3, a, a)

```

---

## Notes on OP1, OP2, OP3 and OP4

> Earlier, we said that when we called operation 1, 2, 3, and 4, the output looks like `INPUTS SEPARATOR OUTPUT`, where separator is some string that looks like any other "number". After finding out that these operations are probably in modulo, we hypothesized that `SEPARATOR` is not actually a separator, but the modulo of operation. This seems to be correct since , but we didn't investigate much further.

---

## OP5 [ a × b, 1 identity element, *1/x invers-ish, 0 absorbing element ]

> This was a weird one, this operations acts like op4, except it doesn't in some cases. It gives back the exact same results for small numbers (obtained adding 1, the identity element of op3, a bunch of times), but with numbers given by op0 it acts differently (probably because they are really big).

> It seems like this is **multiplication** with **some other modulo** or maybe with **no modulo**. It probably just has a bigger modulo since we couldn't make it overflow or anything like that.

> This would explain why (using * everything goes back to modulo N), while (1/a is a different number than the inverse of a with respect to op5, since they have a different modulo).

>

> Some other things we didn't write here point in this direction too, like even though .

### Commutativity

```python

# 5ab = 5ba : a × b = b × a

assert op(5, a, b) == op(5, b, a)

```

### Associativity

```python

# 5a5bc = 55abc = 5abc : a × (b × c) = (a × b) × c

assert op(5, a, op(5, b, c)) == op(5, op(5, a, b), c)

```

### Identity element

```python

# 5b4a2a = b : b × (a*1/a) = b × 1 = b

assert op(5, b, op(4, a, op(2, a))) == b

# together with other permutations (because of commutativity)

```

### Absorbing element

```python

5b3a1a = 3a1a : b × (a+(-a)) = b × 0 = 0

assert op(5, b, op(3, a, op(1, a))) == op(3, a, op(1, a))

# together with other permutations (because of commutativity)

```

### Invers-ish

```python

# 42a5ab = b : 1/a * (a × b) = b

assert op(4, op(2, a), op(5, a, b)) == b

# together with other permutations (because of commutativity)

```

```python

# 52a5ab != b : 1/a × a × b != b

assert op(5, op(2, a), op(5, a, b)) != b

```

### Identities

```python

# 452aa52aa = 42aa : (1/a × a) * (1/a × a) = @ * @ = 1

assert op(4, op(5, op(2, a), a), op(5, op(2, a), a)) == op(4, op(2, a), a)

```

```python

# 252aa = 42aa : 1/(1/a × a) = 1/@ = 1

assert op(2, op(5, op(2, a), a)) == op(4, op(2, a), a)

```

```python

# 152aa = 142aa : -(1/a × a) = -@ != -1 = -(1/a * a)

assert op(1, op(5, op(2, a), a)) == op(1, op(4, op(2, a), a))

```

```python

# 52aa != 42aa : (1/a × a) = @ != 1 = (1/a * a)

assert op(5, op(2, a), a) != op(4, op(2, a), a)

```

---

## OP6 [ a^b, 1 right identity element, 1 and 0 left absorbing elements ]

> This operation acts like **exponentiation** with **no modulo** or **with the same modulo as op5**. It behaves well with op5, exponentiation rules work with it, while they often don't work with op4 (unless we're using small numbers)

>

### NO Commutativity

```python

#6ab != 6ba : a ! b != b ! a

assert op(6, a, b) != op(6, b, a)

```

### NO Associativity

```python

# 6a6bc != 66abc : a ! (b ! c) != (a ! b) ! c

assert op(6, a, op(6, b, c)) != op(6, op(6, a, b), c)

```

### Right identity element

```python

# 6b42aa = b : b ! (1/a * a) = b ! 1 = b

assert op(6, b, op(4, op(2, a), a)) == b

```

### Left absorbing elements

```python

# 642aab = 42aa : (1/a * a) ! b = 1 ! b = 1

assert op(6, op(4, op(2, a), a), b) == op(4, op(2, a), a)

```

```python

# 631aab = 31aa : ((-a) + a) ! b = 0 ! b = 0

assert op(6, op(3, op(1, a), a), b) == op(3, op(1, a), a)

```

### Identities

```python

# 6a31aa = 42aa : a ! ((-a) + a) = a ! 0 = 1 = (1/a * a)

assert op(6, b, op(3, op(1, a), a)) == op(4, op(2, a), a)

```

---

## Notes on OP6

### Confirmations and doubts on exponentiation

```python

# 3 ^ 2 = 3 * 3

assert op(6, n3, n2) == op(4, n3, n3)

# a ^ 2 != a * a

assert op(6, b, n2) != op(4, b, b)

# a ^ 2 = a × a

assert op(6, b, n2) != op(4, b, b)

```

```python

# (2 ^ 3) ^ 4 = 2 ^ (3*4)

assert op(6, op(6, n2, n3), n4) == op(6, n2, op(4, n3, n4))

# (a ^ b) ^ c != a ^ (b*c)

assert op(6, op(6, a, b), c) != op(6, a, op(4, b, c))

# (a ^ b) ^ c = a ^ (b × c)

assert op(6, op(6, a, b), c) == op(6, a, op(5, b, c))

```

```python

# a ^ -1 != 1/a

assert op(6, a, op(1, n1)) != op(2, a)

```

```python

# a^b × a^c != a^(b + c)

assert op(5, op(6, a, b), op(6, a, c)) != op(6, a, op(5, b, c))

```

```python

# a^c × b^c = (a × b)^c

assert op(5, op(6, a, c), op(6, b, c)) == op(6, op(5, a, b), c)

```

## Notes on OP5 and OP6

> OP5 and OP6 also have a "separator", but it's different from the separator of 1 2 3 4. This reinforces the theory that op5 and op6 are also in modulo, but with a different modulo, which is given in every op5 and op6 output.

## OP7

> We don't know much about op7, it asks for one input, and performs some operations of which we don't know the operands (we only know the operators).

---

## OP8

> This operator does exactly what 9 does, but it has 3 inputs: A B C, as opposed to 2 (one of the 3 inputs of op8 corresponds to one of the constants of op9). It then does the same operations as op9, but with some numbers permuted: `(A,B,C)[op9] = (B,C,A)[op8]`.

---

## OP9

> Here is where the magic happens, as we said earlier, op9 does a bunch of operations on two inputs A and B, and then gives a result Q. If Q == A then it also gives us the flag.

```python

>>>

0

1

2

3

4

5

6

7

8

9

-

> 9

> A

> B

2 B (modN) 2B

4 C 2B (modN) 4C2B

4 A 2B (modN) 4A2B

6 D 4C2B (modM) 6D4C2B

6 E 4A2B (modM) 6E4A2B

5 6D4C2B 6E4A2B (modM) 56D4C2B6E4A2B

0 F

1 F = 1F

3 56D4C2B6E4A2B F (modN) 356D4C2B6E4A2BF

3 356D4C2B6E4A2BF 1F (modN) A

9 C A B A

```

> Our goal is to find A and B for the equation `A = 3356D4C2B6E4A2BF1F` (prefix notation), or, using the symbols we defined earlier:

>

>

>

> We can rewrite this (because of associativity of + and because -F is the inverse of F) as

>

>

>

> Now, we can rewrite this (thanks to the property of the exponentiation) as

>

>

>

> This pass might seem wrong because the operator * is used inside the exponent, and not the operator ×. However, because in the original equation, we add and subtract F, the result is modulo , so everything should work properly.

> Next, thanks to another property of exponentiation, we can rewrite as

>

>

>

> At this point, we can either continue trying to solve the equation for A or B, or try some numbers. We lost quite some time thinking that maybe we should find the modulo of op6, and we tried some strategies, but we realized that the modulo changed every time, and that it was also probably not the correct way to solve the challenge. So, we tried some numbers.

>

> Since , and , forcing makes the . We can now force , which makes the , and the equation is solved.

# Getting the flag

Now that we solved the equation, we can just do

```python

a = op(0)

n0 = op(3, a, op(1, a))

n1 = op(4, a, op(2, a))

op(9, n1, n0)

```

And the server returns us the flag: `OOO{D0Y0uUnd3rstandWh4tANumberIs?}`
