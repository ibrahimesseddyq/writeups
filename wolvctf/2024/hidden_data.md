# Hidden_Data

**CTF:** WolvCTF 2024  
**Category:** forensics  
**Original writeup:** <https://nopedawn.github.io/posts/ctfs/2024/wolv-ctf-2024/> — by **nopedawn**. All credit goes to the original author; this is an archived copy.

---

Forensics: Hidden Data#

Overview Author: dree_

Description: WOLPHV sent me this file. Not sure what to comment about it

wctf_evil.jpg

Given an image file of Wolv CTF logo,

I was assumed there’s a flag on the metadata or like LSB hidden-data, and if we take look at the metada of it, we can find the flag instantly in Comment tag

$ exiftool wctf_evil.jpg ExifTool Version Number : 12.40 File Name : wctf_evil.jpg Directory : . File Size : 11 KiB File Modification Date/Time : 2024:03:15 17:23:14+07:00 File Access Date/Time : 2024:03:18 05:44:58+07:00 File Inode Change Date/Time : 2024:03:16 11:14:42+07:00 File Permissions : -rw-r--r-- File Type : JPEG File Type Extension : jpg MIME Type : image/jpeg JFIF Version : 1.01 Resolution Unit : None X Resolution : 1 Y Resolution : 1 Comment : wctf { h1dd3n_d4t4_n0T_s0_h1dD3N } Image Width : 250 Image Height : 307 Encoding Process : Progressive DCT, Huffman coding Bits Per Sample : 8 Color Components : 3 Y Cb Cr Sub Sampling : YCbCr4:2:0 ( 2 2 ) Image Size : 250x307 Megapixels : 0.077

$ file wctf_evil.jpg wctf_evil.jpg: JPEG image data, JFIF standard 1.01, aspect ratio, density 1x1, segment length 16, comment: "wctf{h1dd3n_d4t4_n0T_s0_h1dD3N}" , progressive, precision 8, 250x307, components 3

FLAG wctf{h1dd3n_d4t4_n0T_s0_h1dD3N}
