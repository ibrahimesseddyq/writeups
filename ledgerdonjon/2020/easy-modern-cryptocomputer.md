# easy-modern-cryptocomputer

**CTF:** LedgerDonjon 2020  
**Category:** misc  
**Original writeup:** <https://blog.cryptohack.org/hacking-eos-ledger-donjon-ctf-writeup> — by **blog.cryptohack.org**. All credit goes to the original author; this is an archived copy.

---

Easy Modern Cryptocomputer (100pts)

A smart contract seems to be a good place to hide a secret flag. Doesn’t it?

We were given a README and a Dockerfile for building a Docker image running a patched EOSIO node.

EOSIO is a blockchain which can run smart contracts, similar to Ethereum. For our purposes, a key feature of EOSIO is that the smart contracts are compiled WebAssembly programs that run sandboxed on EOSIO nodes. EOS smart contracts are usually written in C++, but developers can use any programming language that can be compiled to WebAssembly (wasm).

The idea behind this challenge is that a CRC32 intrinsic function has been added to EOSIO nodes, but there might be a problem with it. So the creators have added a secret flag to their node which can be read by smart contracts through the new get_secret_flag intrinsic that reads an environment variable. The function is part of the wasm “privileged API”, therefore only a system user or a privileged contract on the challenge remote node should be allowed to call it. On the other hand, any user or contract may call the crc32 function. Since the Dockerfile builds from the latest copy of the EOSIO source code, it looks like we need to abuse the crc32 function to grab the flag (unless we can find a zero day in EOS ;)).

Here’s the patch provided - note that wasm_interface.cpp contains intrinsics that can be called from smart contracts such as hash functions:

+++ b / libraries / chain / wasm_interface . cpp @@ - 314 , 6 + 314 , 17 @@ class privileged_api : public context_aware_api { }); } + uint32_t get_secret_flag ( array_ptr < char > dest , uint32_t length ) { + const char * flag = getenv ( "FLAG" ); + if ( ! flag ) { + flag = "No flag provided" ; + } + if (( size_t ) length >= strlen ( flag )) { + length = ( uint32_t ) strlen ( flag ); + } + :: memcpy ( dest , flag , length ); + return length ; + } }; class softfloat_api : public context_aware_api { @@ - 909 , 6 + 920 , 18 @@ class crypto_api : public context_aware_api { void ripemd160 ( array_ptr < char > data , uint32_t datalen , fc :: ripemd160 & hash_val ) { hash_val = encode < fc :: ripemd160 :: encoder > ( data , datalen ); } + + // Implement the same CRC as binascii.crc32 in Python + uint32_t crc32 ( array_ptr < char > data , uint32_t value , uint32_t datalen ) { + printf ( "INPUT POINTER: %p

" , & data [ 0 ]); + value = ~ value ; + for ( int i = 0 ; i < datalen ; i ++ ) { + value ^= data [ i ]; + for ( unsigned int bitpos = 0 ; bitpos < 8 ; bitpos ++ ) { + value = ( value >> 1 ) ^ ( 0xEDB88320 & - ( value & 1 )); + } + } + return ~ value ; + } };

However, it turns out that this “easy” version of the challenge didn’t require any memory exploitation. In fact, the main part of this challenge was understanding EOS and figuring out how to get up and running with it.

First, by iterating over blocks on the blockchain of the remote node, we were able to find a smart contract:

$ curl http://modern-cryptocomputer.donjon-ctf.io:30510/v1/chain/get_block -d '{"block_num_or_id": 223}' { "timestamp" : "2020-10-06T15:54:59.000" , "producer" : "eosio" , "confirmed" :0, "previous" : "000000de5cb1cb15e6e2b40f1c2c01b43de1313b154ff2fcd9a6e5d528445f68" , "transaction_mroot" : ...

This block contained a WebAssembly program in hex, and associated data. An actions array showed that an actor called “flagchecker” is allowed to update the authorisation on the contract. So we were pretty sure we’d stumbled upon the right thing to investigate.

We decompiled the WebAssembly using wabt. This gave messy pseudocode which took some work to untangle, but eventually we were able to find locations in the code which performed CRC32 over input, and compared an input hash to the SHA256 of the flag. We also found that a flag was getting copied to memory location 0x10000 (although we later discovered this was the flag for the hard version of the challenge).

We wanted to start experimenting with calling this contract, and by reading the EOSIO docs, realised that we could deploy the WASM on our local node if we had the contracts’ Application Binary Interface (ABI). We were able to extract that from the blockchain also:

{ "version" : "eosio::abi/1.1" , "types" : [], "structs" : [ { "name" : "checkhashflg" , "base" : "" , "fields" : [ { "name" : "user" , "type" : "name" }, { "name" : "hash_flag_hex" , "type" : "string" } ] } ], "actions" : [ { "name" : "checkhashflg" , "type" : "checkhashflg" , "ricardian_contract" : "" } ], "tables" : [], "ricardian_clauses" : [], "error_messages" : [], "abi_extensions" : [], "variants" : [] }

So the flagchecker contract has a single function, checkhashflg , with two parameter, user and hash_flag_hex .

After following the EOSIO developers’ tutorial from sections 1.4 to 1.6, we had a local copy of the node running and could deploy this smart contract to it, using the following invocation:

$ cleos set contract eosio /opt/eosio/contracts/easy/ -p eosio@active

Now the contract was on our own blockchain, we could push an action to it:

$ cleos push action eosio checkhashflg '{"user": "eosio", "hash_flag_hex": "00"}' -p eosio executed transaction: 539c06f374427cd51f27f2c4e60151fed6b01b6b7f073bc3b4d080896b1f6a49 104 bytes 791 us # eosio <= eosio::checkhashflg {"user":"eosio","hash_flag_hex":"00"} >> Hello, eosio! You are not allowed to call this contract.

Initially we thought this was an error coming from the node due to an incorrect way we had called the contract (we had faced enough errors already to get to this point), we eventually realised this message was coming from the contract itself. So closer dissasembly of why it was appearing would be required. After some more reading we noticed this bit of decompilation:

h = env_crc32 ( e , 0 , h ); if ( eqz ( d [ 64 ] : ubyte & 1 )) goto B_i ; f_gb (( d + 72 )[ 0 ] : int ); label B_i : d [ 27 ] : int = h ; if ( h != 1226134910 ) goto B_l ;

This revealed we needed a username that CRC32’s to the value 1226134910. This was easy to bruteforce as “maze”.

Finally, we needed to create a “maze” account on our local node and call checkhashflg using maze as the principle, and the contract spat out the easy flag:

$ cleos create account eosio maze EOS87K6LyQ48We4Pyq9zumXHSQjzozLX5A8FHgxEnrrWh955ZSQkd $ cleos push action eosio checkhashflg '{"user": "maze", "hash_flag_hex": "00"}' -p maze Error 3050003: eosio_assert_message assertion failure Error Details: assertion failure with message: Unexpected SHA256 size pending console output: Hello, maze! easyflag = CTF { 04f5f3fbbc08dac23645890b03dd0d72fed6c5988621e62295610ff23a377e3b }
