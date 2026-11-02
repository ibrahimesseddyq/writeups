# Shai-Hulud

**CTF:** SpaceHeroes 2022  
**Category:** rev  
**Original writeup:** <https://breaking-bits.gitbook.io/breaking-bits/spaceheros-ctf-2022/re-shai-hulud> — by **breaking-bits.gitbook.io**. All credit goes to the original author; this is an archived copy.

---

The Shai-Hulud problem was the capstone reversing problem in spaceheros CTF 2022. The problem had one attached binary titled "worm"

The challenge presents a game like snake which expects you to collect all the "*" marks on the map using the "wasd" keys.

Inside the binary we have some game setup followed by a standard game play loop:

A couple things stand out immediately, like the hard-coded seed into srand ( of 0x2454 ) and the SHA256Init() call. This means the challenge will likely deal with some predictable psuedo random number generation and the SHA256 init call is indicative of us building a hash as a final answer.

The clear() and wormsign() functions are responsible for the game setup, where clear() will setup the game grid to have all "-1" in them and wormsign will initialize the player piece with a starting position and value.

The arrakis global symbol is used to reference the game board and is composed of a 2 dimensional array of integers. The double for loop in the clear function tells us the the array is an 0x13 by 0x20 grid. We can visually confirm it by looking back at the running game too.

The wormsign function sets our initial X and Y positions in addition to the worm length. We need this worm length value to be 0x295 to have the loop back in main() print our flag.

The first solve - play the game

My first thought towards solving the problem was asking myself, "why not play the game"? If we can write something to very quickly play the game and produce the 0x295 score, it should just print out the flag.

To do that we need to speed the game as much as possible. We can do that first by removing the usleep call in that main function loop. One option is patching, but it was faster for me to use the LD_PRELOAD trick and write a quick library:

Copy // gcc usleep_preload.c -shared -o usleep_preload.so int usleep(unsigned int sec){}

Now we can run the program with LD_PRELOAD=./usleep_preload.so ./worm and it'll load our library and run the game play loop VERY fast.

The next step is writing a solver. Since the game loops the @ character at boundry, we can write a lawn mower algorithm to solve the game by using only the "w" and "d" keys. We simply run through a column, then at the end go over to the next one.

In practice we hit our tail and fail the game really early on with this algorithm, so I opted to patch out the fail condition. Binary ninja simplified the view, but if the draw() function returns 0, then the game stops.

I chose to set rax_27 to 1 on line 0xd2a with a patch, so no matter whether we hit our tail, the lawn mower algorithm wouldn't close the game. I used radare2 with a wa mov rax, 1 command inside of r2.

With those two pieces set I wrote a quick script in pwntools to run the program and constantly send in those "w" and "d" command until it won.

Copy from pwn import * import os os . environ [ "PWNLIB_NOTERM" ] = "1" p = process ( 'LD_PRELOAD=./usleep_preload.so ./worm' , shell = True , raw = False ) moves = [ b "w" , b "a" , b "s" , b "d" ] def handler ( signum , frame ): p . interactive () import signal signal . signal (signal.SIGINT, handler) while True : p . send ( b "w" ) p . send ( b "d" ) data = p . recv () if len (data): for line in data . split ( b '

' ): if b "shctf" in line : print (line) break if b "Size" in line : print ([x for x in line. split ( b ':' ) if b "POW" in x][ 0 ])

I learned through this method that you can't actually beat the game legitimately. The game will always stop at a worm size of 196 due to a bug in placing the next * piece.

Both worm size 195 and 196 place the * on the same place and a bug in that logic means that the arrakis array never gets updated correctly.

I confirmed this bug with the challenge author, but needed to find another way to solve it.

Absolute overkill method

At this point the better way to solve this problem is to reverse the vibration function and determine the X and Y values that would be generated for each size and manually build the pieces ourselves. But I really wanted the program to print me the flag so... I opted to over-instrument and write a really complicated set of hooks to generate the flag.

Hooking

I'm a big fan of the LD_PRELOAD trick, and with that trick did you know that you can hook a binaries main function? Well really the libc function to call it. So the idea here is to run our code AFTER the program has been loaded into memory so that we can reference all the symbols and functions within our hooks.

My generic hook for the program looks like the following:

Copy /* * Wrapper for __libc_start_main() that replaces the real main * function with our hooked version. */ int __libc_start_main( int (*main)(int, char **, char **), int argc, char **argv, int (*init)(int, char **, char **), void (*fini)(void), void (*rtld_fini)(void), void *stack_end) { /* Save the real main function address */ main_orig = main; /* Find the real __libc_start_main()... */ typeof(&__libc_start_main) orig = dlsym(RTLD_NEXT, "__libc_start_main"); printf("main located at 0x%lX

", main_orig); main_addr = main_orig; /* ... and call it with our custom main function */ return orig(main_hook, argc, argv, init, fini, rtld_fini, stack_end); }

I compiled this hook using: gcc solve_worm_main_hook.c -o main_hook.so -fPIC -shared -ldl which links in the dlfcn headers and I ended up running LD_PRELOAD=./main_hook.so ./worm each time I wanted to test and run my hook.

Setup

My objective was to replicate all the functionality out of the main() function in the binary, so the very first thing I needed were references to each function and symbol. Unfortunately they weren't exported so I couldn't use dlsym to do it, so I relied on pwntools' ELF parsing to show me the offsets from the program's main function.

In the hook for main, the very first argument passed to my is the relocated/PIE address of main and so I can use that to calculate where everything else in memory is loaded! I set it as a global variable and referenced it inside of my setup code.

Copy int main_hook(int argc, char **argv, char **envp) { puts("In main hook"); // no dlsym, no problem void (*clear_func)() = main_addr - (4864 - 2730); void (*wormsign_func)() = main_addr - (4864 - 4694); unsigned long (*vibration_func)() = main_addr - (4864 - 2826); void (*print_flag_func)() = main_addr - (4864 - 4741); void (*frame_func)() = main_addr - (4864 - 2987); void (*draw_func)() = main_addr - (4864 - 3623); }

This technique would enable me to call a function like clear() by calling clear_func() in my hook, which is perfect for emulating the calls in main(). My next issue was getting the values for the worm symbol and other global variables used across the program.

Since those symbols were globals, they're mapped into the program's BSS and aren't guaranteed to be at a certain offset after the loader loads the binary into memory. So I needed a way to search for them.

Scanning memory

Since the program hasn't run yet, nothing is initialized. Fortunately we can call the functions present in the binary like the wormsign function!

Since the wormsign functions writes to these variables, we have values we can search for in memory. So the plan is to call the wormsign function and then search memory for them. The BSS section is a R/W section, so we can use GDB to dump which sections we should be trying to parse:

So far we're replicating the start of the main function and then we're scanning memory to find our variables:

Copy // Replicate what main() is doing in the binary srand(0x2454); clear_func(); // Call wormsign first so we can find the buf for the // SHA init wormsign_func(); unsigned int * worm = find_bss_addrs(); unsigned int * X_loc = worm+(8 / 4); unsigned int * Y_loc = worm+(12 / 4); unsigned char * buf = worm+ (0x20 / 4); unsigned char * buf2 = worm- (0x20 / 4);

The scan is going to iterate over /proc/self/maps and find that r/w memoy region and then start looking from the start of that region to the end of the region until we find the 0x10 and 0xa for our X and Y values:

Copy int * find_bss_addrs() { /* Addresses are laid out like below: 00202b00 int32_t worm = 0x0 00202b04 int32_t direction = 0x0 00202b08 int32_t X_loc = 0x0 00202b0c int32_t Y_loc = 0x0 after wormsign: X_loc = 0x10; Y_loc = 0xa; worm = 1; direction = 0; // So use those as a heuristic to find it! */ char line[PATH_MAX] = {0}; unsigned int line_count = 0; unsigned long start_addr = 0; unsigned long stop_addr = 0; char * path = "/proc/self/maps"; char *ptr; /* Open file */ FILE *file = fopen(path, "r"); /* Get each line until there are none left */ while (fgets(line, PATH_MAX, file)) { if (strstr(line,"rw-") && strstr(line, "worm")) { // 56370c202000-56370c203000 rw-p 00002000 103:02 57418616 /home/chris/ctf/spaceheroes/worm printf("%s

", line); start_addr = strtoul(line, &ptr, 16); ptr += 1; // Jump over - stop_addr = strtoul(ptr, &ptr, 16); printf("Scanning from 0x%lX - 0x%lX

", start_addr, stop_addr); } } /* Close file */ fclose(file); int * x_ptr = (int *)start_addr; // Scan memory looking for x_loc == 0x10 for (x_ptr = start_addr; x_ptr < stop_addr; x_ptr++) { // Check for 0x10 and 0xa being set if (*x_ptr == 0x10 && *(x_ptr + sizeof(int)) == 0xa) { printf("Found x_loc at 0x%lX

",x_ptr); return x_ptr; } } return start_addr + 0xb00; // test add for no va_randomize }

This reliably returns the location of the worm global variable and since the rest of the globals are relative to it in the BSS, we can use those offsets to give our hook references to them!

Reading the game map

Just like the other variables, the game map is a hardcoded offset away from the worm variable, so I set it in the hook.

Copy printf("worm at %lX

",worm); unsigned int * arrakis = worm; arrakis -= (0xa80 / 4); printf("arrakis at %p

",arrakis); // no need to steal the binaries call to Init since it's the same SHA256_Init(buf); // 20 x 13 grid // First one at X=13,Y=6 vibration_func();

I followed the rest of the main() functions flow and needed a way to detect where the X and Y values would be placed after a vibration() call. The vibration function is responsible for setting the X and Y locations of the new * to go and find. It sets it in the massive two dimensional array for that arrakis variable.

Instead of modifying the function or reversing it, I opted to scan the entire arrakis buffer looking for the change that the vibration function performed:

The function sets the value to "-2", so we can use that as a marker to search for!

Copy int * find_change(int * arrakis) { int i =0; for (i = 0; i < (0xa60 / 4); i++) { if (*(arrakis+i) == 0xfffffffe) // -2 is marker { printf("Found change at %lx

", arrakis+i); return arrakis+i; } } return NULL; }

Once we have the location, it's important to reset it so our locating function doesn't return the same address each time:

Copy // Vibration function will change a value in the // arrakis symbol, we can scan memory for it and // use that to determine our X and Y values new_loc = find_change(arrakis); *new_loc = 0xffffffff; // reset it // For 2-D arrays, we can get our X value // by dividing by our width and our Y // by doing a modulo operator marker_x = (new_loc - arrakis) / 20; marker_y = (new_loc - arrakis) % 20;

Run game loop and print the flag

Now that we have our all pieces in place the rest of the hook just needs us to set our global X and Y coordinates based on that found value. Then we need to copy out the part of the program that updates that weird hash we saw earlier and loop it.

The frame() function contains the portion responsible for updating the hash:

We generate an aggregate value from our X and Y positions and feed it into the SHA256Update function. Then a SHA256Final call is made and our worm counter is incremented and another vibration() call is made to set the next *.

So all together is looks like this:

Copy while (*worm != 0x295) { // Vibration function will change a value in the // arrakis symbol, we can scan memory for it and // use that to determine our X and Y values new_loc = find_change(arrakis); *new_loc = 0xffffffff; // reset it // For 2-D arrays, we can get our X value // by dividing by our width and our Y // by doing a modulo operator marker_x = (new_loc - arrakis) / 20; marker_y = (new_loc - arrakis) % 20; printf("%d : Marker is at X: %d, Y: %d

",*worm, marker_x, marker_y); // Copy what the innerweb of frame() is doing to build // our aggregate and update the hash aggregate = marker_y + (marker_x * 0x10); SHA256_Update(buf, &aggregate, 8); SHA256_Final(buf2, buf); vibration_func(); *worm +=1; } print_flag_func();

Copy shctf{pr41s3_th3_r3m4k3_4nd_1ts_fr4m3s}
