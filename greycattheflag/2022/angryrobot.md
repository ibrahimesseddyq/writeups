# AngryRobot

**CTF:** GreyCatTheFlag 2022  
**Category:** rev  
**Original writeup:** <https://rainbowpigeon.me/posts/grey-cat-the-flag-qualifiers-2022/> — by **rainbowpigeon.me**. All credit goes to the original author; this is an archived copy.

---

🩸 Angry Robot

You have entered a top secret robot production facilities and your clumsy friend tripped the alarm. You and your friends are about to be “decontaminated”. Luckily, you have unpacked the authentication firmware during previous reconaissance. Can you use them to override the decontamination process? MD5 (authentication.tar.gz) = 965b5a18735af4bbfa81879e2cedc9cc rootkid nc challs.nusgreyhats.org 10523

Attached: authentication.tar.gz

First Look

In the provided archive, there are 100 stripped 64-bit ELF binaries.

1 2 3 4 5 6 7 8 a@a:/grey/authentication$ ls -1 03e73f2bd4c480b209caa321eb3e2eb974b680e2ec7cbada8cfd457375eaa3c0 04480e6a6ad345bf50f4f428f802656264d4738a000270d8967734cecd0ba2a9 08d4a97677fd1c8d14b069b9922518e5c7a680b67417990d998c41357d4c7c14 ... ... a@a:/grey/authentication$ ls -1 | wc -l 100

1 2 a@a:/grey/authentication$ file 03e73f2bd4c480b209caa321eb3e2eb974b680e2ec7cbada8cfd457375eaa3c0 03e73f2bd4c480b209caa321eb3e2eb974b680e2ec7cbada8cfd457375eaa3c0: ELF 64-bit LSB executable, x86-64, version 1 ( SYSV ) , dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, BuildID [ sha1 ]= 3f33add36e8e79282d0d0ed3aaa826744ddc5a39, for GNU/Linux 3.2.0, stripped

If we open a few of them in IDA Pro , we’ll see that all of these binaries basically do the same thing: fgets is used to read an input string from stdin and then each character is checked against 2 hardcoded strings using some arithmetic operations. What differs across the binaries, though, are the input length, the 2 hardcoded strings, the constant used for the arithmetic operations, and the addresses of the main function and some locations.

Below is the pseudocode for the 03e73f2bd4c480b209caa321eb3e2eb974b680e2ec7cbada8cfd457375eaa3c0 binary with variables and function names renamed by me:

1 2 3 4 5 6 7 8 9 10 11 12 13 14 15 16 17 18 19 20 21 22 23 24 25 26 _BOOL8 __fastcall main ( int a1 , char ** a2 , char ** a3 ) { * input = 0LL ; fgets ( input , 30 , stdin ); return check_input ( input ) == 0 ; } __int64 __fastcall check_input ( __int64 input ) { qmemcpy ( str1 , "Li?ittSAXlsKEi9d2GQNP3uCOTHDD" , 29 ); qmemcpy ( str2 , "Yp9ASTqGSZ`psG3IU7jk3TdeFvwpm" , 29 ); is_success = 1 ; for ( i = 0 ; i <= 28 ; ++ i ) { if ( check_chr ( * ( i + input ), i ) != * ( str1 + i ) || check_chr ( * ( i + input ), * ( str1 + i )) != * ( str2 + i ) ) is_success = 0 ; } return is_success ; } __int64 __fastcall check_chr ( char a1 , int a2 ) { if ( a1 <= '\x1F' || a1 == '\x7F' ) exit ( 1 ); return (( a2 * a1 % 1431 + a1 + a2 ) % 73 + 48 ); }

If we connect to the server and port provided in the challenge description, we’ll see that we’re supposed to provide the correct input strings (passwords) for specified binaries (challenge codes) within 10 minutes.

1 2 3 4 5 Unauthorized access detected. Decontamination in 10 minutes Do you wish to override? ( y/n ) y Password required for challenge code: ffbb5c9d6d32299cc53c90d410282b42f7431c60ee40540f19603f0b4c1cfc90

Technically there’s enough time to do this manually since the server turned out to only request input strings for 5 binaries, but I didn’t know this and went ahead to learn and use angr for automated reverse engineering which I suppose is what’s hinted by the challenge name. It went surprisingly well for my first time!

Using angr

We’ll be using angr’s SimulationManager to help us find a program state with the correct input string by means of symbolic execution, so there’s a few things we need to identify in each binary we process:

Addresses we want to find/reach

Addresses we want to avoid

Input length

Addresses

The is_success = 0; basic block in check_input with address 0x401305 is what we’ll tell angr to avoid.



And in main we want check_input(input) == 0; to fail meaning we don’t want the jz after check_input to be taken, so the location we should tell angr to find for is 0x401395 .



But of course as mentioned earlier, I noticed these address locations and the address of main change across the binaries provided, so I first located fgets using the ELF binary’s Procedure Linkage Table (PLT) and then retrieved the first and only callgraph predecessor of fgets which is main .

To get to the basic block in check_input that we want angr to avoid, I retrieved the second last call site in main which is call check_input . I then retrieved the return site of the last function call in check_input which is a call check_chr , and added the offset of 0xF to reach the mov [rbp+is_success], 0 that we want to avoid.

To get to the basic block in main that we want angr to find, I sorted the basic blocks in main by address and took the address of the 5th last basic block.

Input

I retrieved the first basic block in main and searched through the instructions for mov esi, xx where xx is the length argument given to fgets to retrieve user input. xx - 1 will then represent the actual input length because fgets will also take in a newline character.

I also constrained the input to only be printable characters because the code in check_chr appears to try to check for that.

Solution Script

1 2 3 4 5 6 7 8 9 10 11 12 13 14 15 16 17 18 19 20 21 22 23 24 25 26 27 28 29 30 31 32 33 34 35 36 37 38 39 40 41 42 43 44 45 46 47 48 49 50 51 52 53 54 55 56 57 58 59 60 61 62 63 64 65 66 67 68 69 70 71 72 73 74 75 76 77 78 79 80 81 82 83 84 85 86 87 88 89 90 91 92 93 94 95 96 97 98 99 100 101 102 103 import angr import claripy import capstone import logging from pwn import * logging . getLogger ( "claripy.ast.bv" ) . setLevel ( "ERROR" ) logging . getLogger ( "angr.storage" ) . setLevel ( "ERROR" ) logging . getLogger ( "angr.simos" ) . setLevel ( "ERROR" ) logging . getLogger ( "angr.analyses.cfg" ) . setLevel ( "ERROR" ) def char ( state , c ): return state . solver . And ( c <= "~" , c >= " " ) def solve_file ( file ): print ( file ) p = angr . Project ( "authentication/ {} " . format ( file ), auto_load_libs = False ) fgets_addr = p . loader . main_object . plt [ "fgets" ] cfg = p . analyses . CFGFast () # find the basic block to FIND fgets_callers = cfg . functions . callgraph . predecessors ( fgets_addr ) # get the first and only calling predecessor which is main main_func = [ cfg . functions [ caller ] for caller in fgets_callers ][ 0 ] print ( "main: {} " . format ( hex ( main_func . addr ))) main_blocks = sorted ( main_func . blocks , key = lambda b : b . addr ) # we want to reach the 5th last block where the jz is not taken FIND = main_blocks [ - 5 ] . addr print ( "FIND: {} " . format ( hex ( FIND ))) # find length of input required via # mov esi, 1Eh first_block = main_blocks [ 0 ] for insn in first_block . capstone . insns : if ( len ( insn . operands ) == 2 and insn . mnemonic == "mov" and insn . operands [ 0 ] . reg == capstone . x86 . X86_REG_ESI and insn . operands [ 1 ] . type == capstone . x86 . X86_OP_IMM ): INPUT_LENGTH = insn . operands [ 1 ] . imm - 1 print ( "input length: {} " . format ( INPUT_LENGTH )) # find the basic block to AVOID functions = cfg . kb . functions [ main_func . addr ] main_calls = functions . get_call_sites () # second last call in main is check_input check_input_addr = [ functions . get_call_target ( call ) for call in main_calls ][ - 2 ] functions = cfg . kb . functions [ check_input_addr ] check_input_calls = functions . get_call_sites () # get the last return site of the function calls in check_input AVOID = max ( filter ( None , [ functions . get_call_return ( call ) for call in check_input_calls ], ) ) OFFSET = 0xF AVOID += OFFSET print ( "AVOID: {} " . format ( hex ( AVOID ))) flag = claripy . BVS ( "flag" , INPUT_LENGTH * 8 ) state = p . factory . entry_state ( stdin = flag ) state . options . add ( angr . options . LAZY_SOLVES ) for c in flag . chop ( 8 ): state . solver . add ( char ( state , c )) sm = p . factory . simulation_manager ( state ) sm . explore ( find = FIND , avoid = AVOID ) print ( "finding..." ) if sm . found : found = sm . found [ 0 ] solution = found . solver . eval ( flag , cast_to = bytes ) print ( "found: {} " . format ( solution )) else : print ( "not found" ) solution = None print ( "-" * 64 ) return solution if __name__ == "__main__" : conn = remote ( "challs.nusgreyhats.org" , 10523 ) conn . recvuntil ( b ")" ) conn . recvline () conn . sendline ( b "y" ) response = b "" while True : response = conn . recvline () if b ": " in response : challenge_code = response . split ( b ": " )[ - 1 ] . rstrip ( b "

" ) . decode () password = solve_file ( challenge_code ) conn . sendline ( password ) else : break print ( response )

1 2 3 4 5 6 7 8 9 10 11 12 13 14 15 16 17 18 19 20 21 22 23 24 25 26 27 28 29 30 31 32 33 34 35 36 37 38 39 40 41 42 43 44 45 46 47 48 49 [ x ] Opening connection to challs.nusgreyhats.org on port 10523 INFO | 2022-06-11 03:22:06,986 | pwnlib.tubes.remote.remote.2891224537696 | Opening connection to challs.nusgreyhats.org on port 10523 [ x ] Opening connection to challs.nusgreyhats.org on port 10523: Trying 35.247.145.55 INFO | 2022-06-11 03:22:06,992 | pwnlib.tubes.remote.remote.2891224537696 | Opening connection to challs.nusgreyhats.org on port 10523: Trying 35.247.145.55 [ + ] Opening connection to challs.nusgreyhats.org on port 10523: Done INFO | 2022-06-11 03:22:07,013 | pwnlib.tubes.remote.remote.2891224537696 | Opening connection to challs.nusgreyhats.org on port 10523: Done a9f59335ea82cf55f4f7f7dd5d8eb3f8ded783bd2d902adcc25a4aca75328895 main: 0x40131f FIND: 0x401380 input length: 25 AVOID: 0x4012f7 finding... found: b '08xsiic4i626rqn2urg5yhx0x' ---------------------------------------------------------------- 65636c7430d22186ac375d173de25ea6305d5aa7b0feb4526e1ec1dd2bcc16cd main: 0x40132b FIND: 0x40138d input length: 27 AVOID: 0x401303 finding... found: b 'vccbxfmn6cr1mvfe2f902sgzdpf' ---------------------------------------------------------------- 70b4bed62eb3e1c2cbbd9e393154a7b5a403e973400c662df032d4f59fe8a333 main: 0x40132b FIND: 0x40138d input length: 27 AVOID: 0x401303 finding... found: b 'brrb92dt6qeolkxdalihzid6kte' ---------------------------------------------------------------- 5e346c4ce1773d5e4a5bb498dd3c9d5124d03ecfc494e15ee5ac6b643be7ee9d main: 0x40132f FIND: 0x401397 input length: 29 AVOID: 0x401307 finding... found: b 'zxokdvjcwbdo0bq55xpoq7qv23df5' ---------------------------------------------------------------- d98f704409467ff7d85c439371b0f1acf1e5850b6e030effe4c9d2a9af83c825 main: 0x401331 FIND: 0x40139d input length: 30 AVOID: 0x401309 finding... found: b 'wtg60dw2ew3tgn112c0nybfbr29o9|' ---------------------------------------------------------------- b 'grey{A11_H4il_SkyN3t}

' [ * ] Closed connection to challs.nusgreyhats.org port 10523 INFO | 2022-06-11 03:23:39,041 | pwnlib.tubes.remote.remote.2891224537696 | Closed connection to challs.nusgreyhats.org port 10523

Flag: grey{A11_H4il_SkyN3t}
