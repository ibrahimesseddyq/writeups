# zipzipzipzip

**CTF:** TCP1P 2023  
**Category:** misc  
**Original writeup:** <https://ctftime.org/writeup/38009> — by **ctftime.org**. All credit goes to the original author; this is an archived copy.

---

# Misc: zipzipzipzip

## Task:

```

Author: botanbell

unzip me pls

```

## Solution:

Attached to the challenge is a password.txt and a file called zip-25000.zip.

The zip-25000.zip file is password-encrypted, so you must unlock it with the password contained in password.txt

Upon unzipping zip-25000.zip, you will soon figure out that inside the unzipped folder, there is a file called password.txt and zip-24999.zip.

This usually means that you'll have to do this process 24999 more times, which, if you do it manually, and assuming you take 5 seconds to unzip one zip file, it would take ~35 hours of nonstop unzipping.

That's crazy. No one wants to do that. (unless you're crazy)

So obviously you want to make a script to automate it. Some used python to solve it, but I used bash (only because it's simpler)

My thought process was to answer all of the following questions:

* How do I automate it?

* How do I loop it?

* Is there a way to make this process more efficient?

From the questions above, I knew I had to implement these:

* Read password, use it on zip

* Unzip

* Loop 25000 times

* Delete the original file (this idea came up later on because I realized it would clog up my storage)

So I made this bash script to help me

```bash

#!/bin/bash

j=0

for i in {1..25000}

do

unzip -P $(strings password.txt) -o $(ls *.zip)

rm zip-$((25000-$j)).zip

j=$((j+1))

done

```

Unzipping all of them gives a flag.txt, which reads:

## Flag:

```

TCP1P{1_TH1NK_U_G00D_4T_SCR1PT1N9_botanbell_1s_h3r3^_^}

```

## Notes:

I believe this code isn't fully optimized, as it took me ~10 minutes to fully unzip all of them.

(i'm still learning)
