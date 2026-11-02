# GuardiansOfTheGalaxy

**CTF:** SpaceHeroes 2022  
**Category:** pwn  
**Original writeup:** <https://github.com/BrennenWright/CaptureTheFlag-Writeups/blob/main/2022/SpaceHeroesCTF/GuardiansOfTheGalaxy/README.md> — by **BrennenWright**. All credit goes to the original author; this is an archived copy.

---

# Guardians of the Galaxy

## The Problem

Points: 100

Rating:

Author: GlitchArchetype

Flavor Text:

```

Ronan the Accuser has the Power Stone. Can Starlord find a successful distraction format? (please note that you should only contact the admin if you receive the error while using the command below)

nc 0.cloud.chals.io 12690

```

Attachments : [https://github.com/BrennenWright/CaptureTheFlag-Writeups/blob/main/2022/SpaceHeroesCTF/GuardiansOfTheGalaxy/guardians.out](guardians.out)

## Solution

Downloaded the source file. ran

```

cat guardians.out

```

and

```

strings guardians.out

```

nothing but ./flag jumped out at me from the file.

it looks to be a simple c program that works with printing the output

it responds with "OH no..." each time you enter an input and follows it up with a copy of your input.

the flavor text mentioned a "format" for the distraction. this is likely a input format vuln where its response can be messed with to provide alternate response instead of the text intended.

to the google

looked up C program print format vuls and found a few things such as:

```

Since printf has a variable number of arguments, it must use the format string to determine the number of arguments. In the case above, the attacker can pass the string “%p %p %p %p %p %p %p %p %p %p %p %p %p %p %p” and fool the printf into thinking it has 15 arguments. It will naively print the next 15 addresses on the stack, thinking they are its arguments.

At about 10 arguments up the stack, we can see a repeating pattern of 0x252070 – those are our %ps on the stack! We start our string with AAAA to see this more explicitly - https://www.geeksforgeeks.org/format-string-vulnerability-and-prevention-with-example/

```

used that to play with %ps and ended up with %p %p %p %p %p %p before the ps started showing up in the printout they look like:

>0x252070

in memory

so I swapped the last p for an %s to output it as a string and got:

## Flag

```

shctf{im_distracting_you}

```
