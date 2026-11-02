# ReService

**CTF:** GreyCatTheFlag 2023 Quals  
**Category:** rev  
**Original writeup:** <https://jh123x.com/blog/2023/greyctf23-reservice/> — by **jh123x.com**. All credit goes to the original author; this is an archived copy.

---

This is the author’s writeup for the challenges ReService in the rev category.

This describes the intended solution when I made the challenge.

Challenge Description #

I found this file that was installed by the virus. Can you find out what it does?

It seems to connect to a c2 server and makes use of the current time.

Junhua

A distributable file called malware is also provided.

My Notes #

This is mean to be an easy question for golang rev veterans.

My intended solution is not to actually reverse the binary but to make use of the information provided in the challenge description

I’ve also consulted my other Rev CTF players to see if it is solvable and it seems like it was quite solvable.

I’ll be going through the fastest solution that does not actually involve reversing.

Notes to future self when setting up this challenge. #

Do not name a distributable malware

Analyzing the challenge description #

The challenge description mentions that the malware connects to a c2 server and makes use of the current time.

This means that the “malware” connects to some remote server and fetches some information from it.

Analyzing the file #

Instead of using static analysis, I’ll make use of dynamic analysis and check the types of information it collects from the C2 server.

This can be done using wireshark.

By loading up wireshark and running the malware. We can see that the malware connects to the remote server using TCP.

By following the stream, we can get the flag

Wireshark result from malware executable

Spoilers for those who are attempting the reverse route #

Below is the original binary for malware
