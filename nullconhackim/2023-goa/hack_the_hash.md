# Hack_the_Hash

**CTF:** nullconHackIM 2023 Goa  
**Category:** pwn  
**Original writeup:** <https://github.com/CSecIITB/nullcon-HackIM-CTF-2023-Writeups/tree/main/pwn/HacktheHash> — by **CSecIITB**. All credit goes to the original author; this is an archived copy.

---

# Hack the Hash

The binary is a stripped binary. The function at `0x000013d6` is the main function.

The function at `0x00001329` ensures that username is not gehaxelt and compares hash of password with a hard coded hash.

The function creates a 0x12 byte space for storing the SHA-1 hash of the password entered and a variable which stores 0 if the user is authorized.

The variable is initialized with 0x1337, which makes first 2 bytes of variable 0.

If we enter a password with SHA-1 hash ending in 2 0 bytes, the check variable is overwritten with 0, thus authorizing the user.

One may use the following script to get such a password.

```python

i=0

while True:

if sha1(int.to_bytes(i,(int)(i**0.5+1),'little')).digest()[-2:] == b'\x00\x00':

print(int.to_bytes(i,(int)(math.log(i)/math.log(256))+1,'big'))

break

i+=1

```

We get the flag:

**ENO{C_H4SHing_1s_H4rd:D}**
