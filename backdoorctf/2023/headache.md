# Headache

**CTF:** BackdoorCTF 2023  
**Category:** misc  
**Original writeup:** <https://seall.dev/posts/backdoorctf2023> — by **seall.dev**. All credit goes to the original author; this is an archived copy.

---

I’ve had a headache since last evening. Is there a magic spell that can cure it immediately?

At an original look at this challenge I got stumped for a while overlooking something very minor…

We start with a seemingly ‘corrupt’ file, chal.png which doesn’t open. I check EXIF first:

1 2 3 4 5 6 7 8 9 10 11 12 13 14 15 16 17 18 19 20 21 $ exiftool chal.png ExifTool Version Number : 12.70 File Name : chal.png Directory : . File Size : 90 kB File Modification Date/Time : 2023:12:18 14:43:09+11:00 File Access Date/Time : 2023:12:18 18:10:35+11:00 File Inode Change Date/Time : 2023:12:18 18:10:33+11:00 File Permissions : -rw-rw-r-- File Type : PNG File Type Extension : png MIME Type : image/png Image Width : 1265 Image Height : 687 Bit Depth : 8 Color Type : RGB with Alpha Compression : Deflate/Inflate Filter : Adaptive Interlace : Noninterlaced Image Size : 1265x687 Megapixels : 0.869

Nothing… Anything trying binwalk?

1 2 3 4 $ binwalk -e chal.png DECIMAL HEXADECIMAL DESCRIPTION -------------------------------------------------------------------------------- 41 0x29 Zlib compressed data, compressed

Nothing of interest in the data… So here I get stumped until I decide to check the bytes…

Files have ‘signatures’ (magic numbers). PNGs magic numbers are 89 50 4e 47 0d 0a 1a 0a . This does not match our files magic numbers ( ae 71 00 ff 3d 62 24 6d )

So… I change the files magic bytes to be the correct bytes, and we get the following image (that opens!).

There’s our flag! flag{sp3ll_15_89_50_4E_47}
