# echo

**CTF:** GoogleCTF 2020  
**Category:** misc  
**Original writeup:** <https://github.com/saaramar/echo_googlequals2020> — by **saaramar**. All credit goes to the original author; this is an archived copy.

---

This is my solution for the echo pwn challenge from Google Quals CTF 2020. Since this challenge doesn't involve an unusual vulnerability/exploit, I won't do a full writeup on it, but I will drop the high level plan and exploit for fun:) I'll skip the introduction to the challenge, its interface and how it works - all of that is straightforward, please check out the [challenge](https://ctftime.org/task/12838) itself:)

As it turns out, the solution I have shares many similar concepts with the great solution published by *RedSocket* (check out their awesome [writeup](http://blog.redrocket.club/2020/08/30/google-ctf-quals-2020-echo/)!). For example, filling the holes created by a huge allocation of std::string by allocating sizes of decreasing powers of 2 is a classic trick we both use. However, the approach I took triggered the vulnerability twice instead of three times. Generally speaking I tend to prefer to trigger bugs as few times as possible in my exploits - in some cases this results in dramatic increase in the exploit’s stability. Specifically here we are talking about a CTF challenge with a very stable bug to exploit, so it doesn’t matter much. Anyways, it's all about the cool tricks and concepts we can do and learn! :)

This is another great opportunity to thank all of the Google CTF authors for an amazing CTF and great time, as in every year :) Keep up the fantastic work!
