# eaxy

**CTF:** NahamCon 2021  
**Category:** crypto  
**Original writeup:** <https://h0j3n.gitbook.io/ctf/writeup/ctf-writeup/nahamcon-ctf-2021> — by **h0j3n.gitbook.io**. All credit goes to the original author; this is an archived copy.

---

Eaxy

The spelling should be easy but why eaxy ? So xor come out :) Trying with Cyberchef give us this

Copy Key = 66: The XOR key you used to find string this is the 0 character index of the flag #More information 66 = f - Which means that we need to try and get the correct key for the next sentence.

Like usual script all the wayyy!

Copy key = [ '0' , '1' , '2' , '3' , '4' , '5' , '6' , '7' , '8' , '9' , 'a' , 'b' , 'c' , 'd' , 'e' , 'f' , 'g' , 'h' , 'i' , 'j' , 'k' , 'l' , 'm' , 'n' , 'o' , 'p' , 'q' , 'r' , 's' , 't' , 'u' , 'v' , 'w' , 'x' , 'y' , 'z' , '{' , '}' ] flag = [ "-" ] * 38 for keys in key : b = bytearray ( open ( 'eaxy' , 'rb' ). read ()) for i in range ( len (b)): b [ i ] ^= int ( hex ( ord (keys)), 16 ) if "The XOR key you used" in str (b): for i in str (b). split ( "this is the " ) [ 1 : ] : flag [ int (i[ 0 : 2 ])] = str (keys) print ( "" . join (flag))
