# xorop

**CTF:** DamCTF 2020  
**Category:** misc  
**Original writeup:** <https://ctftime.org/writeup/24132> — by **ctftime.org**. All credit goes to the original author; this is an archived copy.

---

the key is generated with a dumb algorithm (srand(time(0)) and some xor..

it depends on time so..

the server and the attacker must have the same date to generate the good key

this key is added to libc addresses on .bss (got) to make them unusable...

the idea, is to calculate the negative key + the offset from puts address in libc to a onegadget

then we return back to the function which encode the libc addresses to cancel the libc addresses modifications on .bss

then we just call puts, which will contain the onegadget address by now

the key was stored at address 0x600c38 on .bss,

we modify it with a xor gadget (gadget3)

it takes 2 o 3 tries to work sometimes...no more..

stdin is closed by the program, and will need to be redirected to fd number 3, that's why we do "/bin/sh 1>&3" first,

to see the output of commands

brought to you, by nobodyisnobody , pwner for **RootMeUpBeforeYouGoGo**
