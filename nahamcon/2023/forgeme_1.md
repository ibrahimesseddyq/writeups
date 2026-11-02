# ForgeMe_1

**CTF:** NahamCon 2023  
**Category:** crypto  
**Original writeup:** <https://meashiri.github.io/ctf-writeups/posts/202306-nahamcon/> — by **meashiri**. All credit goes to the original author; this is an archived copy.

---

Forge Me 1¶

Can you break my hashed-based MAC? Below are some resources that may be useful.

The challenge description helpfully provides the link to the SHA1 RFC and a clue about the length extension attack. Further exploring the length extension attack on SHA1 to achieve HMAC forgery. Forgery refers to the fact that we can take a valid message and its tag and predict a new message and its tag, that can be validated. Here is a good explanation of it :

We are given two source files:

crypto.py : This seems to be a standard implementation of the SHA1 hashing algorithm.

: This seems to be a standard implementation of the SHA1 hashing algorithm. oracle.py : This is the implementation of the hashing oracle, which implements the following methods:

Can you break my hashed-based MAC? What would you like to do ? ( 1 ) MAC Query ( 2 ) Verification Query ( 3 ) Forgery ( 4 ) Exit Choice:

The plan of attack is the use Option #1 to generate a valid tag for a sample message, and then apply a length extension attack to create a new message, with a predictable tag and invoke Option #3 to verify the forgery. If we are successful, we would get the flag.

KEYLEN = 64 TAGLEN = 20 # SHA1() tag is 20 bytes ... # H(key || msg) == tag def vf_query ( conn , key , first_part = None ): conn . sendall ( b 'msg (hex): ' ) msg = conn . recv ( 1024 ) . strip () conn . sendall ( b 'tag (hex): ' ) tag = conn . recv ( 1024 ) . strip () try : m = unhexlify ( msg ) calc_t = hashed_mac ( msg , key ) print ( f "Msg: { msg } --> { calc_t } with key: { key } " ) msg = m # The message must inlude a given part in it. This would be the genuine message, with a known-to-be-good tag. if first_part is not None and first_part not in msg : conn . sendall ( f 'Yoinks scoobs! Ya gotta include the first part

' . encode ()) elif len ( tag ) != TAGLEN * 2 : conn . sendall ( f 'Invalid tag length. Must be { TAGLEN } bytes

' . encode ()) else : t_ret = hashed_mac ( msg , key ) return t_ret . encode () == tag , msg except Exception as e : conn . sendall ( b 'Invalid msg format. Must be in hexadecimal

' ) return False , b ''

Searching for previous CTF writeups for SHA1 Length Extension Attacks, led me to this pure python implementation of the algorithm. The key method in the module is:

extend ( appendData , knownData , secretLength , startHash ): # Performs a hash length extension attack. Returns the bytestring to # use when appending data.

Putting it all together, the approach is as follows:

Start with an arbitrary known message and get its tag from the oracle. Use the hash length extention module to extend the message with the required text b'I guess you are just gonna have to include this!' ` and get the predicted hash, using the known keylength=64. Submit the extended message and the predicted tag as the proof of forgery and get the flag.

The complete solution is similar to this:

msg = b 'bazooka' keylen = 64 msg_hex = binascii . hexlify ( msg ) # I used the given signature as the extension, you can use it as the original message too. ext = b 'I guess you are just gonna have to include this!' # get the tag for a known message C = remote ( 'challenge.nahamcon.com' , 31837 ) C . recvuntil ( b 'Choice: ' ) C . sendline ( b '1' ) C . recvuntil ( b 'hex): ' ) C . sendline ( msg_hex ) C . recvuntil ( b 'H(key || msg):' ) first_tag = C . recvline () . strip () print ( f " { msg =} tag= { first_tag } " ) # extend the message and predict the tag sha = hlextend . new ( 'sha1' ) query = sha . extend ( ext , msg , keylen , first_tag . decode ()) umac = sha . hexdigest () print ( f " { query =}

{ umac =} " ) # submit the extended message and the tag as the proof of forgery C . recvuntil ( b 'Choice: ' ) C . sendline ( b '3' ) C . recvuntil ( b 'msg (hex):' ) C . sendline ( binascii . hexlify ( query )) C . recvuntil ( b 'tag (hex):' ) C . sendline ( umac . encode ()) # get the flag C . interactive ()
