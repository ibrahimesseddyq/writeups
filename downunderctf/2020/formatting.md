# Formatting

**CTF:** DownUnderCTF 2020  
**Category:** rev  
**Original writeup:** <https://github.com/joaofcmb/DownUnderCTF-writeups/tree/master/reversing/formatting> — by **joaofcmb**. All credit goes to the original author; this is an archived copy.

---

- **Category:** Reversing

- **Difficulty:** Beginner

- **Author:** h4sh5

Its really easy, I promise

**Attached file:** [formatting](./formatting)

## Writeup

The description doesn't really lead us anywhere. Let's download the attached file and see what kind it is.

```bash

file formatting

```

```

formatting: ELF 64-bit LSB shared object, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, BuildID[sha1]=bd9f51c1f1535b269a0707054009063f984f6738, for GNU/Linux 3.2.0, not stripped

```

It appears to be an executable file in **ELF**. Let's try to run it.

```bash

./formatting

```

```

haha its not that easy}

```

Clearly that didn't work. Looks like we have to dig up the file a little. Since it's an **ELF** file, let's use the `readelf` command to dump the `.rodata` section.

```bash

readelf -x .rodata formatting

```

```

0x00002000 01000200 00000000 44554354 467b6861 ........DUCTF{ha

0x00002010 68612069 7473206e 6f742074 68617420 ha its not that

0x00002020 65617379 7d000000 25732530 32782530 easy}...%s%02x%0

0x00002030 32782530 32782530 32782530 32782530 2x%02x%02x%02x%0

0x00002040 32782530 32782530 32787d00 6431645f 2x%02x%02x}.d1d_

0x00002050 596f755f 4a757374 5f6c7472 6163655f You_Just_ltrace_

0x00002060 00

```

It appears to be hinting that `DUCTF{ha ha its not that easy}` is the flag, but it doesn't really fit the typical flag format, and if we try it, we'll get a failed submission. This is not a deadend though, because there are two other strings; `%s%02x%02x%02x%02x%02x%02x%02x%02x}`, which looks like a **format string**, and `d1d_You_Just_ltrace_`, which appears to be hinting at the use of the `ltrace` command. Let's give it a shot.

```bash

ltrace formatting

```

```

sprintf("d1d_You_Just_ltrace_296faa2990ac"..., "%s%02x%02x%02x%02x%02x%02x%02x%0"..., "d1d_You_Just_ltrace_", 0x29, 0x6f, 0xaa, 0x29, 0x90, 0xac, 0xbc, 0x36) = 37

puts("haha its not that easy}"haha its not that easy}

) = 24

```

We can see that `sprintf` is being used to generate a string with `d1d_You_Just_ltrace_` and a sequence of eight 2-character sequences. If we manually format the end result, we obtain the flag `DUCTF{d1d_You_Just_ltrace_296faa2990acbc36}`.
