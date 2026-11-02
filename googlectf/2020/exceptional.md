# exceptional

**CTF:** GoogleCTF 2020  
**Category:** misc  
**Original writeup:** <https://ctf.harrisongreen.me/2020/googlectf/exceptional/> — by **ctf.harrisongreen.me**. All credit goes to the original author; this is an archived copy.

---

Exceptional

This binary is quite exceptional in its structure. Do you accept the challenge?

I spent nearly 12 hours on this challenge and finally solved it at 6:30 am after staying up all night and going through at least 5 cups of coffee. r3ndom helped with some of the later reversing steps.

The challenge was very fun and I developed a lot of patching techniques that I think will be handy for related problems in the future.

Files:

exceptional : original binary

: original binary exceptional_patch : heavily patched binary with better control flow

Part 1: Investigation

After opening the binary, we find that the control flow has been fragmented into tons of small blocks that are linked together with call ’s:

Our main function is:

which calls:

and that calls:

and so on…

To get some reasonable disassembly, we can convert all of these call instructions into an equivalent jmp . This will hint to our disassembler (I use Binary Ninja) that all this code is part of a single subroutine.

Patching by hand is tedious so we can use regular expressions™

Transform 1:

In x86, a 4-byte relative call is encoded as e8 xx xx xx xx . To convert to 4-byte relative jump, we simply replace the first byte with e9 . The following regex can be used in python: b'\xe8[^\xe8](.|

){3}' (there is a false positive with an e8 right before a call so we can avoid that specific case).

Before replacing the jump, we should also ensure thst the jump target is one of the user-defined functions (in the range [0x1160, 0x2b0f] ).

I found that this patch ends up working fairly well by itself. However, the decompiler gets confused with all the stack frame shenanigans. To prevent this I removed all the function prologues from our jump targets:

After applying these modifications, I found that binja had a much easier time with decompilation. It was able to chain some blocks into larger chunks:

New main:

which calls:

As we start to poke around, we see a lot of calls to __cxa_throw in strange places:

We also find several handler routines that seem to just invoke a different user subroutine:

It is evident that this exception handling stuff is just used to obfuscate control flow. In order to deobfuscate the binary, we need to figure out the mapping between a __cxa_throw call and a specific handler routine.

Part 2: Exception patching

I briefly investigated the __cxa_throw mechanics and observed the following:

There are 5 different possible exception types (specified as tinfo in the __cxa_throw call).

in the call). There are 15 handler routines

There are 23 places where __cxa_throw is used

So how does the program figure out which __cxa_throw goes to which handler? Well, in short, it’s magic:

At this point we have two options:

Learn about how C++ exceptions work in order to make this process not magic Use gdb to dynamically figure out this exception mapping buisness

Of course, being a lazy CTF player, I used option 2. The process works as follows:

Set a breakpoint at every call __cxa_throw instruction. Set a breakpoint inside every handler routine. (this is equivalent to setting a breakpoint at every call __cxa_begin_catch instruction). Interact with the program normally and wait for the program to break on a call __cxa_throw . Continue so it hits a corresponding handler breakpoint. Keep track of the throw: handler mapping. GOTO 3

Eventually, I uncovered mappings for most of the __cxa_throw calls. Luckily all except three of these were deterministic and always mapped to the same handler. The other three could map to one of two different handlers.

Unfortunately, because I chose option 2 earlier, I have no idea why this happens and this is still magic.

# address mapping # (call __cxa_throw) -> (call __cxa_begin_catch) TARGETS = { 0x1335 : 0x2392 0x1357 : 0x15e3 0x13d8 : 0x2844 0x1430 : 0x21b5 0x149b : 0x18e5 0x1528 : 0x1649 0x18a7 : 0x2bbb 0x198f : 0x2844 0x1a2c : 0x2392 0x1be2 : 0x14c1 0x1fb9 : 0x2bbb 0x24ed : 0x2211 0x257b : 0x2844 0x2736 : 0x2844 0x29b3 : 0x2bbb 0x2a88 : 0x2408 } MULTI = { 0x1c90 : [ 0x17fa , 0x2486 ] 0x2607 : [ 0x2967 , 0x229d ] 0x292b : [ 0x17fa , 0x2486 ] }

To deobfuscate these exception “jumps” we can use a similar strategy as before. Each time we encounter a call __cxa_throw , we can simply patch in a jump directly to the handler routine. Since the handlers all follow a similar pattern, we can even bypass the __cxa_begin_catch and jump directly to where the handler returns control to the user code:

In the case where we have two possible exception handlers, we need to somehow indicate to the disassembler that either handler could be used. We can use a similar patching strategy but this time, insert a jz to decide which target to take. For the purposes of this kind of patching, we don’t need the final executable to actually work; we just want some reasonable control flow to work with:

Using this strategy we end up with some very clean control flow and we can get some readable decompilation such as this add_city function:

(There is a residual unpatched __cxa_throw at the end here but it doesn’t affect the main logic)

Part 3: Reversing

Now that we have a usable binary, we can start reversing what the program is doing. Essentially the program keeps track of cities and lets us convert between different timezones. Here is an example interaction:

Welcome to city timezone calculator. Menu: 1. Add city 2. List cities 3. Calculate time 4. Quit 1 Type city name and timezone (e.g. "NewYork -4"): NewYork -4 Menu: 1. Add city 2. List cities 3. Calculate time 4. Quit 1 Type city name and timezone (e.g. "NewYork -4"): other_city 5 Menu: 1. Add city 2. List cities 3. Calculate time 4. Quit 2 List of cities: 01. NewYork UTC-4 02. other_city UTC+5 Menu: 1. Add city 2. List cities 3. Calculate time 4. Quit 3 Type your city and current time (e.g. "NewYork 12:34"): other_city 12:10 Type the other city: (e.g. "Washington"): NewYork Current time in NewYork: 03:10. Menu: 1. Add city 2. List cities 3. Calculate time 4. Quit 4

The add_city method (shown above) is fairly straightforward and we see that the city struct is something like:

struct city { int unk1; int unk2; // -1 int unk3; // -1 int timezone; char [ 0x30 ] name; };

City information is stored in an array of size 100 starting at 0x75d0 however internally cities are referenced by an index where the address is computed as 0x7120 + (idx * 4) . So the first city is at index 0x12c .

Conveniently the program also reads from flag.txt and stores the flag at address 0x7440 (equivalent index: 0xc8 ).

After initializing the struct, add_city calls link :

This function reveals how city information is stored. At the start it computes a 32-bit hash of the city name via something like:

def str_hash (s, k = 0x12345678 ): s = (s + ( b ' \x00 ' * 48 ))[: 48 ] # pad for i in range( 0xc ): blk = s[i * 4 :i * 4 + 4 ] v = struct . unpack( '<I' , blk)[ 0 ] k ^= v return k

Then it iterates through a sorted binary tree of cities (sorted by hash) until it either finds the city and prints "City already exists" or it doesn’t find the city and instead inserts it into the binary tree. So with this information, we can complete the city struct:

struct city { int hash; struct city * left; struct city * right; int timezone; char [ 0x30 ] name; };

Note: left and right are not actually pointers but rather “city indexes” as described earlier

Part 4: Exploitation

Based on the way city data is structured, it seems clear that the goal is to corrupt this data and somehow interpret the flag as a city struct which could let us leak information.

During the reversing process, I found that the binary uses an “operation stack” that seems to be used to curry information between different basic blocks. For example, during add_city it pushes 1 (the add operation number) and then city_idx . These values are later popped off and the stack is reset before the program loops.

However, the stack starts at 0x9120 and conveniently grows towards smaller addresses. If we can get this to overlap our city array, we could corrupt some city information.

At this point it is useful to draw out a memory diagram:

I spent some time trying to figure out how to collide the stack with the city data and I eventually realized that when you call compute_time , the program will push the current hash and index to the stack as it searches through each node in the binary tree.

So given a very lopsided binary tree, you could force a “worst-case” search where it iterates close to 100 times.

It’s fairly easy to construct a lopsided tree; we can simply compute a bunch of hash values and then sort them:

names = [str(i) . encode( 'ascii' ) for i in range( 100 )] names = [(x, str_hash(x)) for x in names] names = sorted(names, key = lambda x: x[ 1 ])[:: - 1 ] order = [x[ 0 ] . decode( 'ascii' ) for x in names] print(order)

# names with descending hash values [ '79' , '69' , '59' , '49' , '39' , '29' , '19' , '99' , '89' , '78' , '68' , '58' , '48' , '38' , '28' , '18' , '98' , '88' , '71' , '61' , '51' , '41' , '31' , '21' , '11' , '91' , '81' , '70' , '60' , '50' , '40' , '30' , '20' , '10' , '90' , '80' , '73' , '63' , '53' , '43' , '33' , '23' , '13' , '93' , '83' , '72' , '62' , '52' , '42' , '32' , '22' , '12' , '92' , '82' , '75' , '65' , '55' , '45' , '35' , '25' , '15' , '95' , '85' , '74' , '64' , '54' , '44' , '34' , '24' , '14' , '94' , '84' , '77' , '67' , '57' , '47' , '37' , '27' , '17' , '97' , '87' , '76' , '66' , '56' , '46' , '36' , '26' , '16' , '96' , '86' , '7' , '6' , '5' , '4' , '3' , '2' , '1' , '0' , '9' , '8' ]

After a bit of experimenting, I found that if I added 98 cities with these names and then searched for a city with a lower hash than all the cities (such that we need to traverse in the worst case), I could overflow city[97].left with the hash value of the city we are searching for (as it is pushed to the stack during compute).

So what is our goal here?

Well, a nice feature of compute_time is that it prints the message "Current time in %s: %d:%d." and specifically dereferences the target city we provide. Now this message will only print if the target city is actually found. This means the hash of the city we provide must match the hash of a city in the tree.

So if we can add index 0xc4 to the tree (right before flag data), we would have a city with values:

struct city { int hash = 0 ; struct city * left = 0 ; struct city * right = 0 ; int timezone = 0 ; char [ 0x30 ] name = < flag > ; // at address 0x7440 };

Then if we search for a name with hash 0 , such as the string 3aaaK7Us , we would match this fake city and it should print the flag.

Now there is a slight caveat: we want the first search to take a long time so we have a stack collision. However when we search for the flag hash, we want this to be quick so we don’t accidentally overwrite city data a second time. To do this, we simply need to set up two chains: city[0] -(left)-> city[97] with descending hashes that will let us search for the flag quickly after we overwrite city[97] . And we also want: city[0] -(right)-> city[1] -> city[2] ... that will let us search for the hash 0xc4 and collide the stack.
