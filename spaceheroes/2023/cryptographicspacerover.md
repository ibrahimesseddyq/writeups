# CryptographicSpaceRover

**CTF:** SpaceHeroes 2023  
**Category:** crypto  
**Original writeup:** <https://github.com/PointerPioneers/CTFs/blob/main/Space%20Heroes%20CTF%202023/Write-ups/crypto%20Cryptographic%20Space%20Rover/README.md> — by **PointerPioneers**. All credit goes to the original author; this is an archived copy.

---

# Cryptographic Space Rover

From the description, we find that we're dealing with a Python script that prints the CPU usage and other system information.

To make sure that the reader fully understood it, it sends this information multiple times.

The description also mentions that processes will be spawned for certain characters at certain indices, and we're supposed to find which characters do this.

## The script

Looking at the script, we find some more information: we're assigned a random UUID that is given to the user, and the user must input some characters as a guess.

It then iterates over both the characters of the flag and the characters of the user guess simultaneously, so we start with the first character of both, then the second, etc.

For each iteration, it compares the two characters, and passes either `True` or `False` to the function named `print_top` based on the result of this comparison.

This function simulates the `top` command in Linux, which prints information about the processes running, the CPU and memory usage, and more.

However, it has one extra feature: if the iterated characters mentioned above are equal, we change the name of the currently running process (the Python process) to the aforementioned UUID, resetting it back to `python3` before returning from the function.

## Testing

In the challenge description, we're given Python code to connect to the remote using pwntools. When executing this code, we're first given a header with our random UUID in it.

Then, when we input for example the guess `abcd`, we see some system information printed 5 times (yes, 5 instead of 4, as we run one more `print_top` when we run out of characters from the guess, in this case the boolean paramater effectively indicates that the characters are not equal).

In this test, we can also see the Python 3 process that we're dealing with in all but one of the outputs. That is because in the 3rd output, corresponding to the `c` of our guess, we see the UUID we were given earlier.

## Solving

From all of this information, we can deduce that when we perform guesses for the flag, the program tells us at which indices the guess was right, and where it was wrong.

The character match is given to us by the name of the Python process: if it's equal to `python`, the characters don't match, but if it is our random UUID, they match.

Although we don't know which process is our Python process, we have a random UUID, so we just need to make sure that that UUID is in the output.

We can automate this as follows: for each character that may be in the output, we send a guess that consists only of that character. This way, we can see which indices in the flag have that character.

We then store this information, which we can use to reconstruct the entire flag.

We did this in the following Python script:

```py

from pwn import *

import re

# During our tests, we found the length by inputting a very long guess, and seeing how much output we get.

length = 39

# Make the guessed flag initially empty (just spaces)

flag = [b' '] * length

# We thought a-z0-9_-{} would probably cover all the characters used in the flag

# No need for upper case here, as their Python script converts the guess to lower case

alphabet = b'abcdefghijklmnopqrstuvwxyz0123456789_-{}'

for j in range(0, len(alphabet)):

# Encode the iterated characters as a bytes object

letter = bytes(chr(alphabet[j]), encoding='ASCII')



# Open the connection

p = remote("spaceheroes-cryptographic-space-rover.chals.io", 443, ssl=True, sni="spaceheroes-cryptographic-space-rover.chals.io")

# Construct the guess consisting of only the iterated letter

guess = letter * length

# Get the UUID from the header using a RegEx match

header = p.recvuntil(b':: ').decode('ASCII')

uuid = re.findall("[0-9a-fA-F]{8}-[0-9a-fA-F]{4}-[0-9a-fA-F]{4}-[0-9a-fA-F]{4}-[0-9a-fA-F]{12}", header)[0]

print(f'uuid: {uuid}')

# Send the guess

p.sendline(guess)

# Check which characters match, and update the known flag

for i in range(length):

x = p.recvuntil(b'top')

if uuid in x.decode('ASCII'):

flag[i] = letter

print(letter, i)

p.close()

# Print out prediction

print(b''.join(flag))

```

Running this script, we get the following output:

```

[+] Opening connection to spaceheroes-cryptographic-space-rover.chals.io on port 443: Done

uuid: 59cc24c0-39bd-46ed-b1a3-c52f14ec56a7

b'a' 21

[*] Closed connection to spaceheroes-cryptographic-space-rover.chals.io port 443

...

[+] Opening connection to spaceheroes-cryptographic-space-rover.chals.io on port 443: Done

uuid: 33e828be-4ccf-46fc-8a54-f9a6407d8dce

[*] Closed connection to spaceheroes-cryptographic-space-rover.chals.io port 443

b' shctf{met30rs_4r3nt_as_b4d_4s_sl0w_cpu'

```
