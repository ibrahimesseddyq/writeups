# easy-shell

**CTF:** nullconHackIM 2019  
**Category:** misc  
**Original writeup:** <https://lordidiot.github.io/2019-02-03/nullcon-hackim-ctf-2019/#easy-shell> — by **lordidiot**. All credit goes to the original author; this is an archived copy.

---

easy-shell

Go get yourself a shell while it’s possible nc pwn.ctf.nullcon.net 4010 challenge +exploit.py

Disclaimer: I was not the one who solved this during the CTF, my teammate Engimatrix solved this instead. However, I was working on a separate solution from him in parallel to have a better idea for when we want to solve the next part of the challenge

Overview

In short, we are given a binary that mmaps an RWX region, reads shellcode from the user into this region, then jumps to the shellcode. Pretty straightforward. However, there are 2 restrictions we have to bypass. Firstly, the shellcode we pass must be alphanumeric, in regex form: [a-zA-Z0-9] . Additionally, there is a seccomp rule implemented which prevents the execve syscall.

line CODE JT JF K ================================= 0000: 0x20 0x00 0x00 0x00000004 A = arch 0001: 0x15 0x00 0x03 0xc000003e if (A != ARCH_X86_64) goto 0005 0002: 0x20 0x00 0x00 0x00000000 A = sys_number 0003: 0x15 0x01 0x00 0x0000003b if (A == execve) goto 0005 0004: 0x06 0x00 0x00 0x7fff0000 return ALLOW 0005: 0x06 0x00 0x00 0x00051234 return ERRNO(4660)

Approach

Just with any other shellcoding challenge, the first thing to do it search exploit-db to check if the relevant shellcode already exists. So simply google “x64 alphanumeric shellcode”. There are a few samples but they all use execve to pop a shell, in our case, we would either need to use execveat or use a open-read-write(orw) shellcode instead. Since I could not find any such existing shellcodes, it is time to start writing our own shellcode.

With the strict restriction of only alphanumeric shellcode, my approach is always to figure out how to perform a read shellcode from user input into the RWX region. This is commonly referred to as a multi-stage attack, and I think it is the most effective approach to restrictive shellcoding challenges. Once we are able to perform a read from user input into a RWX region, we can then write a second stage payload without any restrictions! It is far easier to setup the read shellcode as opposed to the entire open-read-write chain in alphanumeric.

An important factor to consider when doing shellcoding is the context in which our shellcode runs. By this, I’m referring to the current state of the registers and memory before the first instruction from our shellcode is even executed. This is very important as it enables us to do a lot more, and reduce a lot of unnecessary instructions. To find out the state of the registers and memory, we can simply set a breakpoint using pie break * 0xb94 in gef, and checkout the memory.

$rax : 0x0 $rbx : 0x4000 $rcx : 0x00007f26a1c82260 → <__read_nocancel+7> cmp rax, 0xfffffffffffff001 $rdx : 0x56 $rsp : 0x00007fffa9d69ce8 → 0x00005652ce7e9b97 → jmp 0x5652ce7e9b4c $rbp : 0x0 $rsi : 0x00007f26a2176000 → "Our shellcode!" $rdi : 0x0 $rip : 0x00007f26a2176000 → "Our shellcode!" $r8 : 0x00007f26a1f51780 → 0x0000000000000000 $r9 : 0x00007f26a2155700 → 0x00007f26a2155700 → [loop detected] $r10 : 0x22 $r11 : 0x246 $r12 : 0x00007f26a2176000 → "Our shellcode!" $r13 : 0x00007fffa9d69de0 → 0x0000000000000001 $r14 : 0x0 $r15 : 0x0

The most important registers we should look at are generally rax, rdi, rsi, rdx when in x64. These registers are used for the syscalls and the arguments to function calls, thus if they are already set to a good value, a lot of our work is already done for us. And indeeed, the registers are actually setup in a perfect manner for us to do a read syscall into the RWX region as we’d planned to do. Now all we need to do is provide a syscall instruction, and we can read in our second stage payload! Unfortunately, the syscall instruction uses the bytes \x0f\x05 , which are not in our alphanumeric charset, how can we bypass this?

Self-modifying shellcode

As the header suggests, we can create our syscall instruction in the shellcode, by making our shellcode modify itself! This is a useful technique whenever you are not able to write certain bytes. Additionally, since all our instructions are present in RWX regions, there is nothing stopping us from overwriting the instructions. After doing some research, I find that the following instructions only require alphanumeric characters to represent.

xor DWORD PTR [rcx + imm8], eax push [any reg] push imm32 pop rax pop rcx

With these instructions, do you see how we can perform our self-modifiying shellcode? Since we know that we can push any registers, we can notice from above that rsi, rip, r12 all contain a pointer to our shellcode. Thus we can simply push this value onto the stack, then pop it off into rcx . Now using the push imm32 followed by a pop rax , we are able to get any 4 byte value (within alphanumeric range) into eax. Now we are in a perfect setup to use the xor DWORD PTR [rcx + imm8], eax to xor some instructions in our own shellcode! Now all we have to do is to find pairs of alphanumeric characters that xor to form \x0f and \x05 . Just open up your python interpreter and do a quick bruteforce.

charset = [ '0' , '1' , '2' , '3' , '4' , '5' , '6' , '7' , '8' , '9' , 'A' , 'B' , 'C' , 'D' , 'E' , 'F' , 'G' , 'H' , 'I' , 'J' , 'K' , 'L' , 'M' , 'N' , 'O' , 'P' , 'Q' , 'R' , 'S' , 'T' , 'U' , 'V' , 'W' , 'X' , 'Y' , 'Z' , 'a' , 'b' , 'c' , 'd' , 'e' , 'f' , 'g' , 'h' , 'i' , 'j' , 'k' , 'l' , 'm' , 'n' , 'o' , 'p' , 'q' , 'r' , 's' , 't' , 'u' , 'v' , 'w' , 'x' , 'y' , 'z' ] for i in charset : for j in charset : if ord ( i ) ^ ord ( j ) == 0xf : print "{:x}, {:x}: {:x}" . format ( ord ( i ), ord ( j ), ord ( i ) ^ ord ( j ))

All we have to do is make sure that imm8 and imm32 are only using alphanumeric bytes and we’re good to go with forming our syscall instruction. Also, since we are forced to use an alphanumeric imm8, our syscall instruction is a bit far from the start our shellcode, so we need to use some nop instructions to reach that syscall instruction. Since the actual nop instruction is \x90 (not alphanumeric), we can use a two byte nop, like push rax; pop rax . This achieves nothing and helps us travel down the shellcode to our syscall.

After this, our shellcode will hit the syscall , and read some input from the user. We simply provide an open-read-write shellcode that will open “flag” and print it. I will not go into detail on this and you can check my exploit source code for details.

hackim19{to_read_or_not_to_r34d}
