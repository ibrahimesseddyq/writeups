# Trailing Bits

**CTF:** Crypto 2020  
**Category:** crypto  
**Original writeup:** <https://blog.cryptohack.org/cryptoctf2020> — by **blog.cryptohack.org**. All credit goes to the original author; this is an archived copy.

---

Trailing Bits

Challenge

The text that includes the flag is transmitted while unfortunately both of its head and tail bits are lost :(

We are given a file containing binary.

Solution

From the challenge description, we can guess that the binary data provided has a couple bits removed from either end. Not to worry however, since the removed bits won’t affect the rest of the data at all.

We know that some bits have been removed, so we can just replace those with some decoy bits, and then try decoding from binary until we get readable text.

from Crypto.Util.number import * flag = open ( 'output.txt' , 'r' ). read (). strip () i = 0 while True : data = long_to_bytes ( int ( flag , 2 ) * 2 ** i ) if b 'CCTF' in data : print ( data ) exit () i += 1

Output:

basic unit of information in computing and digital communications. The name is a portmanteau of binary digit.[1] The bit represents a logical state with one of two possible values. These values are most commonly represented as either 0or1, but other representations such as true/false, yes/no, +/−, or on/off are common. The flag is CCTF{it5_3n0u9h_jU5T_tO_sH1ft_M3} The correspondence between these values and the physical states of the underlying storage or device is a matter of convention, and different assignments may be used even within the same device or program. It may be physically implemented with a two-st

Flag

CCTF{it5_3n0u9h_jU5T_tO_sH1ft_M3}
