# Deoxyencoded_Nucleic_Acid

**CTF:** KillerQueen 2021  
**Category:** crypto  
**Original writeup:** <https://ctftime.org/writeup/31191> — by **ctftime.org**. All credit goes to the original author; this is an archived copy.

---

name: Deoxyencoded Nucleic Acid

After getting the text, it was found to be the base sequence of DNA. According to the beginning of flag, it was identified as K, and it was speculated that A stood for 00, T for 01, G for 10, and C for 11

```python

from Crypto.Util.number import *

with open('dna1.txt','r') as f:

s = f.read().replace('T','01').replace('A','00').replace('C','11').replace('G','10')

print(s)

print(long_to_bytes(0b011010110111000101100011011101000110011001111011011010010111010001110011010111110110001001100001011100110110100101100011011000010011000100110001011110010101111101100010011000010111001101100101010111110110011001101111011101010111001001111101))
