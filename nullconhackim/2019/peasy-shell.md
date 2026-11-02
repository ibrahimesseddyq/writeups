# peasy-shell

**CTF:** nullconHackIM 2019  
**Category:** misc  
**Original writeup:** <https://lordidiot.github.io/2019-02-03/nullcon-hackim-ctf-2019/> — by **lordidiot**. All credit goes to the original author; this is an archived copy.

---

peasy-shell

one more easy shell for free! nc pwn.ctf.nullcon.net 4011 challenge +exploit.py

Overview

Now, after Enigmatrix had solved easy-shell, we began working on the sequel peasy-shell . This challenge is pretty much the same, with the big difference that the shellcode region is made RX before we jump to it. This is tragic as it kills our self-modifying strategy to achieve the syscall instruction. … Or does it?

If you were to do a quick reversing of the make_rx function that they use to do this, you will notice something very important.

int __fastcall make_rx ( void * a1 ) { int result ; // eax@1 result = mprotect ( a1 , 1uLL , 5 ); if ( result < 0 ) { put ( "mprotect failed: rx" ); exit ( - 1 ); } return result ; }

The len parameter on the mprotect call is only 1! In this case, what will happen is that mprotect will round it upwards to 0x1000, however, our mmaped region is larger than 0x1000 :D. This means that we can use the same self-modifying shellcode strategy as before, just that our syscall instruction is 0x1000 bytes after the beginning.

Approach

Now that we know this, we can try to modify our inital shellcode to get this working. An additional challenge is that the condition of the registers is also different from before, and our registers are no longer setup nicely for a read syscall. This requires a lot of weird popping and xor-ing in the stack to get our registers set nicely. I think the tricks are quite cool, so I suggest that you take a look through the shellcode and figure it out :P (feel a bit lazy to describe them thoroughly).

After the modifications to fix the registers, we place many two byte nops to traverse all the way to the syscall instruction. Perfect! We read in our second stage payload and get the flag locally! Now lets run it on the server!

[+] Opening connection to pwn.ctf.nullcon.net on port 4011: Done 103 [*] Switching to interactive mode [*] Got EOF while reading in interactive

oof

Ah, maybe it’s failing because the exploit only works 50% of the time, even locally (this is because I am using an xor trick to increment a register, this creates a situation where sometimes it decrements, and sometimes it increments).

[+] Opening connection to pwn.ctf.nullcon.net on port 4011: Done 103 [*] Switching to interactive mode [*] Got EOF while reading in interactive

oof *20 more tries

Now our shellcode should not be failing this many times, a 50% chance to work is very good already. So why does it fail 100% on remote but work locally? Then I remembered something I learnt from someone in OpenToAll in the past. When sending our data across the network, they are sometimes chunked into smaller packets before being sent across, this will happen for larger chunks of data. However, the read call that is initially reading our stage 1 payload will not consider this chunking, and only read the first chunk. Since our shellcode is a giant size of 0x1032 bytes (with all the nops), only a part of our shellcode is read into memory, causing the exploit to fail. So now there’s only one option: We need to make it smaller

Size matters

So what’s interesting about this challenge is that at the end of our shellcode, the binary will add a ret instruction.

* (( _BYTE * ) buf + v6 ) = 0xC3u ;

What you may realise is, if you can push the address of our syscall instruction onto the stack, and get to the ret instruction, this is effectively a jump! Therefore, we can modify our shellcode to the xor magic to form the syscall instruction inside eax. Then we get that syscall instruction in memory using xor DWORD PTR [rcx + imm8], eax as before. Now just push the correct address, and we ret to the syscall ! Just as before, we read in our second stage open-read-write shellcode and get the flag!

hackim19{maybe_this_is_where_you_stop_getting_easy_shells}
