# warmup

**CTF:** SHELL 2022  
**Category:** rev  
**Original writeup:** <https://blog.hamayanhamayan.com/entry/2022/08/14/222128> — by **blog.hamayanhamayan.com**. All credit goes to the original author; this is an archived copy.

---

[rev] warmup

ghidraに食わせてmain関数を見る。

用意された変数に対して右に2つビットシフトして比較をしている。

コードを複製して、右に2つビットシフトしたものを文字列として出力してみよう。

#include <stdio.h> int main ( void ) { int local_a8 [ 28 ]; local_a8[ 0 ] = 0x1cc ; local_a8[ 1 ] = 0x1a0 ; local_a8[ 2 ] = 0x194 ; local_a8[ 3 ] = 0x1b0 ; local_a8[ 4 ] = 0x1b0 ; local_a8[ 5 ] = 0x18c ; local_a8[ 6 ] = 0x1d0 ; local_a8[ 7 ] = 0x198 ; local_a8[ 8 ] = 0x1ec ; local_a8[ 9 ] = 0x188 ; local_a8[ 10 ] = 0xc4 ; local_a8[ 11 ] = 0x1d0 ; local_a8[ 12 ] = 0x15c ; local_a8[ 13 ] = 0x1a4 ; local_a8[ 14 ] = 0xd4 ; local_a8[ 15 ] = 0x194 ; local_a8[ 16 ] = 0x17c ; local_a8[ 17 ] = 0xc0 ; local_a8[ 18 ] = 0x1c0 ; local_a8[ 19 ] = 0xcc ; local_a8[ 20 ] = 0x1c8 ; local_a8[ 21 ] = 0x104 ; local_a8[ 22 ] = 0x1d0 ; local_a8[ 23 ] = 0xc0 ; local_a8[ 24 ] = 0x1c8 ; local_a8[ 25 ] = 0x14c ; local_a8[ 26 ] = 50 ; for ( int local_ac = 0 ; local_ac < 0x1b ; local_ac = local_ac + 1 ) { printf ( " %c " , local_a8[local_ac] >> 2 ); } return 0 ; }

これを実行すると shellctf{b1tWi5e_0p3rAt0rS と出力される。最後にカッコを補ったらフラグになる。

shellctf{b1tWi5e_0p3rAt0rS}
