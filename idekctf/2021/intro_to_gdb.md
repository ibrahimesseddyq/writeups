# Intro_To_GDB

**CTF:** idekCTF 2021  
**Category:** rev  
**Original writeup:** <https://n00bzunit3d.xyz/writeups/idekctf/rev/introtogdb/readme/> — by **n00bzunit3d.xyz**. All credit goes to the original author; this is an archived copy.

---

# TUTORIAL - Intro To GDB

***Writeup with pictures at website - ```https://n00bzunit3d.xyz/writeups/idekctf/rev/introtogdb/readme/```***

This challenge was mostly an intro to using GDB, so it wasn't too tricky. Running ```strings``` or ```rabin2 -z```(for you ultra-efficient people) didn't yield anything. I opened the binary up in Cutter, a static analyzer, and saw a lot of hex strings. Opening it up in Cutter translates the values for us, but even if it didn't, popping the long hex values into an hex-to-ascii converter does the same thing(hex goes in backwards, because the binary's little-endian):

## Flag: ``` idek{m0m_g3t_th3_c4m3rA!} ```
