# OldSchool

**CTF:** GoogleCTF 2023  
**Category:** rev  
**Original writeup:** <https://w0y.at/writeup/2023/07/18/google-ctf-2023-oldschool.html> — by **w0y.at**. All credit goes to the original author; this is an archived copy.

---

Write an oldschool keygen for an oldschool login interface.

Google CTF 2022 presented us with oldschool , a typical, as the name suggests, oldschool crackme with an ncurses terminal interface. The goal of the challenge was to write a keygen, which would be able to generate keys for a list of users provided by the CTF organizers. The official and detailed writeup is available here, which goes through the intended solution of manually reverse engineering the key verification algorithm. However, since we are researchers (and most importantly, too lazy to manually reverse engineer the key verification), we took a look at the binary and decided that this must be solvable using symbolic execution. Tl;dr, yes, it is indeed possible, but our angr skills were a bit rusty and it took us a bit to get to the solution.

Initial Overview of the Binary

To start off, we took a look at what we were dealing with and saw that we were provided with a 32-bit ELF binary. To avoid having to pull in old 32-bit dependencies on our host system, we used the i386/debian Docker image and installed libncurses6 , as the binary wouldn't start without it. Running it for the first time, we were then presented with the following error message:

root@699a7aa1afd8:/oldschool# ./oldschool [!] Error. ASSERT_EQ failed!

This didn't really help us a lot. Was this an error message from ncurses, or was this part of the oldschool binary itself? The string cannot be found in the binary directly, but we quickly realized that all relevant strings in the binary have been obfuscated. After every call to functions from libncurses , we could see decoding of error strings like this:

bVar17 = 0 ; local_18 = & stack0x00000004 ; iVar7 = initscr (); if ( iVar7 == 0 ) { endwin (); FUN_00012a08 (); for ( local_98 = 0 ; local_98 < 0x11 ; local_98 = local_98 + 1 ) { FUN_0002bd72 (); uVar5 = FUN_0002bd96 (); puVar8 = ( undefined * ) FUN_0002bd72 (); * puVar8 = uVar5 ; } puVar8 = ( undefined * ) FUN_0002bd72 (); * puVar8 = 0 ; FUN_0002bdc6 (); FUN_000129b6 (); for ( local_9c = 0 ; local_9c < 0xe ; local_9c = local_9c + 1 ) { FUN_0002bcfc ( auStack_2858 ); uVar5 = FUN_0002bd20 ( local_2867 ); puVar8 = ( undefined * ) FUN_0002bcfc ( auStack_2858 ); * puVar8 = uVar5 ; } puVar8 = ( undefined * ) FUN_0002bcfc ( auStack_2858 ); * puVar8 = 0 ; pcVar9 = ( char * ) FUN_0002bd50 ( auStack_2858 ); fprintf ( _stderr , pcVar9 ); /* WARNING: Subroutine does not return */ exit ( -1 ); } iVar7 = cbreak (); if ( iVar7 != 0 ) { /* ... */ fprintf ( _stderr , pcVar9 ); /* WARNING: Subroutine does not return */ exit ( -1 ); } iVar7 = curs_set (); /* ... */

Looking at the decompiled code (and also the CFG of the function) reveals function calls, whose value is checked in an if condition followed by some code looping, which we assumed to be the code that prints the error message. In order to get the program to run, we realized that it needed the correct TERM variable to be set for libncurses , as this was a common problem when getting these programs to run:

export TERM = xterm-256color

While we were working on manually noting down the function calls between the chunks of error printing code, we were at the same time running the code through ltrace , in order to track the executed dynamic library calls. During manual analysis of the decompiled code, we stumbled upon the following software interrupt (namely int 0x80 , a syscall in x86 Linux):

pcVar2 = ( code * ) swi ( 0x80 ); iVar7 = ( * pcVar2 )(); * ( int * )( iStack_2900 + 0x32c ) = - iVar7 ; local_28 = local_74 ; while ( local_28 = local_28 + 1 , local_28 <= local_74 + 0x11 ) {

We didn't check what this syscall did for now, but noted it in the back of our minds, as it seemed extremely out of place for the binary to manually perform a syscall. So we went on the play around with ltrace in the hopes we could find something that would narrow down the search window for looking for meaningful stuff within the main function. And we did, as the ltrace contained the following:

[ 0 x56567f01 ] strlen ( "thisistheusername " ... ) [ 0 x56567f61 ] newwin ( 7 , 50 , 34 , 88 ) [ 0 x56567f96 ] newwin ( 7 , 50 , 35 , 89 ) [ 0 x56568181 ] box ( 0 x565b85a8 , 0 , 0 , 89 ) [ 0 x5656835a ] wbkgd ( 0 x565b74e8 , 1312 , 0 , 89 ) [ 0 x5655714c ] strlen ( "thisisthepassword" )

We can see that strlen is called two times here, once for the username and once for the password, with ltrace thankfully giving us the instruction pointer for instruction following the strlen (we disabled ASLR using echo 0 | sudo tee /proc/sys/kernel/randomize_va_space so that ltrace would always give us the same addresses). Checking the strlen XRefs in Ghidra, we can click through and find the two respective function calls with the username being checked in the main function and the password being checked in another one.

* ( int * )( puVar8 + -0x10 ) = local_90 ; * ( undefined4 * )( puVar8 + -0x14 ) = 0x22f01 ; local_58 = strlen ( * ( char ** )( puVar8 + -0x10 ));

The strlen takes puVar8 + -0x10 as argument, which is set earlier to local_90 which is where our username is located. Looking for other occurrences of local_90 leads us to this piece of code:

* ( undefined4 ** )( puVar8 + -0xc ) = & local_28e9 ; * ( int * )( puVar8 + -0x10 ) = local_90 ; * ( undefined4 * )( puVar8 + -0x14 ) = 0x2353b ; iVar7 = FUN_0001212a (); if ( iVar7 == 1 ) {

Ghidra does not really show this nicely, due to the arguments being placed on the stack, but FUN_0001212a actually takes two parameters, one of them being the username. We can also see that the function return value is checked for 1, quickly looking at the ltrace output shows us that the else branch is taken (the return value is not 1) if our password is false. So we can assume that this function takes username and password, returns 1 if the password is correct and 0 if the password is incorrect. Peeking into the function confirms this, as we can see the password strlen that we previously identified within our ltrace output:

uStack_270 = 0x1213b ; sVar2 = strlen ( param_2 ); if ( sVar2 == 29 ) {

This also already gives us a hint that the password should have a length of 29 characters. The function only has 200 lines of decompiled C code (which allows us to ignore the rest of the 7000 lines of decompiled C code of the main function). Reading on shows us some more relatively simple to reverse infos about the password:

if (((( local_20 == 5 ) || ( local_20 == 0xb )) || ( local_20 == 0x11 )) || ( local_20 == 0x17 )) { if ( param_2 [ local_20 ] != '-' ) { return 0 ; } }

From this code we can see that every 5th character of the password is supposed to be a - . For every other character, the following is executed:

local_29 = '\0' ; for ( i = 0 ; ( int ) i < 0x20 ; i = i + 1 ) { FUN_000120da ( local_b3 , & local_71 ); for ( j = 0 ; j < 0x20 ; j = j + 1 ) { puVar3 = ( undefined * ) FUN_0002b8d6 ( auStack_92 , j ); local_25d = FUN_0002b8fa ( local_b3 , * puVar3 , j ); puVar3 = ( undefined * ) FUN_0002b8d6 ( auStack_92 , j ); * puVar3 = local_25d ; } puVar3 = ( undefined * ) FUN_0002b8d6 ( auStack_92 , 0x20 ); * puVar3 = 0 ; iVar5 = FUN_0002b92a ( auStack_92 ); if ( iVar5 [ i ] == password [ local_20 ]) { iVar6 = local_24 / 5 ; iVar7 = local_24 % 5 ; local_24 = local_24 + 1 ; local_118 [ iVar6 * 5 + iVar7 ] = i ; local_29 = '\x01' ; break ; } } if ( local_29 != '\x01' ) { return 0 ; }

The most interesting line here is iVar5[i] == password[local_20] , which effectively ensures that the other characters of the password are only valid if they are part of the lookup table iVar5 . Checking with GDB, we saw that this lookup table stays the same (despite being recreated every iteration), corresponding to:

23456789 ABCDEFGHJKLMNPQRSTUVWXYZ

This means that this string is as password that passes all checks for the formatting:

23456 - 789 AB - CDEFG - HJKLM - NPQRS

Now at this point we looked at the rest of the code and figured that we were too lazy to reverse it by hand and thought that angr should be able to easily solve this automatically. This seemed reasonable as the structure of the function lends itself well for solving, since we have a username and password as parameter, and 0 or 1 as feedback if the combination is valid or not. Also, all loops appear to be bounded given the input limits and there is nothing crazy happening that would cause us a headache when trying symbolic execution.

First Steps with Angr

Our first attempt with angr was to create a call_state at the function of interest, with parameters provided by us, and then exploring to the location in the function where 1 is returned. This looked something like that:

check_addr = proj . loader . min_addr + 0x212a password = PointerWrapper ( claripy . BVS ( 'password' , 0x1e * 8 ), buffer = True ) username = PointerWrapper ( b 'gdwAnDgwbRVnrJvEqzvs \x00 ' , buffer = True ) state = proj . factory . call_state ( check_addr , username , password , prototype = 'int password_check(char *username, char *password)' ) simgr = proj . factory . simgr ( state ) val_addr = proj . loader . min_addr + 0x29a8 simgr . explore ( find = val_addr )

Now, we haven't touched angr in forever, and we're not sure if that is how you actually use the PointerWrapper. We tried to run this, and it worked, but it didn't terminate within a reasonable amount of time. Debugging this a little, we realized that we already had issues passing the first password check loop iteration, even when constraining the password to the correct characters. We realized, that we were likely missing out on state that would be initialized before the function is called, but letting the whole program run in angr would most likely not yield satisfying results. Due to the ncurses interface of the application, we were also not sure how well we can control the program automatically/from the outside, so using angr inbuilt concolic execution techniques might not have worked.

Concolic Execution

So our main way of thinking was: given a program in GDB, can we dump its state and use that with angr ? If you think "wow that sounds cool", we have to disappoint you a bit, because we didn't end up doing that exactly, but we found something that was sufficient. After searching for GDB angr integrations and not being able to find something recent that works, we thought of whether we could somehow make use of avatar2, which we knew from previous research. While this does not have direct angr integration, angr does have a module which allows us to use avatar2 with the GDB backend, called Symbion. So using Symbion from angr , so that we can use avatar2 , so that we can finally achieve our goal of syncing GDB state with angr .

To start of, we start oldschool in a GDB server and then connect to it using the Symbion avatar2 target:

avatar_gdb = AvatarGDBConcreteTarget ( avatar2 . archs . x86 . X86 , '127.0.0.1' , 12345 , 'gdb' )

Then, we set up angr as usual, creating an entry state and explore to the address offset 0x2747 . We picked this offset, as it is located within the function right before username state and password state is mixed together, meaning this is the last possible location for us to replace the password with symbolic variables if we want to solve for it.

p = angr . Project ( './oldschool' , concrete_target = avatar_gdb , use_sim_procedures = True ) entry_state = p . factory . entry_state () entry_state . options . add ( angr . options . SYMBION_SYNC_CLE ) entry_state . options . add ( angr . options . SYMBION_KEEP_STUBS_ON_SYNC ) target_addr = BASE_ADDRESS + 0x2747 simgr = p . factory . simgr ( entry_state ) simgr . use_technique ( Symbion ( find = [ target_addr ])) exploration = simgr . run ()

So far so good. If we just run gdbserver 0.0.0.0:12345 ./oldschool in our container and start the angr script, the password prompt of the oldschool binary will show up. Upon entering the password, our script successfully explores the binary to the specified location and returns us a state, great! Now we can go ahead and extend our script to introduce symbolic variables for solving. To this end, we take the value from the ebp register and subtract 0x114 , which corresponds to the start address of the (substituted) password on the stack, which we determined from Ghidra. We know the length of the password without the separating - is 25, and that every character of the password has been substituted with a 32-bit number, which corresponds to the index of the respective password character in the array of allowed characters. For each of these 25 characters, we create a 32 bit symbolic variable. Because we know that the lookup dictionary of valid characters is 32 characters long, we can also add a constraint to highlight that the symbolic variable cannot be initialized with values bigger than 32. Finally, we write the symbolic variables to the memory and save them for later:

# Previous state we reached through concrete execution (Symbion) state = exploration . found [ 0 ] # Defining symbolic vars symbolic_vars = [] password_base = state . regs . ebp - 0x114 for i in range ( 25 ): svar = state . solver . BVS ( f 'pw { i } ' , 8 * 4 ) state . add_constraints ( claripy . ULT ( svar , 0x20 )) state . mem [ password_base + i * 4 ] . uint32_t = svar symbolic_vars . append ( svar )

Now we want to explore to the location in the program, where 1 is returned, indicating a correct password:

target_addr = BASE_ADDRESS + 0x29a8 simgr = p . factory . simgr ( state ) simgr . explore ( find = target_addr )

We start the GDB server and run the angr script again, but after entering our password, angr seems to crash due to memory areas being presumably not mapped. Our guess as to why this was happening was that maybe the base address for the symbolic execution was not the same as for the concrete execution. After some research, we found that this bug is an open issue referenced here. The issue can ultimately be circumvented by specifying the base address of the concrete execution when creating the angr project:

BASE_ADDRESS = 0x56555000 p = angr . Project ( './oldschool' , concrete_target = avatar_gdb , use_sim_procedures = True , load_options = { 'main_opts' : { 'base_addr' : BASE_ADDRESS }} )

After setting the base address and repeating the GDB server/angr script procedure, the symbolic execution takes a few seconds and we get a valid state, yay! This means there is definitely at least one solution for the symbolic variables that we specified and that we can now read out this solution, by asking the solver to evaluate our symbolic variables:

state = simgr . found [ 0 ] concrete_vals = [] for svar in symbolic_vars : val = state . solver . eval ( svar , cast_to = bytes ) concrete_vals . append ( val )

The values we get back are the 32-bit integers after substitution of the password characters with the indices of the character dictionary. In order to get the actual password, we need to substitute it again, split it into chunks of 5, and add - between the chunks:

def chunks ( l , n ): for i in range ( 0 , len ( l ), n ): yield l [ i : i + n ] lookup = '23456789ABCDEFGHJKLMNPQRSTUVWXYZ' c = chunks ( '' . join ([ lookup [ x [ - 1 ]] for x in concrete_vals ]), 5 ) print ( '-' . join ( c ))

Which gives us the following for the first username in the list we have to crack ( gdwAnDgwbRVnrJvEqzvs ):

GL3EX-E3WHZ-EJ57M-UEPYX-62PK7

Anti-Debug

Of course, we go ahead and try if that actually works and... it does not? After some trial and error, we figured out that the password does work - if we run the binary using the gdbserver. This means we are dealing with some anti-debugging and made us remember the syscall we found before. We guessed that this was the ptrace syscall using the well known ptrace(PTRACE_TRACEME, 0, 1, 0) trick, since this will fail if the program is already being ptraced (like when we run it in GDB). While we could just patch that check out, we decided it would be nice to go ahead and just use the existing angr/Symbion/avatar2 flow to intercept ptrace/change the return value. So instead of exploring to the location from where we want to do symbolic execution, we explore to the location of the ptrace call and change the return value. To this end, we set the return register to 0, and then tell Symbion to concretize the changes (i.e. copying them to the GDB target):

# Exploring until after the ptrace call entry_state = p . factory . entry_state () entry_state . options . add ( angr . options . SYMBION_SYNC_CLE ) entry_state . options . add ( angr . options . SYMBION_KEEP_STUBS_ON_SYNC ) target_addr = BASE_ADDRESS + 0xaf79 simgr = p . factory . simgr ( entry_state ) simgr . use_technique ( Symbion ( find = [ target_addr ])) exploration = simgr . run () # Exploring until the location where we want to do symbolic execution state = exploration . found [ 0 ] state . regs . eax = 0 target_addr = BASE_ADDRESS + 0x2747 simgr = p . factory . simgr ( state ) simgr . use_technique ( Symbion ( find = [ target_addr ], register_concretize = [( 'eax' , state . regs . eax )])) exploration = simgr . run ()

After that, we continue our symbolic execution as before. After evaluating it, we now get this password for the first username:

2 UTQS - QFE2D - Z3P9K - 6 CFM9 - TRRH5

We try it out and it works!

Now the challenge requires us to do this for 49 more usernames, then concatenate everything together and hash it, to give us the flag. We didn't automate this process as the terminal interface was a bit of a pain to automate (we were too lazy), so we just sat down and did it 49 times manually.
