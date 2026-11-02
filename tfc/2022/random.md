# RANDOM

**CTF:** TFC 2022  
**Category:** pwn  
**Original writeup:** <https://blog.hamayanhamayan.com/entry/2022/07/31/213142> — by **blog.hamayanhamayan.com**. All credit goes to the original author; this is an archived copy.

---

[pwn] RANDOM

$ file random random: ELF 64-bit LSB pie executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, BuildID[sha1]=defd9a2d9f6d0a4ec5067bc2ed810ee1444ea52c, for GNU/Linux 3.2.0, not stripped $ checksec random [*] '/mnt/c/Users/eric/Downloads/TFC CTF/pwn-random/random' Arch: amd64-64-little RELRO: Full RELRO Stack: Canary found NX: NX enabled PIE: PIE enabled

ふむ…ghidraでCに戻してみる。

puts ( "Menu:

1. Generate number" ); __isoc99_scanf (&DAT_0010201e,&local_14); if (local_14 == 1 ) { uVar1 = rand (); printf ( " %d " ,(ulong)uVar1); } else if (local_14 == 0x539 ) { pcVar3 = getenv ( "FLAG" ); printf ( " %s " ,pcVar3); }

こういう感じなので、入力に0x539、つまり1337を入れればフラグが手に入る。

$ nc 01.linux.challenges.ctf.thefewchosen.com 51657 Menu: 1. Generate number 1337 TFCCTF{Th3r3_w3r3_m0r3_0pt10n5_4ft3r_4ll!}
