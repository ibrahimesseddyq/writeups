# linker_revenge

**CTF:** 3kCTF 2020  
**Category:** misc  
**Original writeup:** <https://wrecktheline.com/writeups/3kctf-2020/> — by **wrecktheline.com**. All credit goes to the original author; this is an archived copy.

---

Linker Revenge - 9 Solves by FeDEX Menu based challenge with the option to: Add page

Edit page

Delete page

View page

Relogin The vulnerability is a Use After Free, in the Edit method. When deleting a page, the pointer to the page is not set to NULL and the check for editing is done via the page pointer instead of check_pages . This vulnerability allows us to overwrite the fd pointer in a freed heap chunk and allocate a chunk in the .bss . Now that we have control over the array of pointers I will first overwrite an entry in the page array with a GOT address in order to leak libc. Once we got a libc leak, we can overwrite an entry in the page array with the address of environ in order to leak the stack address. Once we got a stack leak, we can overwrite an entry in the page array with the address of the return value of the main function. Now we can start building a ropchain to run mprotect(0x602000, 0x100, 0x7) and read(0, 0x602000, 0x100). The final step consists of a shellcode which will perform: openat(0, './flag', 0)

read('rax', 0x602700, 0x100)

write(1, 0x602700, 0x100) Exploit can be found HERE
