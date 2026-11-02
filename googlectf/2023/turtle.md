# Turtle

**CTF:** GoogleCTF 2023  
**Category:** rev  
**Original writeup:** <https://bronson113.github.io/2023/06/26/googlectf-2023-writeup.html> — by **bronson113**. All credit goes to the original author; this is an archived copy.

---

Turtle

1 2 3 Are we not all but turtles drifting in the sea, executing instructions as we stumble upon them? solves: 27

turt.py (From googleCTF github)

This is a vm implemented using turtles! After watching the program run for a bit, we can observe what each turtle are responsible for. For example, the S turtle act like a stack, C is like code, and R is like some registers. Based on this knowledge, I write a translator (translate.py) from c.png “code” file to get a pseudocode (trans.txt) of the program. I notice that each 5 pixel movement for the turtle seems to be for 1 unit, so I translate a lot of the moving instructions into pointer arithmatic, as they are easier to work with.

With the translated pseudocode, we notice that the program first check for the flag format, then make sure all characters are unique, make sure the characters are in a specific order, and lastly run some function on each character.

It took a while to understand that the function is actually a binary search on the character value, and match each step with some value in memory. For example, if the target value is smallar than the current middle element, we first match if the memory value is 1, then call the same search function recursively. To recover the flag, I extract the reference value, take all the found characters, and order it using the reording in the memory. See simplify.txt for notes taken during the process, and solve.py for more detail.

CTF{iT5_E-tUr+1es/AlL.7h3;waY:d0Wn}
