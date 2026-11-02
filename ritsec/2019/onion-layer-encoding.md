# Onion Layer Encoding

**CTF:** RITSEC 2019  
**Category:** misc  
**Original writeup:** <https://ctftime.org/writeup/17179> — by **ctftime.org**. All credit goes to the original author; this is an archived copy.

---

> Encoding is not encryption, but what if I just encode the flag with base16,32,64? If I encode my precious flag for 150 times, surely no one will be able to decode it, right?

We have a file `onionlayerencoding.txt`, which is encoded in base16,32,64 150 times. We don't know the order, so we guess which decoding algorithm to use based on the characters of the string to the decode.

```

#!/usr/bin/env python3

from base64 import b64decode, b32decode, b16decode

from base64 import b64encode, b32encode, b16encode

import string

import sys

b64table = list(string.digits + string.ascii_uppercase + string.ascii_lowercase + '+/=')

b32table = list(string.ascii_uppercase + '234567=')

b16table = list(string.hexdigits.upper())

def is_b16(s):

return set(list(s)).issubset(set(b16table))

def is_b32(s):

return set(list(s)).issubset(set(b32table))

def is_b64(s):

return set(list(s)).issubset(set(b64table))

def main():

f = open("./onionlayerencoding.txt").read().encode()

for i in range(150):

try:

if is_b16(f.decode()):

sys.stdout.write('\rb16, i : ' + str(i))

f = b16decode(f)

continue

except:

pass

try:

if is_b32(f.decode()):

sys.stdout.write('\rb32, i : ' + str(i))

f = b32decode(f)

continue

except:

pass

try:

if is_b64(f.decode()):

sys.stdout.write('\rb64, i : ' + str(i))

f = b64decode(f)

continue

except:

pass

print("

" + f.decode())

if __name__ == '__main__':

main()

```

Output:

`RITSEC{0n1On_L4y3R}`
