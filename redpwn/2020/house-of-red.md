# house-of-red

**CTF:** redpwn 2020  
**Category:** misc  
**Original writeup:** <https://blog.robertchen.cc/2020/06/28/house-of-red/> — by **blog.robertchen.cc**. All credit goes to the original author; this is an archived copy.

---

### House of Red

House of Red was meant to be a continuation of zero-the-hero . It was a C++ web server with a similar relative write vulnerability.

long size; inline void resize ( long len) { if (len >= size) { free (buffer); buffer = ( char *) malloc (len); } size = len; } ... if (buffer != NULL ) { for ( int i = 0 ; i < size; i++) { buffer[i] = 0 ; } buffer[size] = 0 ; } int x; cin >> x;

house-of-red.cpp

By defining a negative size , we are able to perform the exact same attack as in zero-the-hero . There are three primary differences here.

We’re reading an int from C++ stdio functions. The competitor needs to make the observation that C++ uses the same file structs as in C. The grouping of the two problems was intended to hint towards this behavior. The overwrite is a null byte as opposed to \x30 . This means we are unable to perform an extension attack on ->_IO_buf_end . Instead, we must overwrite ->_IO_buf_base . The program immediately runs _Exit(0) (syscall exit) after reading in the int. Thus, we must find some self-contained way to trigger heap activity.

Diving deeper into file structs, the internal macro for reading characters looks as such.

It checks if ->_IO_read_ptr is greater than or equal to ->_IO_read_end . Intuitively, this represents if the pointer for the data that it’s reading has passed the end of the buffer. If so, it triggers the underflow operation which we looked at previously. Otherwise, it increments the read pointer, and returns.

The intended solution was to use cin >> x to trigger three consecutive reads. An important thing to note is that the C++ read int will immediately exit if it does not encounter a valid integer character. Thus, we need to be careful to adjust ->_IO_read_ptr such that it always points to a valid place.

_IO_new_file_underflow will also adjust ->_IO_read_end . However, this is by a predictable amount, which allows us to easily subtract this number from what we want ->_IO_read_end to be.

count = _IO_SYSREAD (fp, fp->_IO_buf_base, fp->_IO_buf_end - fp->_IO_buf_base); ... fp->_IO_read_end += count;

Overwriting the lowest byte of ->_IO_buf_base will allow us to control all of the pointers. The first read overwrites all of the file pointers. At the same time, it points ->_IO_read_ptr to a “1” char, as well as setting ->_IO_read_end to be equal to ->_IO_read_ptr after adjustments. This allows us to trigger another read in the same cin >> x code. We also adjust the ->_IO_buf_end pointer such that our second read overlaps with __free_hook .

The second read overwrites __free_hook with system, and primes the file struct for heap activity on the third read.

In order to trigger heap activity, I used the following code in _IO_new_file_underflow .

if (fp->_IO_buf_base == NULL ) { if (fp->_IO_save_base != NULL ) { free (fp->_IO_save_base); fp->_flags &= ~_IO_IN_BACKUP; } _IO_doallocbuf (fp); }

By setting ->_IO_buf_base to NULL and pointing ->_IO_save_base to “/bin/sh”, we are able to free something with "/bin/sh" . Note that previously we overwrote __free_hook with system, completing the exploit and allowing us to pop a shell.

The third time _IO_new_file_underflow is called, it will free our fake ->_IO_save_base , popping a shell.
