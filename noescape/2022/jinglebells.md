# JingleBells

**CTF:** NOESCAPE 2022  
**Category:** crypto  
**Original writeup:** <https://www.linkedin.com/pulse/escape-ctf-2022-write-ups-tyron-parry/> — by **linkedin.com**. All credit goes to the original author; this is an archived copy.

---

Jingle Bells

Oh, jingle bells, jingle bells Jingle all the way Oh, what fun it is to ride In a one horse open sleigh Jingle bells, jingle bells Jingle all the way Oh, what fun it is to ride In a one horse open sleigh Dashing thr...





Oh Sorry! i think you need flag, here's the file. Go & look for it, All the best.

We are given a zip file to download, when we extract we can see it extracts 2 files. Lets unzip and take a look at the flag file.

Well this is not, but there was another file .DS_STORE, lets have a look at that.

Alright, we have a bunch of hex, lets clean this up and just get the hex, so 41 52 50 47 53 7b 51 40 66 55 31 61 74 5f 67 75 65 42 68 74 75 5f 67 55 33 5f 24 61 30 6a 7d you can go to CyberChef, but I did this in the command line using xxd.

The -r is to convert from hex to ASCII and the -p is used to print out the result. So now we have something resembling a flag ARPGS{Q@fU1at_gueBhtu_gU3_$a0j}, there was alot of ROT13 in this CTF so far, what if it is the same. Go to CyberChef with this and select ROT13 and you will get the flag.

Flag = NECTF{D@sH1ng_thrOugh_tH3_$n0w}
