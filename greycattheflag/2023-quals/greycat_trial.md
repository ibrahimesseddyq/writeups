# GreyCat_Trial

**CTF:** GreyCatTheFlag 2023 Quals  
**Category:** crypto  
**Original writeup:** <https://hong5489.github.io/2023-05-22-greyctf2023/> — by **hong5489**. All credit goes to the original author; this is an archived copy.

---

GreyCat Trial

Description

Attachment

We are given a python script, we need to pass 3 trials to get the flag

We need to input 2 numbers:

a = int ( input ( "The first element: " )) b = int ( input ( "The second element: " )) all_seeing_number = 23456789

First Trial

# FIRST TRIAL if b <= 0 : print ( "Verily, those who would cheat possess not the might of true wizards." ) exit ( 0 ) if pow ( all_seeing_number , a - 1 , a ) != 1 : print ( "Alas, thy weakness hath led to defeat in the very first trial." ) exit ( 0 )

As you can see, we need to input the a value to satisfy the equation \(23456789^{a-1} \equiv 1 \mod a\)

If you’re familiar with math, you will notice this is Fermat’s little theorem:

\[a^{p-1} \equiv 1 \mod p\]

It means any prime number will satisfy the equation above, so we just input a prime number will pass the first test!

Second Trial

# SECOND TRIAL trial_numbers = [ randint ( 0 , 26 ) for i in range ( 26 )] for number in trial_numbers : c = a + b * number if pow ( all_seeing_number , c - 1 , c ) != 1 : print ( "Thou art not yet strong enough, and thus hast been vanquished in the second trial" ) exit ( 0 )

This was hard to satisfy, because the trial_number is 0 to 26 random pick 26 numbers, and the value c has to be prime number?

Well after I read the wikipedia for Fermat’s little theorem notice pseudoprime/Carmichael number will also satisfy the equation

Then I look at the wiki of Carmicheal number stated:

Thomas Wright proved that if \(a\) and \(m\) are relatively prime, then there are infinitely many Carmichael numbers in the arithmetic progression \(a+k \cdot m\), where \(k=1,2,...\)

Which was quite similar for the second trial \(a+b \cdot n\), but I cannot find any example for this statement..

Closest I found was this article, but I tested the success rate is low (we need to satisfy 26 times for trial numbers)

Solving

So I just solve it by bruce forcing the value of \(a\) and \(b\), we know \(a\) must be prime so I copy the first 1-1000 prime numbers and brute force it with python script:

for b in range ( 1 , 1000 ): for a in [ 2 , 3 , 5 , 7 , 11 , 13 , 17 , 19 , 23 , 29 , 31 , 37 , 41 , 43 , 47 , 53 , 59 , 61 , 67 , 71 , 73 , 79 , 83 , 89 , 97 , 101 , 103 , 107 , 109 , 113 , 127 , 131 , 137 , 139 , 149 , 151 , 157 , 163 , 167 , 173 , 179 , 181 , 191 , 193 , 197 , 199 , 211 , 223 , 227 , 229 , 233 , 239 , 241 , 251 , 257 , 263 , 269 , 271 , 277 , 281 , 283 , 293 , 307 , 311 , 313 , 317 , 331 , 337 , 347 , 349 , 353 , 359 , 367 , 373 , 379 , 383 , 389 , 397 , 401 , 409 , 419 , 421 , 431 , 433 , 439 , 443 , 449 , 457 , 461 , 463 , 467 , 479 , 487 , 491 , 499 , 503 , 509 , 521 , 523 , 541 , 547 , 557 , 563 , 569 , 571 , 577 , 587 , 593 , 599 , 601 , 607 , 613 , 617 , 619 , 631 , 641 , 643 , 647 , 653 , 659 , 661 , 673 , 677 , 683 , 691 , 701 , 709 , 719 , 727 , 733 , 739 , 743 , 751 , 757 , 761 , 769 , 773 , 787 , 797 , 809 , 811 , 821 , 823 , 827 , 829 , 839 , 853 , 857 , 859 , 863 , 877 , 881 , 883 , 887 , 907 , 911 , 919 , 929 , 937 , 941 , 947 , 953 , 967 , 971 , 977 , 983 , 991 , 997 ]: count = 0 for i in range ( 27 ): c = a + b * i if pow ( 23456789 , c - 1 , c ) == 1 : count += 1 if count > 20 : print ( a , b ) print ( count ) # 3 2 # 22 # 5 2 # 21 # 7 2 # 21 # 11 2 # 21 # 13 2 # 21

From the result we can see the higest success rate is 3 and 2 , so I made a while true loop to keep sending 3

2 until it return the flag!

while true ; do echo -e "3

2" | nc 34.124.157.94 10592| grep grey ; done # grey{Gr33N-tA0_The0ReM_w1z4rd}

Flag

grey{Gr33N-tA0_The0ReM_w1z4rd}
