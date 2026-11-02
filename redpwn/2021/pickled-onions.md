# pickled-onions

**CTF:** redpwn 2021  
**Category:** rev  
**Original writeup:** <https://irissec.xyz/articles/categories/re/2021-07-12/pickled-onions> — by **irissec.xyz**. All credit goes to the original author; this is an archived copy.

---

pickled-onions

2021-07-12

by not_really Kevin Higgs told me pickles can do anything! Author: KyleForkBomb Files: pickled.py

Recursive python pickle files

Depth 1 (pickled.py):

__import__ ( 'pickle' ). loads ( b '(I128...' )

At this layer we get a long string loading a long string. Whenever pickle loads it, it starts to execute code. Python comes with a pickle disassembler, so let’s try using that.

1. Write to file

t = b '(I128...)' f = open ( "pickled_l2.dat" , "wb" ) f . write ( t ) f . close ()

python -m pickletools pickled_l2.dat > pickled_l2.pkl

The stack not empty after STOP is normal.

Depth 2 (pickled_l2.pkl):

0: ( MARK 1: I INT 128 6: I INT 4 ... 544265: I INT 82 544269: I INT 46 544273: t TUPLE (MARK at 0) 544274: p PUT 69420 544281: c GLOBAL 'pickle loads' 544295: c GLOBAL 'builtins bytes' 544311: g GET 69420 544318: \x85 TUPLE1 544319: R REDUCE 544320: \x85 TUPLE1 544321: R REDUCE 544322: . STOP

‘pickle loads’ looks like it’s loading another pickle file, so let’s extract that into another file. I found it easiest to go into the original pickled_l2.dat file and replace all I with nothing, all newlines with , , then throw it into a python script and write out the contents.

1. Write to file

a = [ 128 , 4 , 99 , 112 , 105 , 99 , 107 , 108 ,..., 82 , 46 ] f = open ( "pickled_l3.dat" , "wb" ) f . write ( bytearray ( a )) f . close ()

python -m pickletools pickled_l3.dat > pickled_l3.pkl

Depth 3 (pickled_l3.pkl):

The first half of the file has some constants and functions in pickle. You could decode them with pickletools, but it’s pretty obvious what they’re doing:

def pickledhorseradish (): return pickledmacadamia . __add__ ( pickledbarberry ) def pickledcoconut (): return pickledmacadamia . __sub__ ( pickledbarberry ) def pickledlychee (): return pickledmacadamia . __xor__ ( pickledbarberry ) def pickledcrabapple (): return pickledmacadamia . __eq__ ( pickledbarberry ) def pickledportabella (): return pickledmacadamia . __ne__ ( pickledbarberry ) def pickledquince (): return pickledmacadamia . __le__ ( pickledbarberry ) def pickledeasternmayhawthorn (): return pickledmacadamia . __ge__ ( pickledbarberry ) def pickledmonstera (): return 1 def pickledcorneliancherry (): return 0 def pickledalligatorapple (): #??? def pickledboysenberry (): #very long function

Before we move onto the boysenberry function, we can also find that the input string is pickledximenia and its length should be 64:

119509: \x86 TUPLE2 119510: \x8c SHORT_BINUNICODE 'pickledximenia' 119526: c GLOBAL 'builtins input' 119542: \x8c SHORT_BINUNICODE 'What is the flag? ' 119562: \x85 TUPLE1 119563: R REDUCE ... 119626: ( MARK 119627: \x8c SHORT_BINUNICODE 'pickledburmesegrape' 119648: c GLOBAL 'io pickledximenia.__len__' 119675: ) EMPTY_TUPLE 119676: R REDUCE ... 119784: \x8c SHORT_BINUNICODE 'io' 119788: c GLOBAL 'io pickledgarlic.__getitem__' 119818: c GLOBAL 'io pickledburmesegrape.__eq__' 119849: I INT 64 119853: \x85 TUPLE1 119854: R REDUCE

1. Write to file

a = b ' \x80\x04 cpickle

io

(...dcedarbaycherry

\x85 R.' f = open ( "pickled_l4.dat" , "wb" ) f . write ( a ) f . close ()

python -m pickletools pickled_l4.dat > pickled_l4.pkl

Depth 4 (pickled_l4.pkl):

There’s a lot more pickle “programs” here, but really the one that has the start of the fun is in pickledvoavanga . It doesn’t have a header, which is added from pickledalligatorapple . Instead of trying to understand what it was doing, I just called the original script, gave it a 64 length string, and printed the value.

>>> exec ( open ( "chall.py" ). read ()) What is the flag ? aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa Nope ! >>> import io >>> io . pickledalligatorapple b ' \x80\x04 I97

I97

I97

I97

I97

I97

I97

I97

I97

I97

I97

I97

I97

I97

I97

I97

I97

I97

I97

I97

I97

I97

I97

I97

I97

I97

I97

I97

I97

I97

I97

I97

I97

I97

I97

I97

I97

I97

I97

I97

I97

I97

I97

I97

I97

I97

I97

I97

I97

I97

I97

I97

I97

I97

I97

I97

I97

I97

I97

I97

I97

I97

I97

I97

'

So it seems to pass in our string as INT values. We can dump these values and then take a look at the first function, pickledvoavanga (with pickledalligatorapple prepended).

Depth 5 (pickledvoavanga.pkl)

At the start, we see the ints from pickledalligatorapple , but also some pops/puts. This looks like the function is selecting which of the characters to use from the input.

0: \x80 PROTO 4 2: I INT 97 6: I INT 97 ... 246: I INT 97 250: I INT 97 254: I INT 97 258: 0 POP 259: 0 POP ... 265: 0 POP 266: 0 POP 267: p PUT 1 270: 0 POP 271: 0 POP ... 318: 0 POP 319: 0 POP 320: p PUT 0 323: 0 POP 324: 0 POP 325: 0 POP 326: 0 POP 327: 0 POP

Then there’s the other logic

328: c GLOBAL 'pickle io' 339: ( MARK 340: \x8c SHORT_BINUNICODE 'pickledmacadamia' 358: g GET 0 361: \x8c SHORT_BINUNICODE 'pickledbarberry' 378: g GET 1 381: \x8c SHORT_BINUNICODE 'pickledgarlic' 396: \x8c SHORT_BINUNICODE 'pickledcorneliancherry' 420: \x8c SHORT_BINUNICODE 'pickledarugula' 436: \x86 TUPLE2 437: d DICT (MARK at 339) 438: b BUILD 439: c GLOBAL 'pickle loads' 453: \x8c SHORT_BINUNICODE 'io' 457: c GLOBAL 'io pickledgarlic.__getitem__' 487: c GLOBAL 'pickle loads' 501: c GLOBAL 'io pickledeasternmayhawthorn' 531: \x85 TUPLE1 532: R REDUCE

Looks like a lot of these names are from the functions from pickled_l3.pkl . Let’s try to decode some of these names to better understand what’s going on.

pickledmacadamia - left argument pickledbarberry - right argument pickledgarlic - tuple of these two values: pickledcorneliancherry - return 0 pickledarugula - next function in pickled_l4.pkl pickledeasternmayhawthorn - pickledmacadamia >= pickledbarberry

So it looks like one character of the string is compared with another, and if it’s valid, it continues on to the next function. We can write a script to read these functions, but there are actually two types of functions. Let’s take a look at pickledleeks , the fourth function called.

327: 0 POP 328: c GLOBAL 'pickle io' 339: ( MARK 340: \x8c SHORT_BINUNICODE 'pickledmacadamia' 358: g GET 0 361: \x8c SHORT_BINUNICODE 'pickledbarberry' 378: g GET 1 381: \x8c SHORT_BINUNICODE 'pickledgarlic' 396: \x8c SHORT_BINUNICODE 'pickledcorneliancherry' 420: \x8c SHORT_BINUNICODE 'pickledjocote' 435: \x86 TUPLE2 436: d DICT (MARK at 339) 437: b BUILD 438: c GLOBAL 'pickle loads' 452: \x8c SHORT_BINUNICODE 'io' 456: c GLOBAL 'io pickledgarlic.__getitem__' 486: c GLOBAL 'pickle io' 497: ( MARK 498: \x8c SHORT_BINUNICODE 'pickledmacadamia' 516: c GLOBAL 'pickle loads' 530: c GLOBAL 'io pickledlychee' 548: \x85 TUPLE1 549: R REDUCE 550: \x8c SHORT_BINUNICODE 'pickledbarberry' 567: I INT 0 570: d DICT (MARK at 497) 571: b BUILD 572: 0 POP 573: c GLOBAL 'pickle loads' 587: c GLOBAL 'io pickledcrabapple' 608: \x85 TUPLE1

This one xors the first value with the second value, then compares it with the constant (0 on line 567). So we have to account for two types of checks being made.

1. Dump all pickle functions script

Recursively dumps pickle “functions” starting at pickled_l3.pkl .

import subprocess def it ( s ): tmpName = f " { s } .dat" asmName = f " { s } .pkl" subprocess . run ([ "python" , "-m" , "pickletools" , tmpName , "-o" , asmName ]) f = open ( asmName , "r" ) lines = f . readlines () content = [ x . rstrip ( "

" ) for x in lines ] name = "idkwhatthelastnamewas" lastLine = "" #print(f"opening {asmName}") for line in lines : if "SHORT_BINUNICODE" in line : name = line . split ( "SHORT_BINUNICODE '" , 1 )[ 1 ][: - 2 ] elif "SHORT_BINBYTES" in line or "BINBYTES" in line : dataName = f "__tmp_ { name } .dat" df = open ( dataName , "wb" ) s = line . split ( " b'" , 1 )[ 1 ][: - 2 ] if "pickledalligatorapple" in lastLine : s = " \\ x80 \\ x04I97 \\ nI97 \\ nI97 \\ nI97 \\ nI97 \\ nI97 \\ nI97 \\ nI97 \\ nI97 \\ nI97 \\ nI97 \\ nI97 \\ nI97 \\ nI97 \\ nI97 \\ nI97 \\ nI97 \\ nI97 \\ nI97 \\ nI97 \\ nI97 \\ nI97 \\ nI97 \\ nI97 \\ nI97 \\ nI97 \\ nI97 \\ nI97 \\ nI97 \\ nI97 \\ nI97 \\ nI97 \\ nI97 \\ nI97 \\ nI97 \\ nI97 \\ nI97 \\ nI97 \\ nI97 \\ nI97 \\ nI97 \\ nI97 \\ nI97 \\ nI97 \\ nI97 \\ nI97 \\ nI97 \\ nI97 \\ nI97 \\ nI97 \\ nI97 \\ nI97 \\ nI97 \\ nI97 \\ nI97 \\ nI97 \\ nI97 \\ nI97 \\ nI97 \\ nI97 \\ nI97 \\ nI97 \\ nI97 \\ nI97 \\ n" + s df . write ( eval ( "b'" + s + "'" )) df . close () it ( name ) lastLine = line it ( "pickled_l3" )

2. Parse all pickle functions and export to z3 script

This script does some wacky hacky text file parsing to generate conditions. Simple to do by hand, but I just felt like making a script. Expects the pickle “functions” from pickled_l4.pkl to be disassembled to {functionname}.pkl

#quality code obv def it ( s ): asmName = f " { s } .pkl" f = open ( asmName , "r" ) lines = f . readlines () content = [ x . rstrip ( "

" ) for x in lines ] name = "idkwhatthelastnamewas" lastLine = "" pushing = True popping = False coding = False fakeStack = [] var0 = 0 var1 = 0 mode = 0 #0 - entered first mark #1 - exited first mark #2 - entered second mark #3 - exited second mark mark = - 1 poppedValue = - 1 curCompName = "idkCompName" curOpName = "idkOpName" nextFunName = "dunnoNextFunName" for line in lines : if pushing : if " \\ x80 PROTO" in line : pass elif "I INT" in line : fakeStack . append ( len ( fakeStack )) else : pushing = False popping = True if popping : if "POP" in line : poppedValue = fakeStack . pop () elif "PUT 0" in line : var0 = fakeStack [ - 1 ] #poppedValue elif "PUT 1" in line : var1 = fakeStack [ - 1 ] else : popping = False coding = True if "SHORT_BINUNICODE" in line : if "pickledmacadamia" in line : pass elif "pickledbarberry" in line : pass elif "pickledgarlic" in line : pass elif "pickledcorneliancherry" in line : pass else : if mode == 0 : nextFunName = line . split ( "SHORT_BINUNICODE '" , 1 )[ 1 ][: - 2 ] mode = 1 elif "( MARK" in line : mark += 1 elif "b BUILD" in line : mark += 1 elif mark == 2 and "c GLOBAL" in line : if "GLOBAL 'pickle loads'" in line : pass else : curOpName = line . split ( "GLOBAL 'io " , 1 )[ 1 ][: - 2 ] elif mark == 2 and "I INT" in line : curOpValue = line . split ( "INT " , 1 )[ 1 ][: - 1 ] elif ( mark == 1 or mark == 3 ) and "c GLOBAL" in line : if "GLOBAL 'io pickledgarlic.__getitem__'" in line : pass elif "GLOBAL 'pickle loads'" in line : pass elif "GLOBAL 'pickle io'" in line : pass else : curCompName = line . split ( "GLOBAL 'io " , 1 )[ 1 ][: - 2 ] if curOpName == "pickledhorseradish" : curOpName = "+" elif curOpName == "pickledcoconut" : curOpName = "-" elif curOpName == "pickledlychee" : curOpName = "^" if curCompName == "pickledcrabapple" : curCompName = "==" elif curCompName == "pickledportabella" : curCompName = "!=" elif curCompName == "pickledquince" : curCompName = "<=" elif curCompName == "pickledeasternmayhawthorn" : curCompName = ">=" if mark == 3 : #has operation print ( f "s.add((arr[ { var0 } ] { curOpName } arr[ { var1 } ]) { curCompName } { curOpValue } ) # { s } " ) else : print ( f "s.add(arr[ { var0 } ] { curCompName } arr[ { var1 } ]) # { s } " ) it ( nextFunName ) it ( "pickledvoavanga" )

3. Z3 script
