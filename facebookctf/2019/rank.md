# rank

**CTF:** FacebookCTF 2019  
**Category:** misc  
**Original writeup:** <https://ling.re/fbctf-rank/> — by **ling.re**. All credit goes to the original author; this is an archived copy.

---

I participated in FBCTF 2019 with my team glua.team where we ended 20th place. One of the challenges we solved was rank .

Analysis

The provided binary is a 64-bit ELF file called rank . Furthermore, a libc file called libc-2.27.so was provided. This means that we most likely have to do ret2libc or a onegadget.

Running checksec on the provided binary yields the following result:

ling@sola:~/Documents/ctf/fb/rank$ checksec r4nk [*] '/home/ling/Documents/ctf/fb/rank/r4nk' Arch: amd64-64-little RELRO: Partial RELRO Stack: Canary found NX: NX enabled PIE: No PIE (0x400000) FORTIFY: Enabled

Figure 1: Output of rank .

Since most security features are enabled, most notably stack canaries, there most likely isn’t going to be a stack buffer overflow.

The program is a simple ranking system. The user is able to show the ranking of all the titles using the show command. This prints all the titles and their respective rank. The other command that is available is rank . This allows the user to change the rank of a title. The output of running the binary can be seen on Figure 1.

Reversing the binary

Opening up the binary in IDA shows that symbols are stripped which makes it a little harder for us. The decompilation of the main function can be seen on Figure 2. Some of the function and variable names have been assigned by me during analysis.

Figure 2: Decompilation of main .

This function doesn’t do anything special besides setting a 30 second timeout and calling main_loop . The decompilation of this function is pretty big but the most important part can be seen on Figure 3.

Figure 3: Decompilation of main_loop .

The function will get user input as a number in an infite loop. The show command can be issued by entering 1 , the rank command by entering 2 , and the program will exit by entering 3 . The first 2 command each have their own function that will handle all the logic. The show_func function doesn’t do anything special and will just print all the titles. The vulnerability in this binary is present in the rank_func function. The decompilation of this function can be seen on Figure 4.

Figure 4: Decompilation of rank_func .

Pwning the binary

This function will get 2 numbers from the user input and then move some data around using this number. Since there are no bounds checks present in this function, this grants us the ability to move any memory we want. The only limitation is that we are limited to an 8 byte boundary. Using this, we can leak the GOT entry for __libc_start_main to get the base address of libc.

1 2 3 4 5 6 7 8 9 10 11 # Step 1: Leak __libc_start_main rank ( "0" , "-263035" ) show () io . recvuntil ( "0. " ) LEAK = u64 ( io . recv ( 6 ) + " \x00\x00 " ) log . info ( "leak @ " + str ( hex ( LEAK ))) LIBC = LEAK - libc . symbols . __libc_start_main log . success ( "libc @ " + str ( hex ( LIBC )))

Using this address, we can ROP to a onegadget and get a shell. By issuing the rank command and using title higher than 16, the return pointer can be overwritten. This allows us to ROP to the onegadget. The problem however, is that the address of libc is so high that the call to strtol will turn the number negative as seen on Figure 5.

Figure 5: Sign problem with strtol .

To get around this problem, we jumped to a pop rsp gadget to pivot the stack. We pointed the new stack pointer to the input buffer. This allowed us to bypass the strtol call and use any characters we wanted.

1 2 3 4 5 6 7 8 9 10 # Step 2: Overwrite return pointer rank ( "17" , str ( int ( 0x400980 ))) # pop rsp rank ( "18" , str ( int ( 0x602120 ))) # rsp (read buffer) rank ( "19" , str ( "0" )) # garbage rank ( "0" , "A" * 40 + p64 ( one_gadget )) # padding + our return pointer quit () io . interactive ()

By issuing the quit command, we jumped to our stack pivot. After the pivot, the stack would point to our onegadget and we would get a shell.
