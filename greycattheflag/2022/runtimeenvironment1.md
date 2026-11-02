# RuntimeEnvironment1

**CTF:** GreyCatTheFlag 2022  
**Category:** rev  
**Original writeup:** <https://rainbowpigeon.me/posts/grey-cat-the-flag-qualifiers-2022/> — by **rainbowpigeon.me**. All credit goes to the original author; this is an archived copy.

---

🩸 Runtime Environment 1

GO and try to solve this basic challenge. FAQ: If you found the input leading to the challenge.txt you are on the right track MD5 (gogogo.tar.gz) = 5515f1c3eee00e4042bf7aba84bbec5c rootkid Attached: gogogo.tar.gz

1 2 3 4 a@a:/grey/gogogo$ file binary binary: ELF 64-bit LSB executable, x86-64, version 1 ( SYSV ) , statically linked, Go BuildID = OHBJFJh5S4MEkda8Q683/cMydJq6y9QbVjBCjK1KP/8R1f9ddSl9EfpM8KP2Dy/3G9-Ju3BW7WUsgoGNyvl, not stripped a@a:/grey/gogogo$ cat challenge.txt GvVf+fHWz1tlOkHXUk3kz3bqh4UcFFwgDJmUDWxdDTTGzklgIJ+fXfHUh739+BUEbrmMzGoQOyDIFIz4GvTw+j--

In the given archive there’s a 64-bit ELF Go binary and a challenge.txt containing encoded text that looks like some variant of Base64 encoding.

From blackbox analysis, the encoding scheme also seems similar to Base64 in that 3-byte input blocks ( aaa ) are independently encoded to 4-byte output blocks ( CGTx ), and padding ( - ) is used to keep the output length a multiple of 4 when the input length is not a multiple of 3:

1 2 3 4 5 6 7 8 9 10 11 12 a@a:/grey/gogogo$ echo a | ./binary CV-- a@a:/grey/gogogo$ echo aa | ./binary CGJ- a@a:/grey/gogogo$ echo aaa | ./binary CGTx a@a:/grey/gogogo$ echo aaaa | ./binary CGTxCV-- a@a:/grey/gogogo$ echo aaaaa | ./binary CGTxCGJ- a@a:/grey/gogogo$ echo aaaaaa | ./binary CGTxCGTx

Let’s check out the pseudocode of the binary in IDA Pro for further analysis.

1 2 3 4 5 6 7 8 9 10 11 12 13 14 15 16 17 // main.main void __cdecl main_main () { v10 = runtime_newobject ( & RTYPE_string ); v12 [ 0 ] = & RTYPE__ptr_string ; v12 [ 1 ] = v10 ; fmt_Fscanln ( & go_itab__ptr_os_File_comma_io_Reader , os_Stdin , v12 , 1LL , 1LL ); v7 = 4 * (((((( v10 [ 1 ] + 2 ) * 0xAAAAAAAAAAAAAAABLL ) >> 64 ) + v10 [ 1 ] + 2 ) >> 1 ) - (( v10 [ 1 ] + 2 ) >> 63 )); v9 = runtime_makeslice ( & RTYPE_uint8 , v7 , v7 ); v1 = runtime_stringtoslicebyte ( v8 , * v10 , v10 [ 1 ]); main_Encode ( v9 , v7 , v7 , v1 , v2 ); v3 = runtime_slicebytetostring ( 0LL , v4 , v5 , v6 ); v0 = runtime_convTstring ( v3 , * ( & v3 + 1 )); v11 [ 0 ] = & RTYPE_string ; v11 [ 1 ] = v0 ; fmt_Fprintln ( & go_itab__ptr_os_File_comma_io_Writer , os_Stdout , v11 , 1LL , 1LL ); }

A string is read from stdin , converted into a Go slice of bytes, and then passed into main_Encode . The encoded slice of bytes is then converted back into a string and printed.

What does main_Encode do? At the top, we see a string b64str containing 64 unique characters. This is used to form the majority of output in the first while loop where input_ptr is iterated over in chunks of 3 while output is iterated over in chunks of 4.

1 2 3 4 5 6 7 8 9 10 11 12 13 14 15 16 17 18 19 20 21 22 23 24 25 26 27 28 29 30 31 32 33 34 35 36 37 38 39 40 41 42 43 44 45 46 47 48 49 50 51 52 53 54 55 56 57 58 59 60 61 62 63 64 65 66 67 68 69 70 71 72 73 74 75 76 77 78 79 80 81 82 83 84 85 86 // main.Encode __int64 __usercall main_Encode @ < rax > ( __int64 output , unsigned __int64 output_length , __int64 a3 , __int64 input_ptr , unsigned __int64 input_length ) { qmemcpy ( b64str , "NaRvJT1B/m6AOXL9VDFIbUGkC+sSnzh5jxQ273d4lHPg0wcEpYqruWyfZoM8itKe" , sizeof ( b64str )); i_by_3 = 0LL ; j_enc_by_4 = 0LL ; while ( i_by_3 < 3 * ( input_length / 3 ) ) { if ( i_by_3 >= input_length ) runtime_panicIndex (); if ( i_by_3 + 1 >= input_length ) runtime_panicIndex (); if ( i_by_3 + 2 >= input_length ) runtime_panicIndex (); v7 = ( * ( input_ptr + i_by_3 ) << 16 ) | ( * ( input_ptr + i_by_3 + 1 ) << 8 ) | * ( i_by_3 + input_ptr + 2 ); if ( j_enc_by_4 >= output_length ) runtime_panicIndex (); * ( output + j_enc_by_4 ) = * ( b64str + (( v7 >> 18 ) & 0x3F )); v8 = v7 ; v9 = * ( b64str + (( v7 >> 12 ) & 0x3F )); if ( j_enc_by_4 + 1 >= output_length ) runtime_panicIndex (); * ( j_enc_by_4 + output + 1 ) = v9 ; v10 = v8 ; v11 = * ( b64str + (( v8 >> 6 ) & 0x3F )); if ( j_enc_by_4 + 2 >= output_length ) runtime_panicIndex (); * ( j_enc_by_4 + output + 2 ) = v11 ; v12 = * ( b64str + ( v10 & 0x3F )); if ( j_enc_by_4 + 3 >= output_length ) runtime_panicIndex (); * ( output + j_enc_by_4 + 3 ) = v12 ; i_by_3 += 3LL ; j_enc_by_4 += 4LL ; } remainder = input_length - i_by_3 ; if ( input_length == i_by_3 ) return a3 ; if ( i_by_3 >= input_length ) runtime_panicIndex (); if ( remainder == 2 ) { if ( i_by_3 + 1 >= input_length ) runtime_panicIndex (); v14 = ( * ( input_ptr + i_by_3 ) << 16 ) | ( * ( input_ptr + i_by_3 + 1 ) << 8 ); } else { v14 = * ( input_ptr + i_by_3 ) << 16 ; } v15 = v14 ; v16 = * ( b64str + (( v14 >> 18 ) & 0x3F )); if ( j_enc_by_4 >= output_length ) runtime_panicIndex (); * ( output + j_enc_by_4 ) = v16 ; v17 = v15 ; v18 = * ( b64str + (( v15 >> 12 ) & 0x3F )); if ( j_enc_by_4 + 1 >= output_length ) runtime_panicIndex (); * ( j_enc_by_4 + output + 1 ) = v18 ; if ( remainder == 1 ) // add padding { if ( j_enc_by_4 + 2 >= output_length ) runtime_panicIndex (); * ( j_enc_by_4 + output + 2 ) = '-' ; if ( j_enc_by_4 + 3 >= output_length ) runtime_panicIndex (); * ( j_enc_by_4 + output + 3 ) = '-' ; } else if ( remainder == 2 ) { v20 = * ( b64str + (( v17 >> 6 ) & 0x3F )); if ( j_enc_by_4 + 2 >= output_length ) runtime_panicIndex (); * ( j_enc_by_4 + output + 2 ) = v20 ; if ( j_enc_by_4 + 3 >= output_length ) runtime_panicIndex (); * ( j_enc_by_4 + output + 3 ) = '-' ; // add padding } return a3 ; }

At the bottom, we see that for remaining input bytes not covered by the chunks of 3, the character - is padded to output . All these observations match what we found from our blackbox analysis and it really does seem like Base64 encoding. For additional confirmation, I also Googled the >> 18 & 0x3f seen in the code and the results all show code for Base64 encoding.

The difference here is that NaRvJT1B/m6AOXL9VDFIbUGkC+sSnzh5jxQ273d4lHPg0wcEpYqruWyfZoM8itKe is used as a custom character set with - as the padding instead of the standard ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789+/ with = as the padding.

At this point I just tried decoding the contents of challenge.txt in CyberChef with the custom Base64 character set and it turns out 4 rounds of decoding are needed.

Flag: grey{B4s3d_G0Ph3r_r333333}
