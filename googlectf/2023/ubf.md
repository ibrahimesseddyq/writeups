# ubf

**CTF:** GoogleCTF 2023  
**Category:** pwn  
**Original writeup:** <https://ik0ri4n.de/google-ctf-23/> — by **ik0ri4n.de**. All credit goes to the original author; this is an archived copy.

---

UBF

rev / UBF 52 solves / 197 points Please review and test this Unnecessary Binary Format (UBF)! Connection: ubf.2023.ctfcompetition.com 1337 Flag: _____________________________________ ubf.zip

TL;DR: the challenge is a UBF parser with an out of bounds write vulnerability in boolean arrays because of missing header checks. This vulnerability allows us to overwrite string data and get the uncensored flag.

The provided binary is an unpacking utility for a custom binary data format. It accepts a blob of base64-encoded data, tries to interpret this data as its custom format and unpack it and then prints the unpacked data or an error message as a result.

The binary format contains at least one block of data, so called entries, that must contain a header of a certain length and structure and data depending on its type. UBF supports three data types:

String arrays: contain a short array of lengths and then the raw string data Support environment variable expansion via $NAME Flag strings are later censored before printing

contain a short array of lengths and then the raw string data Int arrays: contain raw integer bytes as data

contain raw integer bytes as data Boolean arrays: contain raw boolean bytes as data

Now, the header contains a bunch of type and size information, some of them redundant:

The initial size of the data buffer for the parsed entry

The type byte, 's' for strings, 'i' for integers and 'b' for booleans

for strings, for integers and for booleans The number of array entries

The raw length of the data

Now, there are a few relevant checks, especially for string arrays, to avoid data corruption:

The raw length must be the size of the raw data entry (the shorts containing the length for string arrays) multiplied with the number of array entries.

The initial size of the entry buffer must fit at least the raw length. -The copied data may not exceed the input data

All these checks seem to be correctly implemented for string arrays but integer arrays and boolean arrays simply ignore the specified raw length. Now, @ju256 (with whom I worked on this challenge) noted the suspiciously named method fix_corrupt_booleans() . This method converts up to the number of entries of bytes to real booleans after the copied data while respecting the initial size of the data buffer. Now, this method may fix the boolean data that was copied, but only if the specified raw length is zero. Since this value is never checked though and the only bounds check on this fix is an upper bounds check we can use negative values to abuse this out of bounds (OOB) heap write vulnerability.

void censor_string ( char * input , int length ) { if ( ( ( ( 5 < length ) && ( * input == 'C' ) ) && ( input [ 1 ] == 'T' ) ) && ( ( input [ 2 ] == 'F' && ( input [ 3 ] == '{' ) ) ) ) { memset ( input + 4 , L 'X' , ( long ) ( length + - 5 ) ) ; } return ; }

Our ultimate goal for this challenge is to use variable expansion to get the flag string and somehow avoid it being censored. Because of the check in the censor_string() function we can do so by first sending a string array and then a boolean array and using the vulnerability we found to change one of the 'CTF{' -bytes to 0x01 . Even better, we can overwrite the length short at the start of the data buffer with 0x01 which disables the check in this method and still gives us the whole string because it is printed with snprintf() and %s ignoring the supposed length. To consistently place the two allocated heap blocks for the entries at a fixed offset from each other on the stack we can set the same initial buffer size. This is because heap allocation strategies typically place blocks of the same size together to utilize memory space as best as possible. See the memory dump below to understand the offset:

The final solution simply builds such a payload with a working offset and sends the base64-encoded data:
