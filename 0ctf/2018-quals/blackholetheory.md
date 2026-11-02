# BlackHoleTheory

**CTF:** 0CTF 2018 Quals  
**Category:** pwn  
**Original writeup:** <http://gmiru.com/writeups/octf-bh/> — by **gmiru.com**. All credit goes to the original author; this is an archived copy.

---

0ctf 2018 - Black Hole Theory

The challenge is a simple binary that first sets a few seccomp rules to disable all the syscalls besides open, close, read, mprotect and exit. Then it proceeds to read 0x100 bytes into a stack variable that serves as an obvious stack overrun. To further complicate issues the binary is executed by a wrapper, that reads 0x800 bytes and passes it to the program on its stdin, in one burst, while it also closes the stdout pipe.

The binary is not compiled as a PIC and full-relro is not turned on, this enables the use of the return to CSU primitive (also suggested by the hint). The primitive allows us to call a pointer at an arbitrary location in the address space with the first three parameters controlled. Since there is no way to leak addresses from the binary (stdout closed, no write syscall) we must use that is already there. This very simple binary only uses two libc functions (alarm and read), the address of which are present in the .got.plt section, at a known location.

With these primitives the following exploit flow seems possible:

Use the CSU primitive to call the read libc function and use it to overwrite the LSB of the alarm function pointer

libc function and use it to overwrite the LSB of the function pointer By setting the alarm address 5 bytes higher, the alarm function is turned into an arbitrary syscall primitive (the set rax part is skipped)

address 5 bytes higher, the function is turned into an arbitrary syscall primitive (the set part is skipped) Since rax cannot be controlled by ROP gadgets the return value of read must be used to set it. Read 10 bytes to somewhere to set rax to 10 which is mprotect ’s syscall nr.

cannot be controlled by ROP gadgets the return value of read must be used to set it. Read 10 bytes to somewhere to set to 10 which is ’s syscall nr. Call mprotect to turn the .bss into an rwx region

to turn the into an region Read an arbitrary shell code to this new rwx region and jump to it

region and jump to it The shell code can read the flag and use a timing channel to leak its bits. In each execution it can crash and close the connection or go into an infinite loop based on the next bit of the flag. The difference in the time of the connection interrupt can be detected at receiver side, thus the flag can be reconstructed.

There is only a minor technical challenge while executing this plan. The size of the original payload is limited to 0x100-40 bytes which is enough to call the CSU primitive only three times. This can be alleviated by breaking up the first part of the exploit into multiple stages. Alternatively, with a bit of optimisation three calls can be enough to read and execute the shellcode:

The original read to overwrite the LSB of alarm can be 10 bytes long, setting up for the mprotect call.

to overwrite the LSB of can be 10 bytes long, setting up for the mprotect call. Call mprotect

Call read , to read the address of the shellcode right after the read pointer in .got.plt and read the shell code as well.

If the rbp is set to more than one during the CSU call, the primitive consecutively executes multiple functions from the provided location. Using this the third CSU primitive will execute the shellcode that it read in. At this point the .bss would have the following layout:

random .got.plt entries

entries AAAAAAAAA - from using 10 bytes to overwrite the alarm LSB

LSB modified address of alarm

address of read

address of the shell code (next address)

the shell code

This exploit yields the following flag (after considerable time):

flag{even_black_holes_leak_information_by_Hawking_radiation}
