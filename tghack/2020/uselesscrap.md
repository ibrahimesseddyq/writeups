# UselessCrap

**CTF:** TGHACK 2020  
**Category:** misc  
**Original writeup:** <https://pwn-diaries.com/post/tghack-2020-useless-crap/> — by **pwn-diaries.com**. All credit goes to the original author; this is an archived copy.

---

Event Challenge Category Link TG:Hack CTF 2020 Useless Crap PWN https://ctftime.org/event/932

​

Description

Here’s some useless crap for you. The flag is at /home/crap/flag.txt. nc crap.tghack.no 6001 or use a mirror closer to you: nc us.crap.tghack.no 6001 (US)

(US) nc asia.crap.tghack.no 6001 (Japan) files: crap

libc-2.31.so

ld-2.31.so

TL;DL

Leak Libc address through unsorted bin chunks UAF

Get infinte arbitary read/write/heap allocations

Write open/read/write ROP chain after srip of main function using allowed syscalls

Stack pivot to our ROP chain and read flag.txt TGHack 2020 had some challenging PWN tasks, and Useless Crap was one of the hardest that we were able to solve during the CTF with our teammate Aracna.
