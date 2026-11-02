# AnUnknownDisassembly

**CTF:** SpaceHeroes 2022  
**Category:** rev  
**Original writeup:** <https://hong5489.github.io/2022-04-04-spaceheroctf/> — by **hong5489**. All credit goes to the original author; this is an archived copy.

---

An Unknown Disassembly

Description

Attachment

We got a text file, looks like python bytes code:

5 0 LOAD_GLOBAL 0 (input) 2 LOAD_CONST 1 ('Enter the super secret password:') 4 CALL_FUNCTION 1 6 STORE_FAST 0 (a) 6 8 LOAD_CONST 2 ('') 10 STORE_FAST 1 (b) 7 12 LOAD_CONST 3 (0) 14 STORE_FAST 2 (c) 8 16 LOAD_FAST 0 (a) 18 GET_ITER >> 20 FOR_ITER 136 (to 158) 22 STORE_FAST 3 (x) ... ... ... 30 182 LOAD_GLOBAL 1 (print) 184 LOAD_CONST 14 ('You got the flag!') 186 CALL_FUNCTION 1 188 POP_TOP 31 190 LOAD_GLOBAL 2 (exit) 192 CALL_FUNCTION 0 194 POP_TOP 33 >> 196 LOAD_GLOBAL 1 (print) 198 LOAD_CONST 15 ('Oops, try again.') 200 CALL_FUNCTION 1 202 POP_TOP 204 LOAD_CONST 0 (None) 206 RETURN_VALUE

Analyse

After analyse the code, it just replace our input from:

3 to e or e to 3

to or to a to @ or @ to a

to or to o to 0 or 0 to o

After that it compare the result to S0th3combination1sonetw0thr3efourf1ve , if equals then it will print the flag! (when connect to the netcat port)

Then I wrote a python script to reverse the convertion:

text = "S0th3combination1sonetw0thr3efourf1ve" password = "" for t in text : if t == 'a' : password += '@' elif t == '@' : password += 'a' elif t == '0' : password += 'o' elif t == 'o' : password += '0' elif t == 'e' : password += '3' elif t == '3' : password += 'e' else : password += t print ( password ) # Sothec0mbin@ti0n1s0n3twothre3f0urf1v3

But when I type the password in netcat, it is incorrect!

nc 0.cloud.chals.io 27178 Enter the super secret password:Sothec0mbin@ti0n1s0n3twothre3f0urf1v3 Oops, try again.

Solving

Looks like I missed an important part of the code:

25 >> 148 LOAD_FAST 2 (c) 150 LOAD_CONST 11 (1) 152 INPLACE_ADD 154 STORE_FAST 2 (c) 156 JUMP_ABSOLUTE 20 27 >> 158 LOAD_CONST 12 ('S0th3combination1sonetw0thr3efourf1ve') 160 STORE_FAST 4 (d) 28 162 LOAD_FAST 2 (c) 164 LOAD_CONST 13 (4) 166 BINARY_MODULO 168 LOAD_CONST 3 (0) 170 COMPARE_OP 2 (==) 172 POP_JUMP_IF_FALSE 196 29 174 LOAD_FAST 1 (b) 176 LOAD_FAST 4 (d) 178 COMPARE_OP 2 (==) 180 POP_JUMP_IF_FALSE 196

The variable c must be divisible by 4 then it only compare our input to the string, and we can control c by adding letter l

Because it doesn’t add l to the compare variable b and will add variable c so we can add l to control c :

21 >> 130 LOAD_FAST 3 (x) 132 LOAD_CONST 10 ('l') 134 COMPARE_OP 2 (==) 136 POP_JUMP_IF_FALSE 140 22 138 JUMP_FORWARD 8 (to 148) 24 >> 140 LOAD_FAST 1 (b) 142 LOAD_FAST 3 (x) 144 INPLACE_ADD 146 STORE_FAST 1 (b) 25 >> 148 LOAD_FAST 2 (c) 150 LOAD_CONST 11 (1) 152 INPLACE_ADD 154 STORE_FAST 2 (c) 156 JUMP_ABSOLUTE 20

Our input length is 37 , so we need to add three l to our input to let c = 40:

nc 0.cloud.chals.io 27178 Enter the super secret password:Sothec0mbin@ti0n1s0n3twothre3f0urf1v3lll You got the flag! shctf{1m_jUst_a_p14in_y0gurt_ch4l1eng3}

That’s it! Simple reverse challenge

Flag

shctf{1m_jUst_a_p14in_y0gurt_ch4l1eng3}
