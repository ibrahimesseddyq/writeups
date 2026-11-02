# Calculator

**CTF:** GreyCatTheFlag 2022  
**Category:** misc  
**Original writeup:** <https://rainbowpigeon.me/posts/grey-cat-the-flag-qualifiers-2022/> — by **rainbowpigeon.me**. All credit goes to the original author; this is an archived copy.

---

Calculator

Calculator is a nice invention mechfrog88 US Instance: nc 35.184.68.167 15521 EU Instance: nc 34.77.124.65 15521

nc challs.nusgreyhats.org 15521

1 2 3 4 5 6 7 8 9 10 11 12 13 14 15 16 17 18 19 20 21 22 23 24 25 26 27 28 29 30 31 32 33 a@a:~$ nc challs.nusgreyhats.org 15521 __________ . '----------`. | .--------. | | |########| | __________ | |########| | /__________\ .--------| `--------' | ------ | -- = -- | -------------. | ` ----,-.----- ' | o ====== | | | ______ | _ | _______ | __________ | | | / %%%%%%%%%%%% \ | | / %%%%%%%%%%%%%% \ | | ^^^^^^^^^^^^^^^^^^^^ | +-----------------------------------------------------+ ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ How fast are you on calculating basic math expression? Answer 100 math questions in 30 seconds to the flag Operations include: 1. add x y -> returns x + y 2. mul x y -> returns x * y 3. sub x y -> returns x - y 4. neg x -> returns -x 5. inc x -> returns x + 1 Example1: mul add 1 2 sub 5 1 Ans1: ( 1 + 2 ) * ( 5 - 1 ) = 12 Example2: add mul sub 3 2 inc 5 3 Ans2: ( 3 - 2 ) * ( 5 + 1 ) + 3 = 9 Send START when you are ready!

Prefix notation is used to evaluate these math expressions, so I just took a Python prefix calculator script from https://stackoverflow.com/a/5307565 and edited the tokens to match those required in this challenge:

1 2 3 4 5 6 7 8 9 10 11 12 13 14 15 16 17 18 19 20 21 22 23 24 25 26 27 28 29 30 31 32 33 34 35 36 from collections import deque from pwn import * def parse ( tokens ): token = tokens . popleft () if token == b 'add' : return parse ( tokens ) + parse ( tokens ) elif token == b 'sub' : return parse ( tokens ) - parse ( tokens ) elif token == b 'mul' : return parse ( tokens ) * parse ( tokens ) elif token == b 'neg' : return parse ( tokens ) * ( - 1 ) elif token == b "inc" : return parse ( tokens ) + 1 else : # must be just a number return int ( token ) if __name__ == '__main__' : conn = remote ( 'challs.nusgreyhats.org' , 15521 ) conn . recvuntil ( b '!' ) conn . recvline () conn . sendline ( b 'START' ) for i in range ( 100 ): problem = conn . recvline () problem = problem . rstrip ( b '

' ) answer = ( parse ( deque ( problem . split ()))) conn . sendline ( bytes ( str ( answer ), 'ascii' )) conn . recvline () result = conn . recvline () print ( result ) flag = conn . recvline () print ( flag )

Script output:

1 2 3 4 5 a@a:/grey$ python3 prefix_calculator.py [ + ] Opening connection to challs.nusgreyhats.org on port 15521: Done b 'You done it in 21.74351406097412 seconds!

' b 'grey{prefix_operation_is_easy_to_evaluate_right_W2MQshAYVpGVJPcw}

' [ * ] Closed connection to challs.nusgreyhats.org port 15521

Flag: grey{prefix_operation_is_easy_to_evaluate_right_W2MQshAYVpGVJPcw}
