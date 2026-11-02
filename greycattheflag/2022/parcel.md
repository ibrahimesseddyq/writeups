# Parcel

**CTF:** GreyCatTheFlag 2022  
**Category:** rev  
**Original writeup:** <https://rainbowpigeon.me/posts/grey-cat-the-flag-qualifiers-2022/> — by **rainbowpigeon.me**. All credit goes to the original author; this is an archived copy.

---

🍭 Parcel

The binary is packed. Try to decompile it and you will find unreadable code. Find out how it is packed and unpack it to recover the original binary. MD5 (parcel) = 903adf25f54687ef3d875b51442e20df daniellimws Hint: Run strings on the binary. There are some interesting strings in there. Attached: parcel

1 parcel: ELF 64-bit LSB executable, x86-64, version 1 ( SYSV ) , statically linked, no section header

If we run strings on the provided ELF binary as hinted, we’ll see the strings “UPX!”.

1 2 3 4 5 6 7 8 9 10 11 a@a:/grey$ strings parcel | tail /@1dC ?88l XX$/ ` ?2 7dCI 6 { $O .? ; /QqQ UPX! UPX!

This means that the ELF is likely packed with UPX. If vanilla UPX is used, such binaries can simply be unpacked with the official upx binary itself by adding the -d flag:

1 2 3 4 5 6 7 8 9 10 C : \grey >upx -d parcel -o unpackedparcel Ultimate Packer for eXecutables Copyright (C) 1996 - 2020 UPX 3.96w Markus Oberhumer, Laszlo Molnar & John Reiser Jan 23rd 2020 File size Ratio Format Name -------------------- ------ ----------- ----------- 428392 < - 105224 24.56% linux/amd64 unpackedparcel Unpacked 1 file.

1 2 a@a:/grey$ file unpackedparcel unpackedparcel: ELF 64-bit LSB executable, x86-64, version 1 ( SYSV ) , dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, BuildID [ sha1 ]= f616d88f62c910d6dddddd732eb503f9c61fa88a, for GNU/Linux 3.2.0, not stripped

Ok now where’s the flag? If we run parcel or unpackedparcel , it’ll ask us for the address of certain functions like h12 . Now that we have the unpacked unstripped binary unpackedparcel , we can get those addresses easily using nm -t d unpackedparcel | grep <function_name> .

1 2 3 4 5 6 7 8 9 10 11 a@a:/grey$ nm -t d unpackedparcel | grep h12 0000000004207125 T h12 a@a:/grey$ nm -t d unpackedparcel | grep t80 0000000004221073 T t80 a@a:/grey$ nm -t d unpackedparcel | grep g20 0000000004206113 T g20 a@a:/grey$ ./parcel Tell me the address of the function h12 ( in decimal ) : 4207125 Tell me the address of the function t80 ( in decimal ) : 4221073 Tell me the address of the function g20 ( in decimal ) : 4206113 Congrats! grey { d1d_y0u_us3_nm_0r_objdump_0r_gdb_0r_ghidra_0r_rizin_0r_ida_0r_binja? }

Flag: grey{d1d_y0u_us3_nm_0r_objdump_0r_gdb_0r_ghidra_0r_rizin_0r_ida_0r_binja?}
