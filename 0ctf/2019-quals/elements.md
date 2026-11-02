# Elements

**CTF:** 0CTF 2019 Quals  
**Category:** rev  
**Original writeup:** <https://balsn.tw/ctf_writeup/20190323-0ctf_tctf2019quals/> — by **balsn.tw**. All credit goes to the original author; this is an archived copy.

---

Reverse

Elements

After some static analysis, we can figure out the flag format is flag{xxxxxxxxxxxx-xxxxxxxxxxxx-xxxxxxxxxxxx}. And each part in {} separated by '-' is then hex-decoded and processed into one side of an triangle. The first part ( 391bc2164f0a ) can be figure out in the program.

If the radius of the inscribed circle ( r ) and the radius of the circumscribed circle ( R ) meet the conditions (r = 19400354808065.54 , and R = 47770539528273.91 ), the input is the correct flag.

With gdb, we can easily find out one of the shortest two sides (a = 62791383142154.00 ). Then what we need to do is to calculate the other two sides and convert them back to the original value.

One of our teammates gets the approximate values of the other two sides by a drawing tool. Then we write a python code with higher precision to get the very close values (With equations of Area = a*b*c(4*R) = (a+b+c)*r/2 = sqrt(s*(s-a)*(s-b)*(s-c)) , s = (a+b+c)/2, R is the circumscribed circle radius, and r is the inscribed circle radius).

from decimal import * import math getcontext().prec = 50 r = Decimal( '19400354808065.54' ) R = Decimal( '47770539528273.91' ) a = Decimal( '62791383142154' ) b = Decimal( '70802074077032.9834292' ) for x in range( 50 ): v = Decimal( '0.' + '0' * x + '1' ) for i in range( 20 ): b = b + v c = ( 2 *R*r + ( 2 *R*r/ a )*b) / (b - ( 2 *R*r/ a )) s = ( a + b + c) / 2 heron = s * (s- a ) * (s-b) * (s-c) area1 = heron. sqrt () area2 = ( a * b * c / ( 4 * R)) area3 = ( a + b + c) * r / 2 if (area1 > area2): b -= v break print( 'a =' , a ) print( 'b =' , b) print( 'c =' , c) print( 'Area by method 1:' , area1) print( 'Area by method 2:' , area2) print( 'Area by method 3:' , area3)

After knowing the two sides (b = 70802074077032.9834292890321943043996425948069384041 , c = 95523798483318.0128249775384414796312000736043319645 ), we need to convert them back. We have to know how it processes the original value. We can take the first part of input 391bc2164f0a to figure out what it does.

First it is split into 391b and c2164f0a . The first part is added to 0x4530000000000000 , and the second part is added to 0x4330000000000000 . Now, it is 0x453000000000391b and 0x43300000c2164f0a . (They are put in a 128-bit register xmm0, as 0x453000000000391b43300000c2164f0a . To make it simpler, we talk about them as two separated parts.)

Then it takes them as 64-bit floating-points (double type in C/C++) and subtracts (double) 0x4530000000000000 from the first part, (double) 0x4330000000000000 from the second part. Now they are 0x42cc8d8000000000 and 0x41e842c9e1400000 .

The next step is simpler. It reverses the two parts and saves them into another 128-bit register xmm1. We currently have xmm0 = 0x42cc8d800000000041e842c9e1400000 and xmm1 = 0x41e842c9e140000042cc8d8000000000 .

Finally it adds xmm0 to xmm1, but takes each 64-bit value as type double. That is, The highest 64 bits 0x42cc8d8000000000 in xmm0 is added to 0x41e842c9e1400000 in xmm1, both are in type double. The result is 0x42cc8de10b278500 . The lowest 64 bits works in the same way, but with the same two values in opposite register. The result is 0x42cc8de10b278500 as well. And the lowest 64 bits 0x42cc8de10b278500 is the final result.

In short, if the original value is ghijklmnopqr , the result is the hex value of the following:

((double) 0x453000000000ghij - (double) 0x4530000000000000) + ((double) 0x43300000klmnopqr - (double) 0x4330000000000000)

Since ((double) 0x453000000000ghij - (double) 0x4530000000000000) is multiple times of ((double) 0x43300000klmnopqr - (double) 0x4330000000000000) , the first few bits of the result is decided by the former. We can mask it with 0xffffff0000000000 and add (double) 0x4530000000000000 to the result. And the last two bytes is very close to the input ghij .

For example:

a = 391bc2164f0a The result is 62791383142154.00 , 42cc8de10b278500 in hex. Mask it with 0xffffff0000000000 , we get 0x42cc8d0000000000 . Then we add (double) 0x4530000000000000 to it. We get (double) 0x453000000000391a . The lowest two bytes 391a is very close to 391b . We can try the values near it and calculate klmnopqr . Following is a C++ code for converting the result to its original value (we need to adjust the offset):

union U { double d long long int i } int main() { int offest = 0 U part1, part2, const1, const2, result const1.i = 0x4530000000000000 const2.i = 0x4330000000000000 result.i = 0x42d5b83784e05d81 part1.i = result.i & 0xffffff0000000000 part1.d += const1.d part1.i += offset printf( "%04x " , part1.i & 0xffff ) part2 = result part2.d += const2.d part2.d -= (part1.d - const1.d) printf( "%08x

" , part2.i & 0xffffffff ) part1.d -= const1.d part2.d -= const2.d part1.d += part2.d printf( "Convert back to result: %llx

" , part1.i) return 0 }

With result = 0x42d019391e61da3f (hex value of side b), offset = 0, we get 4064e4798769 . With result = 0x42d5b83784e05d81 (hex value of side c), offset = 0, we get 56e0de138176 .

The flag is flag{391bc2164f0a-4064e4798769-56e0de138176} .
