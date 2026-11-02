# Take it to the cleaners

**CTF:** RITSEC 2019  
**Category:** forensics  
**Original writeup:** <https://abraxas.io/2019/11/20/ritsec-ctf-2019/> — by **abraxas.io**. All credit goes to the original author; this is an archived copy.

---

Take it to the Cleaners!

This challenge is a nice easy introduction forensics challenge which has the above description. The file provided is the PNG below:

The two tools I usually run on forensics and steganography challenges are binwalk and exiftool. Binwalk is great for finding files hidden in files and exiftool will reveal interesting metadata in images such as the location it was taken or other useful tidbits.

First, I ran exiftool and saw the below text:

solson@Goliath:/mnt/c/Users/stewa/Downloads$ exiftool ritsec_logo2.png ExifTool Version Number : 10.80 File Name : ritsec_logo2.png Directory : . File Size : 4.3 kB File Modification Date/Time : 2019:11:15 18:13:11-08:00 File Access Date/Time : 2019:11:17 10:48:47-08:00 File Inode Change Date/Time : 2019:11:15 18:13:23-08:00 File Permissions : rwxrwxrwx File Type : PNG File Type Extension : png MIME Type : image/png Image Width : 328 Image Height : 154 Bit Depth : 8 Color Type : Palette Compression : Deflate/Inflate Filter : Adaptive Interlace : Noninterlaced Palette : (Binary data 129 bytes, use -b option to extract) Exif Byte Order : Big-endian (Motorola, MM) Image Description : Hi there! Looks like youre trying to solve the forensic_fails challenge! Good luck! Resolution Unit : inches Artist : Impos73r Y Cb Cr Positioning : Centered Copyright : RITSEC 2018 Exif Version : 0231 Components Configuration : Y, Cb, Cr, - User Comment : RVZHRlJQe1NCRVJBRlZQRl9TTlZZRl9KQkFHX1VSWUNfTEJIX1VSRVJ9 Flashpix Version : 0100 GPS Latitude Ref : North GPS Longitude Ref : West Image Size : 328x154 Megapixels : 0.051

At first I saw ‘Image Description’ as Hi there! Looks like youre trying to solve the forensic_fails challenge! Good luck! I immediately stopped reading and ran another tool or two before looking back at the exiftool output and seeing:

User Comment : RVZHRlJQe1NCRVJBRlZQRl9TTlZZRl9KQkFHX1VSWUNfTEJIX1VSRVJ9

This is a base 64 string which decodes to: EVGFRP{SBERAFVPF_SNVYF_JBAG_URYC_LBH_URER

We are told that the flag format is RITSEC{} so I took that string to dcode.fr/en and put it in a ceasar cipher solver and got: RITSEC{FORENSICS_FAILS_WONT_HELP_YOU_HERE}
