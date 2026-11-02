# 999 Bottles

**CTF:** RITSEC 2019  
**Category:** pwn  
**Original writeup:** <https://github.com/b0th/CTF/tree/master/RITSECCTF2019/999%20Bottles> — by **b0th**. All credit goes to the original author; this is an archived copy.

---

# 999 Bottles // Writeup

## Problem

*Well, this is embarassing... I've accidentally compiled 999 ELF files with my password somewhere along the line, one character at a time.*

*Solve these in order, each accepting one ASCII character. Keep going...eventually combining these solutions will match the regular expression RITSEC{}*

*Good luck, and thanks for the help!*

Author: INGRESSIVE

## Solution

Every ELF files have two differents output : `Nope!` if the input is wrong and `OK!` if the input is valid. You can check it by decompiling, i used ghidra.

So let's bruteforce all of this program and store the valid inputs.

```python

import os

import subprocess

ascii = '0123456789@ABCDEFGHIJKLMNOPQRSTUVWXYZ^_abcdefghijklmnopqrstuvwxyz{}'

list_ascii = list(ascii)

valid_char = []

for file in os.listdir():

if not 'c.out' in str(file):

continue

for char in list_ascii:

run = subprocess.Popen('echo %s | ./%s' % (str(char) , str(file)), shell=True, stdout=subprocess.PIPE).stdout

byte_output = run.read()

string_output = byte_output.decode()

if 'OK' in string_output:

valid_char.append(str(char))

print('

' + str(''.join(valid_char)))

```

**FLAG** `RITSEC{AuT057v}`
