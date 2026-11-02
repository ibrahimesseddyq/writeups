# babyre

**CTF:** WolvCTF 2024  
**Category:** rev  
**Original writeup:** <https://nopedawn.github.io/posts/ctfs/2024/wolv-ctf-2024/> — by **nopedawn**. All credit goes to the original author; this is an archived copy.

---

Overview Author: iqlusion

Description: Just a wee-little baby re challenge.

babyre

The next beginner challenge is Rev called babyre

prompt $ ./babyre How are you today? I ' m good Not sure how to react to that :/ Did you checkout our cool sponsors? You should totally check them out! I think I am forgetting something... Oh yea the flag! But where did I put it... I know its in here somewhere. Dratts! Can you help me find it? I swear its around here somewhere.

We can check on dissasembler like IDA Pro in strings segments

or simply strings it, we can get the flag

$ strings babyre | grep wctf wctf { n1c3_oNe_y0u_Found_m3 }

FLAG wctf{n1c3_oNe_y0u_Found_m3}
