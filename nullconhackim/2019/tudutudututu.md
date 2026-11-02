# tudutudututu

**CTF:** nullconHackIM 2019  
**Category:** misc  
**Original writeup:** <https://lordidiot.github.io/2019-02-03/nullcon-hackim-ctf-2019/> — by **lordidiot**. All credit goes to the original author; this is an archived copy.

---

tudutudututu

I found a ToDo service to maintain my daily tasks. My life is sorted now! nc pwn.ctf.nullcon.net 4003 challenge +exploit.py

Menu: (1) Create a new todo (2) Set description for a todo (3) Delete an existing todo (4) Print todos (5) Exit >

Another heap menu pwnable!

Overview

This challenge is supposed to be a program to create todos to remember things. When creating a new todo, the program does a simple malloc(0x10) followed by a strdup(user_input) , creating 2 chunks in the heap.

[ 0x10 bytes ] todo_1 [ 0x? bytes ] topic_1

The structure of the todo is as follows:

struct todo { char * topic ; char * description ; }

Now after we initialise the todo with a topic, we have the additional option of adding a description to the todo, this will allocate a new chunk and set the pointer to the chunk in the struct of the todo. The fatal error however, is that this description pointer is not initialised to zero when a todo is created. This means that if there is leftover data in the chunk when it was allocated, the program will think that it is the description pointer. When we reverse the delete functionality, we can see these lines of code.

if ( desc ) free ( desc ); free ( _todo );

Thus, if we can control the value of this uninitialised description member, we can exploit it through the delete functionality of the program. In essense, should we control this description member, we have an arbitrary free primitive.

Exploitation

Now that we understand the main bug, we can try to exploit this. The steps to exploit the uninitialised member are as follows.

create new todo, with 0x10 byte topic [ 0x10 bytes ] todo_1 [ 0x10 bytes ] topic_1 free this todo 0x10 bytes old_todo_1 0x10 bytes old_topic_1 allocate 2 new todos, with topics larger than 0x10 bytes [ 0x10 bytes ] todo_2 [ 0x10 bytes ] todo_3 !! ( this used to be old_topic_1 ) [ 0x? bytes ] topic_2 [ 0x? bytes ] topic_3

As we can see from this scenario, todo_3 uses the same chunk that topic_1 previously used. In this case, we can write an address at the 8 byte offset of topic_1 , this will then be used as the description of todo_3 later on. With this primitive, we can arbitrarily read any address, and free any address. Since we can leak any address, I utilised this to leak the heap address and the libc base.

rip control

Now that we have all the leaks we probably will ever need, we need to figure out how to control rip to change program execution. How I did this was to transition our arbitrary free to a fastbin double free . Since we knew the address of the heap, we could create 2 todos that had descriptions or topics pointing to the same chunk. If we freed both these todos, we would thus create a circular fastbin list, allowing us to perform a fastbin attack . Now to utilise this fastbin attack to control rip , I made it return an arbitrary chunk before the location of __malloc_hook . This allows me to overwrite the value of __malloc_hook with a one_gadget . And that gets us our shell!

hackim19{D0nt_f0r93t_t0_1ni7i4liz3}
