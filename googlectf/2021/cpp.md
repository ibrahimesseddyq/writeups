# Cpp

**CTF:** GoogleCTF 2021  
**Category:** rev  
**Original writeup:** <https://www.cis.upenn.edu/~hanbangw/blog/google-ctf-2021/#cpp> — by **cis.upenn.edu**. All credit goes to the original author; this is an archived copy.

---

# CPP

![cpp.c](https://www.cis.upenn.edu/~hanbangw/assets/images/google-ctf-2021/cpp1.png)

So *CPP* stands for *C Pre-Processor*, clearly seen from the compiler's warning message `-Wcpp`.

## Eyeballing the Code

Open the file we see a bunch of pre-processing macros. In fact, most of the code are macros, and if we scroll to the bottom we can see a tiny bit of actual C code that will compile if the pre-processing passed without errors.

It is pretty obvious that we need to somehow figure out the running process of the pre-processor, and the flag we are looking for is hidden within.

Going back to the top of the file (line 16), we first see a list of definition of flag characters from `FLAG_0` to `FLAG_26`, in total of 27 characters. It's then followed by a list of definition of characters used in the flag string (line 45), which includes all 26 English letters, in both lowercase and uppercase, and 10 numeral digits, plus underscore `_` and brackets `{` and `}`, all defined to be their ASCII values. In total we have 65 characters possibly be used in the flag string. The number of combinations for all possible flags is $$65^{27}$$, which is apparently impossible to brute-force.

The next section is a list of definitions (line 111), including a variable `S` and bunch of variables starts with `ROM_`. Without any further context, we can assume that this is the part where the memory is defined for this program, where `ROM_xxxxxxxx_y` means the `y`th bit of the address `0bxxxxxxxx`. The pre-defined memory values lies within the range of `0b00000000 - 0b01111111` (0x0 - 0x7F), and the flag string is stored in `0b10000000 - 0b10011010` (0x80 - 0x9A).

It is also from this part (line 840) we can tell that our assumption above is correct. Furthermore, we can see that the `0`th bit of each address in `ROM` is the least-significant bit, and the `7`th is the most-significant one. A code snippet like this

```c

#if FLAG_0 & (1<<2)

#define ROM_10000000_2 1

#else

#define ROM_10000000_2 0

#endif

```

checks the second bit of `FLAG_0` and store the value into `ROM_10000000_2`.

The next five lines (line 1920-1924) defines some macro functions. We can see that function `LD(x, y)` is the same as `ROM_x_y`, meaning that this `LD` loads the `y`th bit from address `x` in `ROM`. The function `MA(l0, l1, l2, l3, l4, l5, l6, l7)` concatenates bit `l0` to `l7` together, but in reverse order, meaning `MA(1, 1, 1, 1, 0, 0, 0, 1)` will give out string `10001111`. Here, we can't be sure that `l0` to `l7` are 0, 1 values only yet, but it will become apparent in the following analysis. The final macro `l` is simply a short hand to the above `MA` function called on `l0` to `l7`.

### Code Formatting

The next part starting from line 1926 is very messy, mainly there's a lot of `#if...` instructions without proper indentation, make it really hard to read. We wrote a easy formatter:

```python

with open('cpp.c', 'r') as in_file, \

open('cpp.formatted.c', 'w') as out_file:

indent = ''

for line in in_file:

if line.startswith('#if'):

print(indent + line, end='', file=out_file)

indent += ' '

elif line.startswith('#else'):

print(indent[:-2] + line, end='', file=out_file)

else:

if line.startswith('#endif'):

indent = indent[:-2]

print(indent + line, end='', file=out_file)

```

The end result looks like this

```c

#if S == 3

#undef S

#define S 4

#undef c

#ifndef R0

#ifndef Z0

#ifdef c

#define R0

#undef c

#endif

#else

#ifndef c

#define R0

[...]

```

Which is much more intelligible.

### Structure Overview

This above part looks like the main program, so we'll skip it for now. Jumping all the way to the bottom (line 6217), we see that the code includes itself twice if `S != -1`. We also see that there's a pre-defined macro `__INCLUDE_LEVEL__` used. It is a macro that starts at 0, and increase by 1 for each level an `#include` is expanded. This means the code expands differently at different include level.

Overall structure of the file can be seen as:

```

if (__INCLUDE_LEVEL__ == 0) {

flag_str := "CTF{write_flag_here_please}"

/* define character ascii value */



MEMORY[0x0 - 0x7F] := {...}



copy(&MEMORY[0x80], flag_str)

}

if (__INCLUDE_LEVEL__ > 12) {

// main program

} else {

if (S != -1) {

#include self

}

if (S != -1) {

#include self

}

}

```

## Reversing the Program

For the main program (line 1927 - 6215), we can see a pattern that looks like this:

```c

#if S == [x]

#undef S

#define S [x+1]

[...]

#endif

#if S == [x+1]

#undef S

#define S [x+2]

[...]

```

where `x` ranges from 0 to 58. Experiences tell us that `S` should be the instruction pointer, and it by defaults go to the next one. However, the preprocessor only goes in one direction, so how does this program `jmp`? Or in other words, what happens if the program `#define S [x]` where `x` is less than the current `S` value?

This is where that two `#include <cpp.c>` comes into play. The code include itself twice when `__INCLUDE_LEVEL__` is less than 12 and `S != -1`. From there we know two things,

1. Program ends when `S`, the instruction pointer, `== -1`

2. The program *jmp* by setting `S` and executes the corresponding instruction in the next `#include` part of the same code, if the new `S` is less than the current `S`.

---

Since `S`'s initial value is `0`, we followed the execution path and tried to manually figure out what each instruction means:

`S == 0`: A single `#define S 24` means `JMP 24`.

`S == 24`: What's going on? Why there's a lot of `#undef`?

#### S == 30

Ignoring the confusion, I followed the path all the way to `S == 30` and see a humongous body for this line of instruction.

> I originally thought this was somehow a form of obfuscation, so I wrote a static analyzer for this program and see if I can rip away some of the things. Spent a lot of time on this, later realizing it was heading towards a wrong direction.

We see something look like this:

```c

#ifndef Bx

#ifndef Ix

#ifdef c

#define Bx

#undef c

#endif

#else

#ifndef c

#define Bx

#undef c

#endif

#endif

#else

#ifndef Ix

#ifdef c

#undef Bx

#define c

#endif

#else

#ifndef c

#undef Bx

#define c

#endif

#endif

#endif

```

for `x` ranging 0 to 7. Let's make it clearer by constructing a table:

| Bx | Ix | c | Bx_after | c_after |

| :--: | :--: | :--: | :------: | :-----: |

| N | N | N | N | N |

| N | N | Y | **Y** | **N** |

| N | Y | N | **Y** | **N** |

| N | Y | Y | N | Y |

| Y | N | N | Y | N |

| Y | N | Y | **N** | **Y** |

| Y | Y | N | **N** | **Y** |

| Y | Y | Y | Y | Y |

Where `N` means that the variable is undefined and `Y` means it is defined. `Bx_after` is the result of `Bx` after running this instruction. Notice that 4 rows are marked **bold** for `Bx_after` and `c_after`, meaning their value changed. For all other initial value of `Bx`, `Ix` and `c`, they matched none of the branches, so their value doesn't change.

It is then clear that this is `B = B + I` in binary, where `c` is the carry bit. The only part that is confusing is actually how a bit is represented.

### #define and #undef a Bit

When we first analyze this code, all values are set using `#define [variable] [value]`. This applies to the `flag` value, the `MEM` section and `S` the instruction pointer. However, things changed a bit in the main program. Here, a bit is set or unset using `#define` and `#undef`, and checked using `#ifdef` and `#ifndef` respectively—the existence of the macro defines if the bit is 1 or 0. So, for a code snippet like this:

```

#define B0

#undef B1

#define B2

#undef B3

#undef B4

#define B5

#define B6

#define B7

```

If we consider `B` as a signed 8-bit number, then it is equivalent of setting `B = 0b11100101 (-27)`.

With this knowledge, we can finally start out reversing process.

---

Continue where we left off:

24: `I = 0`

25: `M = 0`

26: `N = 1`

27: `P = 0`

28: `Q = 0`

29: `B = -27`

30: `B = B + I` as we just analyzed.

#### S == 31

```

#ifndef B0

#ifndef B1

#ifndef B2

#ifndef B3

#ifndef B4

#ifndef B5

#ifndef B6

#ifndef B7

#undef S

#define S 56

#endif

#endif

#endif

#endif

#endif

#endif

#endif

#endif

```

By our analysis above, this means that if `B == 0`, we will jump to instruction 56.

Therefore, instruction 31 is `IF B == 0 THEN JMP 56`.

---

32: `B = 0x80`

33: `B = B + I`, same as instruction 30

#### S == 34

There are two parts in this instruction, let's check them out one by one:

```

#undef lx

#ifdef Bx

#define lx 1

#else

#define lx 0

#endif

```

for `x` ranging 0 to 7. It is easy to recognize that this is checking each bit of `B`, and set `l_x` to the literal value of 1 or 0.

The next part looks like this:

```c

#if LD(l, x)

#define Ax

#else

#undef Ax

#endif

```

for `x` ranging 0 to 7. As we have analyzed before, `LD(l, x)` is the function to load `MEM` portion using address in `l` and the `x`th bit. The `#if` is to convert literal 0 and 1 in memory back to the bit representation (defined or undefined) in the program.

Take the above together, we see that this is a memory load operation, where it takes `B` as a memory address and set the resulting value to `A`. Therefore, instruction 34 is `A = LOAD(B)`.

---

35: `B = LOAD(I)`, similar to instruction 34.

36: `R = 1`

37: `JMP 12`

12: `X = 1`

13: `Y = 0`

14: `IF X == 0 THEN JMP 22`, similar to instruction 31.

#### S == 15

```c

#ifdef Xx

#define Zx

#else

#undef Zx

#endif

```

for `x` ranging 0 to 7. It is easy to recognize that this is *copying* or assigning each bit of `X` to `Z`, so this is an equal operation. Instruction 15 is `Z = X`.

#### S == 16

```

#ifdef Zx

#ifndef Bx

#undef Zx

#endif

#endif

```

for `x` ranging 0 to 7. From the syntax of it, we can see that `Zx` will be 0 when `Bx` is 0.

> You can draw out a table for this, but I'll cut to the chase...

Which means that this instruction is a bitwise-and operation, `Z = Z & B`.

---

17: `IF Z == 0 THEN JMP 19`

18: `Y = Y + A`

19: `X = X + X`

20: `A = A + A`

21: `JMP 14`

22: `A = Y`

23: `JMP 1`

#### S == 1

```

#ifdef Rx

#undef Rx

#else

#define Rx

#endif

```

Should be pretty obvious that this is a bitwise-not, `R = ~R`.

---

2: `Z = 1`

3: `R += Z`

4: `R += Z`

5: `IF R == 0 THEN JMP 38`

6: `R += Z`

7: `IF R == 0 THEN JMP 59`

8: `R += Z`

9: `IF R == 0 THEN JMP 59`

10: `#ERROR BUG`

11: `EXIT`, as we talked about `S == -1` means ending the program.

38: `O = M`

39: `O += N`

40: `M = N`

41: `N = O`

42: `A = A + M`

43: `B = 0x20`

44: `B += I`

45: `C = LOAD(B)`

#### S == 46

```c

#ifdef Cx

#ifdef Ax

#undef Ax

#else

#define Ax

#endif

#endif

```

Let's draw a table for this:

| Cx | Ax | Ax after |

| :--: | :--: | :------: |

| 0 | 0 | 0 |

| 0 | 1 | 1 |

| 1 | 0 | **1** |

| 1 | 1 | **0** |

The remaining case when `Cx` is not set, `Ax` will keep unchanged. So we can say that

$$

A_x = \begin{cases}

A_x & \text{if}\ C_x=0 \\



eg A_x & \text{if}\ C_x=1 \\

\end{cases} \implies A_x = A_x \oplus C_x

$$

Meaning instruction 46 is exclusive-or operation, `A = A ^ C`.

---

47: `P = P + A`

48: `B = 0x40`

49: `B = B + I`

50: `A = LOAD(B)`

51: `A = A ^ P`, similar to instruction 46

#### S == 52

```c

#ifndef Qx

#ifdef Ax

#define Qx

#endif

#endif

```

Very similar to instruction 16, but this time we can see `Qx` will be 1 when `Ax` is 1, and otherwise unaffected.

> Again you can draw out a table for this.

Which means that this instruction is a bitwise-or operation, `Q = Q | A`.

---

53: `A = 1`

54: `I = I + A`

55: `JMP 29`

56: `IF Q == 0 JMP 58`

57: `#ERROR "INVALID FLAG"`

58: `EXIT`

### Rewrite the Program

Since we now have analyzed every single instruction of the program, let's write a pseudo program for this:

```c

I = 0 // 24

M = 0 // 25

N = 1 // 26

P = 0 // 27

Q = 0 // 28

for {

B = -27 // 29

B = B + I // 30

if (B == 0) break; // 31

B = 0x80 // 32

B = B + I // 33

A = MEMORY[B] // 34

B = MEMORY[I] // 35

R = 1 // 36

X = 1 // 12

Y = 0 // 13

for X != 0 { // 14

Z = X // 15

Z = Z & B // 16

if (Z != 0) { // 17

Y += A // 18

} // 19

X += X // 20

A += A // 21

}

A = Y // 22



R = ~R // 1

Z = 1 // 2

R = R + Z // 3

R = R + Z // 4

if (R != 0) abort() // 5 - 11 (won't reach here)



O = M // 38

O = O + N // 39

M = N // 40

N = O // 41

A = A + M // 42

B = 0x20 // 43

B = B + I // 44

C = LOAD(B) // 45

A = A ^ C // 46

P = P + A // 47

B = 0x40 // 48

B = B + I // 49

A = LOAD(B) // 50

A = A ^ P // 51

Q = Q | A // 52

A = 1 // 53

I = I + A // 54

}

if (Q != 0) { // 56

"INVALID FLAG" // 57

}

EXIT // 58

```

With some tidy up, and write it in Go, we get

```go

var M uint8 = 0

var N uint8 = 1

var P uint8 = 0

var Q uint8 = 0

for I := uint8(0); I < 27; I++ {

A := MEMORY(0x80 + I)

B := MEMORY(I)

var X uint8 = 1

var Y uint8 = 0

for X != 0 {

if X&B != 0 {

Y += A

}

X += X

A += A

}

A = Y

O := M + N

M = N

N = O

A += M

A ^= MEMORY(0x20 + I)

P += A

Q |= MEMORY(0x40+I) ^ P

}

if Q != 0 {

fmt.Println("invalid flag")

}

```

Notice that `R` is gone. Since `~1 + 2 == 0` is for sure, we can optimize it out. Some of the intermediate variables are also optimized out.

## Get the Flag

It is possible to figure out the logic behind the memory operations and try to extract the flag by reversing the process. However, with some observation, we see that the flag is identified as invalid when `Q != 0` when the program ends. Looking at the entire program, `Q` is only calculated once here `Q |= [some value]`. By the property of the bitwise-or, any set bit will remain to be set. Therefore, in order for `Q == 0` at the end, it must be that for each iteration of the loop, `Q` is kept at zero.

Also, the program processes flag string one byte by one byte, which means that, if at any iteration of the loop, `Q` is some value not `0`, then that byte must be a faulty byte.

Using this knowledge, we are able to reduce the search space from $$65^{27}$$ to $$65 \times 27$$. The exploit then is to test out the string character one-by-one, and continue if `Q` is kept 0 during the loop.

![cpp.go](https://www.cis.upenn.edu/~hanbangw/assets/images/google-ctf-2021/cpp2.png)

The exploit can be found here: [CPP, Google CTF 2021](https://gist.github.com/superfashi/f264b9c9f19b8b25065afd9f66bf7ffd){:target="_blank"}.
