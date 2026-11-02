# MagicTheGatheRIIng

**CTF:** SunshineCTF 2022  
**Category:** pwn  
**Original writeup:** <https://github.com/nobodyisnobody/write-ups/tree/main/SunshineCTF.2022/pwn/Magic.the.GatheRIIng/> — by **nobodyisnobody**. All credit goes to the original author; this is an archived copy.

---

was a pwn challenge from SunshineCTF 2022.

the program is a pseudo "magic the gathering" card game.

let's check binary protections:

![checksec](https://github.com/nobodyisnobody/write-ups/raw/main/SunshineCTF.2022/pwn/Magic.the.GatheRIIng/pics/checksec.png)

ok so no PIE.

well the program present us a menu:

![menu](https://github.com/nobodyisnobody/write-ups/raw/main/SunshineCTF.2022/pwn/Magic.the.GatheRIIng/pics/menu.png)

you can print rules, load cards, play the game (slow of fast), and exits..

well the vulnerability lies in the function at address 0x40153f:

![reverse](https://github.com/nobodyisnobody/write-ups/raw/main/SunshineCTF.2022/pwn/Magic.the.GatheRIIng/pics/reverse.png)

if you look at the reverse produce by IDA,

you can see that the scanf function

>`__isoc99_sscanf(a2, "%hhd %c %s %d %d %d", &v4, &v5, v10, &v6, &v7, &v8)`

initialize various variables, the `v4` variable is used to calculate the `v9` pointer a bit after

>`v9 = 24LL * v4 + a1;`

then this `v9` pointer is used to write the other values `v6`, `v7` and `v8` in particular.

the `v4` index is not checked for any bounds, and as `v9` is based on `a1` variable that points on stack,

we basically have an out of bounds write on stack.

We can only control `v6`, `v7`, and `v8` values , so a qword and a dword,..



we can only load cards two times, so we can only write two really controlled values on stack..

**So here is the plan**

* **First we will try to have a libc leak**

* **Second we will overwrite main return address with a one gadget**

When playing the game, a string "Basic Troll" or "Basic Land" is printed, via pointers to this strings..

Luckily for us, one of this pointer is reachable with our oob write on stack , the "Basic Land" string pointer.

So we will overwrite this pointer, with the `puts()` function GOT address

Then we play a game , and when the game will try to print "Basic Land", it will leak `puts()` function libc address..

with this leak we will calculate libc base address.

Then with our second oob write, we overwrite `main()` return address with a onegadget..

And when we will exit, we will have a shell...

and that's All..
