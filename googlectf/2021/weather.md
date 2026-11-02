# Weather

**CTF:** GoogleCTF 2021  
**Category:** rev  
**Original writeup:** <https://ctftime.org/writeup/34545> — by **ctftime.org**. All credit goes to the original author; this is an archived copy.

---

# Weather

## Analysis

Decompiled C code:

```c

__int64 __fastcall main(int a1, char **a2, char **a3, double a4)

{

int v5; // [rsp+0h] [rbp-40h] BYREF

const char *v6; // [rsp+8h] [rbp-38h]

const char *v7; // [rsp+10h] [rbp-30h] BYREF

int v8; // [rsp+18h] [rbp-28h]

const char *v9; // [rsp+20h] [rbp-20h]

int v10; // [rsp+30h] [rbp-10h] BYREF

const char *v11; // [rsp+38h] [rbp-8h]

puts("Welcome to our global weather database!");

puts("What city are you interested in?");

__isoc99_scanf("%100s", s2);

if ( !strcmp("London", s2) )

{

v10 = 5;

v11 = "W";

v7 = "rain";

v8 = 1337;

v9 = "mm";

v5 = 10;

v6 = "°C";

}

else if ( !strcmp("Moscow", s2) )

{

v10 = 7;

v11 = "N";

v7 = "snow";

v8 = 250;

v9 = "cm";

v5 = -30;

v6 = "°C";

}

else if ( !strcmp("Miami", s2) )

{

v10 = 1;

v11 = "NE";

v7 = "sweat";

v8 = 100;

v9 = "ml";

v5 = 31337;

v6 = "°F";

}

else

{

v10 = 10;

v11 = "SW";

v7 = "nothing";

v8 = 0;

v9 = (const char *)&unk_30BE;

v5 = 15;

v6 = "°C";

}

puts("Weather for today:");

printf("Precipitation: %P

", &v7, a4);

printf("Wind: %W

", &v10);

printf("Temperature: %T

", &v5;;

printf("Flag: %F

", a4);

return 0LL;

}

```

Well, the main function seems to have nothing to do with the flag...

Nevertheless, several special specifiers, such as `%P`, `%W`, `%T`, and `%F`, are presented in the format string. This implies that the generation of the flag may have something to do with customized specifiers.

After some investigation, I end up in a function located at `0x2328`, which contains several lines of code regarding `register_printf_function`.

```c

int register()

{

register_printf_function('W', (printf_function *)sub_2190, (printf_arginfo_function *)arginfo);

register_printf_function('P', (printf_function *)sub_21E0, (printf_arginfo_function *)arginfo);

register_printf_function('T', (printf_function *)sub_225A, (printf_arginfo_function *)arginfo);

register_printf_function('F', (printf_function *)sub_22AA, (printf_arginfo_function *)arginfo);

register_printf_function('C', (printf_function *)sub_2097, (printf_arginfo_function *)arginfo2);

register_printf_function('M', (printf_function *)sub_1185, (printf_arginfo_function *)arginfo2);

register_printf_function('S', (printf_function *)sub_12DC, (printf_arginfo_function *)arginfo2);

register_printf_function('O', (printf_function *)sub_143B, (printf_arginfo_function *)arginfo2);

register_printf_function('X', (printf_function *)sub_159A, (printf_arginfo_function *)arginfo2);

register_printf_function('V', (printf_function *)sub_16FA, (printf_arginfo_function *)arginfo2);

register_printf_function('N', (printf_function *)sub_185A, (printf_arginfo_function *)arginfo2);

register_printf_function('L', (printf_function *)sub_19B8, (printf_arginfo_function *)arginfo2);

register_printf_function('R', (printf_function *)sub_1B19, (printf_arginfo_function *)arginfo2);

register_printf_function('E', (printf_function *)sub_1C7A, (printf_arginfo_function *)arginfo2);

register_printf_function('I', (printf_function *)sub_1DD9, (printf_arginfo_function *)arginfo2);

return register_printf_function('U', (printf_function *)sub_1F38, (printf_arginfo_function *)arginfo2);

}

```

Well, we have already seen `%P`, `%W`, `%T`, and `%F` specifiers. After some brief reconnaissance, the remaining specifiers as well as their corresponding `print_function`s seem to insinuate a *virtual machine* written in format string.

Now, let's analyze the specifiers. We can inspect each `arginfo` structure with gdb. Take `%C` as an example:

```

pwndbg> p *(printf_arginfo_function *)0x5555555562f0

$1 = {int (const struct printf_info *, size_t, int *)} 0x5555555562f0

pwndbg> p *(struct printf_info *)0x5555555562f0

$2 = {

prec = -443987883,

width = -125990584,

spec = -260732600 L'\xf0758948',

is_long_double = 0,

is_short = 0,

is_long = 0,

alt = 1,

space = 0,

left = 0,

showsign = 1,

group = 0,

extra = 1,

is_char = 0,

wide = 0,

i18n = 1,

is_binary128 = 0,

__pad = 4,

user = 59477,

pad = 184 L'¸'

}

```

```c

__int64 __fastcall sub_2097(FILE *stream, struct printf_info *info, const void *const *args)

{

int prec; // [rsp+24h] [rbp-Ch]

_BOOL4 v5; // [rsp+2Ch] [rbp-4h]

prec = info->prec;

if ( (*((_BYTE *)info + 12) & 0x20) != 0 ) // left space

{

v5 = dword_70A0[prec] < 0;

}

else if ( (*((_BYTE *)info + 12) & 0x40) != 0 ) // show sign

{

v5 = dword_70A0[prec] > 0;

}

else

{

v5 = info->pad != '0' || dword_70A0[prec] == 0; // pad field

}

if ( v5 )

fprintf(stream, &a52cS[info->width]);

return 0LL;

}

```

So basically the handling function uses `left space`, `show sign`, and `pad field` to determine the value of `v5`. If `v5` is true, then it will call `fprintf` (Which means that the virtual machine will jump to `info->width` address).

Take the handling function of `%S` as another example:

```c

__int64 __fastcall sub_12DC(FILE *stream, const struct printf_info *info, const void *const *args)

{

int prec; // [rsp+24h] [rbp-14h]

int width; // [rsp+28h] [rbp-10h]

int v6; // [rsp+2Ch] [rbp-Ch]

char *v7; // [rsp+30h] [rbp-8h]

width = info->width;

prec = info->prec;

if ( (*((_BYTE *)info + 12) & 0x20) != 0 ) // left (%-)

{

v7 = &a52cS[width];

}

else if ( (*((_BYTE *)info + 12) & 0x40) != 0 )// show_sign (%+)

{

v7 = &a52cS[dword_70A0[width]];

}

else

{

v7 = (char *)&dword_70A0[width];

}

v6 = 0;

if ( (*((_BYTE *)info + 13) & 2) != 0 ) // char (%hh)

{

v6 = *(_DWORD *)&a52cS[prec];

}

else if ( (*((_BYTE *)info + 12) & 2) != 0 ) // short (%h)

{

v6 = *(_DWORD *)&a52cS[dword_70A0[prec]];

}

else if ( (*((_BYTE *)info + 12) & 1) != 0 ) // long double (%ll)

{

v6 = info->prec;

}

else if ( (*((_BYTE *)info + 12) & 4) != 0 ) // long (%l and %11, but %11 enters previous else if)

{

v6 = dword_70A0[prec];

}

*(_DWORD *)v7 += v6;

return 0LL;

}

```

The function uses `left` and `showsign` to calculate an address `v7`. Similarly, it uses either `char` or `short` or `long double` or `long` to calculate the value `v6`. Finally, it adds `v6` to `[v7]`. All of the other specifiers perform similar calculation. Below is a breif summary of the operations done by each handling function:

* `%C`: conditional branching

* `%M`: =

* `%S`: +

* `%O`: -

* `%X`: *

* `%V`: /

* `%N`: %

* `%L`: <<

* `%R`: >>

* `%E`: ^

* `%I`: &

* `%U`: |

## `printf` assembly code

We can observer several suspicious format strings via `strings` command.

```bash

...

%52C%s

%3.1hM%3.0lE%+1.3lM%1.4llS%3.1lM%3.2lO%-7.3C

%0.4096hhM%0.255llI%1.0lM%1.8llL%0.1lU%1.0lM%1.16llL%0.1lU%1.200llM%2.1788llM%7C%-6144.1701736302llM%0.200hhM%0.255llI%0.37llO%0200.0C

...

```

Hmmm. `%52C%s` looks familiar. If we examine the handling function of specifier `%F`:

```c

__int64 __fastcall sub_22AA(FILE *stream, const struct printf_info *info, const void *const *args, const char *a4)

{

return (unsigned int)fprintf(stream, "%52C%s", **(_QWORD **)args, a4);

}

```

We can clearly see that the string `%52C%s` is provided as the second parameter of `fprintf`. Correspondingly, it is to appropriate to postulate that `%52C%s` is the *entry point* of the printf virtual machine.

## Disassembler

Hence, it is clear that our first goal is to come up with a printf virtual machine disassembler.

```python

import re

import sys

regex = re.compile('([+-0])?([0-9]*)\.?([0-9]*)?([a-z]*)([A-Za-z])')

counter = 0

with open('assembly', 'r') as f:

fmt_str = f.read()

fmt_strs = fmt_str.split('

')

for fmt_str in fmt_strs:

fmt_specs = fmt_str.split('%')

for fmt in fmt_specs:

if fmt == '':

continue

print('{:08x} '.format(counter), end = '')

counter += len(fmt) + 1

g = regex.match(fmt).groups(0)

if g[4] == 'C':

if g[0] == 0:

print('branch {:#x}'.format(int(g[1])))

elif g[0] == '0':

print('if (reg{} == 0) branch {:#x}'.format(g[2], int(g[1])))

elif g[0] == '+':

print('if (reg{} > 0) branch {:#x}'.format(g[2], int(g[1])))

elif g[0] == '-':

print('if (reg{} < 0) branch {:#x}'.format(g[2], int(g[1])))

# Probably used to print the flag

# It's ok to temporarily ignore it

elif g[4] == 's':

print('nop')

continue

else:

if g[0] == '0' and g[1] == '':

g = (0, '0', g[2], g[3], g[4])

if g[0] == 0:

print('reg{}'.format(g[1]), end = '')

elif g[0] == '+':

print('[reg{}]'.format(g[1]), end = '')

elif g[0] == '-':

print('[{:#x}]'.format(int(g[1])), end = '')

if g[4] == 'M':

print(' = ', end = '')

elif g[4] == 'S':

print(' += ', end = '')

elif g[4] == 'O':

print(' -= ', end = '')

elif g[4] == 'X':

print(' *= ', end = '')

elif g[4] == 'V':

print(' /= ', end = '')

elif g[4] == 'N':

print(' %= ', end = '')

elif g[4] == 'L':

print(' <<= ', end = '')

elif g[4] == 'R':

print(' >>= ', end = '')

elif g[4] == 'E':

print(' ^= ', end = '')

elif g[4] == 'I':

print(' &= ', end = '')

elif g[4] == 'U':

print(' |= ', end = '')

if g[3] == 'hh':

print('[{:#x}]'.format(int(g[2])))

elif g[3] == 'h':

print('[reg{}]'.format(g[2]))

elif g[3] == 'll':

print('{:#x}'.format(int(g[2])))

elif g[3] == 'l':

print('reg{}'.format(g[2]))

print('{:08x} '.format(counter), end = '')

counter += 1

print('return

')

```

## Analyze disassembly

### Part 1

Disassembling the two suspicious format strings yields the following pseudo assembly code:

```

00000000 branch 0x34

00000004 nop

00000006 return

00000007 reg3 = [reg1]

0000000d reg3 ^= reg0

00000013 [reg1] = reg3

0000001a reg1 += 0x4

00000021 reg3 = reg1

00000027 reg3 -= reg2

0000002d if (reg3 < 0) branch 0x7

00000033 return

00000034 reg0 = [0x1000]

0000003e reg0 &= 0xff

00000047 reg1 = reg0

0000004d reg1 <<= 0x8

00000054 reg0 |= reg1

0000005a reg1 = reg0

00000060 reg1 <<= 0x10

00000068 reg0 |= reg1

0000006e reg1 = 0xc8

00000077 reg2 = 0x6fc

00000081 branch 0x7

00000084 [0x1800] = 0x656e6f6e

00000098 reg0 = [0xc8]

000000a1 reg0 &= 0xff

000000aa reg0 -= 0x25

000000b2 if (reg0 == 0) branch 0xc8

000000ba return

```

According to the assembly, the purpose of function `0x7` is to xor the byte from address `reg1` to address `reg2` with character `reg0`. Function `0x34` jumps to function `0x7` with argument (0x25 ('%'), 0xc8, 0x6fc) and check whether the character at 0xc8 is 0x25. If the condition is true, the main function will jumps to 0xc8.

This part of the assembly code has nothing to do with the generation of the flag. Hence, let's grab the bytes between 0xc8 & 0x6fc and xor these bytes with '%'.

```python3

from pwn import ELF

elf = ELF('./weather')

fmt_xor = elf.read(0x5148, 0x634)

xor_char = fmt_xor[0] ^ ord('%')

fmt_str = bytes(x ^ xor_char for x in fmt_xor)

fmt_str = fmt_str.replace(b'\0', b'

')

fmt_str = fmt_str.decode()

print(fmt_str)

```

Result:

```

%4.5000llM%0.13200llM%337C%0.0llM%500C%1262C%0653.0C

%1.0llM

%3.0lM%3.2lN%0253.3C%2.1llS%3.2lM%3.3lX%3.0lO%3.1llO%-261.3C

%+4.0lM%4.2llS

%1.1llM%2.2llM%261C%+322.1C%0.1llS%1.13600llM%1.0lO%+337.1C

%0.0llM

%0.2llV

%0.3llX%0.1llS

%1.0lM%1.2llN%0405.1C%+413.1C%470C%0.1llS

%1.0lM%1.1llO%0397.1C%+428.1C

%2.0lM%2.4096llS%4.2hM%4.255llI%+540.4C

%2.0lM%2.2llX%2.5000llS%2.2hM%2.255llI%4.2lE%0.1llS%2.0lM%470C%4.0lS%4.255llI%0.2lM%2.1llO%2.4500llS%+2.4lM%500C

%0.123456789llM%1.0llM%1.4096llS%1.1hM%0.1lE%2.0llM%2.846786818llS%2.0lE%1.0llM%1.6144llS%+1.2lM%1.4llM%1.4096llS%1.1hM%0.1lE%2.0llM%2.1443538759llS%2.0lE%1.4llM%1.6144llS%+1.2lM%1.8llM%1.4096llS%1.1hM%0.1lE%2.0llM%2.1047515510llS%2.0lE%1.8llM%1.6144llS%+1.2lM%1.12llM%1.4096llS%1.1hM%0.1lE%2.0llM%2.359499514llS%2.1724461856llS%2.0lE%1.12llM%1.6144llS%+1.2lM%1.16llM%1.4096llS%1.1hM%0.1lE%2.0llM%2.241024035llS%2.0lE%1.16llM%1.6144llS%+1.2lM%1.20llM%1.4096llS%1.1hM%0.1lE%2.0llM%2.222267724llS%2.0lE%1.20llM%1.6144llS%+1.2lM%1.24llM%1.4096llS%1.1hM%0.1lE%2.0llM%2.844096018llS%2.0lE%1.24llM%1.6144llS%+1.2lM

%0.0llM%1.0llM%1.4500llS%1.1hM%2.0llM%2.1374542625llS%2.1686915720llS%2.1129686860llS%1.2lE%0.1lU%1.4llM%1.4500llS%1.1hM%2.0llM%2.842217029llS%2.1483902564llS%1.2lE%0.1lU%1.8llM%1.4500llS%1.1hM%2.0llM%2.1868013731llS%1.2lE%0.1lU%1.12llM%1.4500llS%1.1hM%2.0llM%2.584694732llS%2.1453312700llS%1.2lE%0.1lU%1.16llM%1.4500llS%1.1hM%2.0llM%2.223548744llS%1.2lE%0.1lU%1.20llM%1.4500llS%1.1hM%2.0llM%2.1958883726llS%2.1916008099llS%1.2lE%0.1lU%1.24llM%1.4500llS%1.1hM%2.0llM%2.1829937605llS%2.1815356086llS%2.253836698llS%1.2lE%0.1lU

```

### Part 2

Now, let's disassemble each format string and analyze it one by one.

First, let's focus on the function at 0xc8:

```

000000c8 reg4 = 0x1388

000000d2 reg0 = 0x3390

000000dd branch 0x151

000000e2 reg0 = 0x0

000000e9 branch 0x1f4

000000ee branch 0x4ee

000000f4 if (reg0 == 0) branch 0x28d

000000fc return

```

It looks like this function set reg4 to 0x1338 and reg0 to 0x3390, tries to call several functions related to flag generation, checks if the result is 0, and jumps to 0x28d if the condition is met. Since these interal functions are essential to our flag generation, it is natural for us to investigate them in details.

### Part 3: function 0x151

Now, let's examine function 0x151 and its accompanied sub functions:

```

000000fd reg1 = 0x0

00000104 return

00000105 reg3 = reg0

0000010b reg3 %= reg2

00000111 if (reg3 == 0) branch 0xfd

00000119 reg2 += 0x1

00000120 reg3 = reg2

00000126 reg3 *= reg3

0000012c reg3 -= reg0

00000132 reg3 -= 0x1

00000139 if (reg3 < 0) branch 0x105

00000141 return

00000142 [reg4] = reg0

00000149 reg4 += 0x2

00000150 return

00000151 reg1 = 0x1

00000158 reg2 = 0x2

0000015f branch 0x105

00000164 if (reg1 > 0) branch 0x142

0000016c reg0 += 0x1

00000173 reg1 = 0x3520

0000017e reg1 -= reg0

00000184 if (reg1 > 0) branch 0x151

0000018c return

```

So basically the function first set reg1 to 1 and reg2 to 2 and jumps to 0x105 first. Then, it gradually perform `reg3 % reg2` (a.k.a reg0 % reg2), starting from 2 and stopping when `reg3 * reg3 >= reg0`. from 0x1111 we can see that if `reg3 % reg2 == 0` (which means that reg2 is a factor of reg3), then the function will call 0xfd and set reg1 to 0.

Sounds familiar, right? It appears that function 0x105 is a prime checker!

Now, let's go back to function 0x151. after calling 0x105, if reg1 is greater that 0, which means that reg0 is a prime number, the function will jump to 0x142, which stores reg0 into reg4. After that, reg4 will increment itself by 2, and the loop continues until 0x3520.

In conclusion, what this function tries to do is to calculate all primes within 0x3390 and 0x3520, then store all of these primes in 0x1388 as an int16_t array.

### Part 4: function 0x1f4

```

0000018d reg0 = 0x0

00000194 return

00000195 reg0 /= 0x2

0000019c return

0000019d reg0 *= 0x3

000001a4 reg0 += 0x1

000001ab return

000001ac reg1 = reg0

000001b2 reg1 %= 0x2

000001b9 if (reg1 == 0) branch 0x195

000001c1 if (reg1 > 0) branch 0x19d

000001c9 branch 0x1d6

000001ce reg0 += 0x1

000001d5 return

000001d6 reg1 = reg0

000001dc reg1 -= 0x1

000001e3 if (reg1 == 0) branch 0x18d

000001eb if (reg1 > 0) branch 0x1ac

000001f3 return

000001f4 reg2 = reg0

000001fa reg2 += 0x1000

00000204 reg4 = [reg2]

0000020a reg4 &= 0xff

00000213 if (reg4 > 0) branch 0x21c

0000021b return

0000021c reg2 = reg0

00000222 reg2 *= 0x2

00000229 reg2 += 0x1388

00000233 reg2 = [reg2]

00000239 reg2 &= 0xff

00000242 reg4 ^= reg2

00000248 reg0 += 0x1

0000024f reg2 = reg0

00000255 branch 0x1d6

0000025a reg4 += reg0

00000260 reg4 &= 0xff

00000269 reg0 = reg2

0000026f reg2 -= 0x1

00000276 reg2 += 0x1194

00000280 [reg2] = reg4

00000287 branch 0x1f4

0000028c return

```

Well, after spotting function 0x195 and function 0x19d, I immediately thought of the Collatz conjecture (3n+1 problem)!

I won't go into details here since the assembly are quite lengthy.

In conclusion, what the function is trying to do is calculate `((primes[i] & 0xff)) + Collatz(i + 1)) & 0xff` and store these values in 0x1194 as a byte array.

### Part 5: function 0x4ee and function 0x28d

```

000004ee reg0 = 0x0

000004f5 reg1 = 0x0

000004fc reg1 += 0x1194

00000506 reg1 = [reg1]

0000050c reg2 = 0x0

00000513 reg2 += 0x51eddb21

00000523 reg2 += 0x648c4a88

00000533 reg2 += 0x4355a74c

00000543 reg1 ^= reg2

00000549 reg0 |= reg1

0000054f reg1 = 0x4

00000556 reg1 += 0x1194

00000560 reg1 = [reg1]

00000566 reg2 = 0x0

0000056d reg2 += 0x32333645

0000057c reg2 += 0x58728e64

0000058c reg1 ^= reg2

00000592 reg0 |= reg1

00000598 reg1 = 0x8

0000059f reg1 += 0x1194

000005a9 reg1 = [reg1]

000005af reg2 = 0x0

000005b6 reg2 += 0x6f57a0a3

000005c6 reg1 ^= reg2

000005cc reg0 |= reg1

000005d2 reg1 = 0xc

000005da reg1 += 0x1194

000005e4 reg1 = [reg1]

000005ea reg2 = 0x0

000005f1 reg2 += 0x22d9bbcc

00000600 reg2 += 0x569fcabc

00000610 reg1 ^= reg2

00000616 reg0 |= reg1

0000061c reg1 = 0x10

00000624 reg1 += 0x1194

0000062e reg1 = [reg1]

00000634 reg2 = 0x0

0000063b reg2 += 0xd531548

0000064a reg1 ^= reg2

00000650 reg0 |= reg1

00000656 reg1 = 0x14

0000065e reg1 += 0x1194

00000668 reg1 = [reg1]

0000066e reg2 = 0x0

00000675 reg2 += 0x74c2318e

00000685 reg2 += 0x7233f6a3

00000695 reg1 ^= reg2

0000069b reg0 |= reg1

000006a1 reg1 = 0x18

000006a9 reg1 += 0x1194

000006b3 reg1 = [reg1]

000006b9 reg2 = 0x0

000006c0 reg2 += 0x6d12a1c5

000006d0 reg2 += 0x6c3422b6

000006e0 reg2 += 0xf213d9a

000006ef reg1 ^= reg2

000006f5 reg0 |= reg1

000006fb return

```

```

0000028d reg0 = 0x75bcd15

0000029c reg1 = 0x0

000002a3 reg1 += 0x1000

000002ad reg1 = [reg1]

000002b3 reg0 ^= reg1

000002b9 reg2 = 0x0

000002c0 reg2 += 0x3278f102

000002cf reg2 ^= reg0

000002d5 reg1 = 0x0

000002dc reg1 += 0x1800

000002e6 [reg1] = reg2

000002ed reg1 = 0x4

000002f4 reg1 += 0x1000

000002fe reg1 = [reg1]

00000304 reg0 ^= reg1

0000030a reg2 = 0x0

00000311 reg2 += 0x560aa747

00000321 reg2 ^= reg0

00000327 reg1 = 0x4

0000032e reg1 += 0x1800

00000338 [reg1] = reg2

0000033f reg1 = 0x8

00000346 reg1 += 0x1000

00000350 reg1 = [reg1]

00000356 reg0 ^= reg1

0000035c reg2 = 0x0

00000363 reg2 += 0x3e6fd176

00000373 reg2 ^= reg0

00000379 reg1 = 0x8

00000380 reg1 += 0x1800

0000038a [reg1] = reg2

00000391 reg1 = 0xc

00000399 reg1 += 0x1000

000003a3 reg1 = [reg1]

000003a9 reg0 ^= reg1

000003af reg2 = 0x0

000003b6 reg2 += 0x156d86fa

000003c5 reg2 += 0x66c93320

000003d5 reg2 ^= reg0

000003db reg1 = 0xc

000003e3 reg1 += 0x1800

000003ed [reg1] = reg2

000003f4 reg1 = 0x10

000003fc reg1 += 0x1000

00000406 reg1 = [reg1]

0000040c reg0 ^= reg1

00000412 reg2 = 0x0

00000419 reg2 += 0xe5dbc23

00000428 reg2 ^= reg0

0000042e reg1 = 0x10

00000436 reg1 += 0x1800

00000440 [reg1] = reg2

00000447 reg1 = 0x14

0000044f reg1 += 0x1000

00000459 reg1 = [reg1]

0000045f reg0 ^= reg1

00000465 reg2 = 0x0

0000046c reg2 += 0xd3f894c

0000047b reg2 ^= reg0

00000481 reg1 = 0x14

00000489 reg1 += 0x1800

00000493 [reg1] = reg2

0000049a reg1 = 0x18

000004a2 reg1 += 0x1000

000004ac reg1 = [reg1]

000004b2 reg0 ^= reg1

000004b8 reg2 = 0x0

000004bf reg2 += 0x324fe212

000004ce reg2 ^= reg0

000004d4 reg1 = 0x18

000004dc reg1 += 0x1800

000004e6 [reg1] = reg2

000004ed return

```

These 2 functions look like flag generation code!

Again, I won't go into much detail here. In short, these two functions perform a series of operations on your input, and then check whether the values obtained after these operations are equal to some array of values. This kind of operation involving sophisticated xor operations byte by byte are often seen in CTF. Since the flag generation function operates byte by byte, it is possible to brute force each character one by one and obtain the flag.

The full flag generation script can be found below.

## Flag generation

```python

primes = [13217, 13219, 13229, 13241, 13249, 13259, 13267, 13291, 13297, 13309, 13313, 13327, 13331, 13337, 13339, 13367, 13381, 13397, 13399, 13411, 13417, 13421, 13441, 13451, 13457, 13463, 13469, 13477, 13487, 13499, 13513, 13523, 13537, 13553, 13567, 13577, 13591, 13597]

magic = 123456789

secrets = [846786818, 1443538759, 1047515510, 359499514 + 1724461856, 241024035, 222267724, 844096018]

sanity = [1374542625 + 1686915720 + 1129686860, 842217029 + 1483902564, 1868013731, 584694732 + 1453312700, 223548744, 1958883726 + 1916008099, 1829937605 + 1815356086 + 253836698]

sanity_bytes = b''.join(i.to_bytes(4, 'little') for i in sanity)

def three_n_plus_one(n):

if n == 1:

return 0

else:

if n % 2 == 0:

n //= 2

else:

n = 3 * n + 1

t = three_n_plus_one(n)

return t + 1

chars = []

for i in range(28):

for c in range(255):

if ((c ^ (primes[i] & 0xff)) + three_n_plus_one(i + 1)) & 0xff == sanity_bytes[i]:

chars.append(c)

flag = b''

for i in range(7):

char_int = int.from_bytes(bytes(chars[i * 4: (i + 1) * 4]), 'little')

magic ^= char_int

flag_part = magic ^ secrets[i]

flag += flag_part.to_bytes(4, 'little')

print(flag)

```

## Flag

```

CTF{curs3d_r3curs1ve_pr1ntf}

```
