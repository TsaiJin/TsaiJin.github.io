---
title: Read and Write functions in linux
date: 2016-04-12 21:43:21
tags: [linux, C]
categories: Technology
---

Resulting from work, I have learned I/O models of the linux operating system during these days. In linux operating system, various read and write APIs are provided to user space for use. Comparasions between them are illustraed below.



### read()

```
#include <unistd.h>
ssize_t read(int fd, void *buf, size_t count);
```

`read()` is the basic read function in linux environment. It attempts to read up to count bytes from file descriptor fd into the buffer starting at buf.
It will start from current file offset. And the current file offset will be increased by the number of bytes read. However, if current file offset is at or past the end of operating file, no bytes will be read into buffer.
On success, the number of bytes read is returned (zero indicates end of file), and the file position is advanced by this number. It is not an error if this number is smaller than the number of bytes requested; this may happen for example because fewer bytes are actually available right now (maybe because we were close to end-of-file, or because we are reading from a pipe, or from a terminal), or because `read()` was interrupted by a signal. On error, -1 is returned, and errno is set appropriately. In this case it is left unspecified whether the file position (if any) changes.[Reference](http://linux.die.net/man/2/read)
`read()` is thread safe in the sense that your program will not have undefined behavior (crash or hung) if multiple threads perform IO on the same open file using at once. But the order and atomicity of these operations could vary greatly depending on the type of the file and the implementation of program.


<!-- more -->

### lseek()

```
#include <sys/types.h>
#include <unistd.h>
off_t lseek(int fd, off_t offset, int whence);
```

The `lseek()` function repositions the offset of the open file associated with the file descriptor fd to the argument offset according to the directive whence
The directive whence can be as follows:
**SEEK_SET** The offset is set to offset bytes.
**SEEK_CUR** The offset is set to its current location plus offset bytes.
**SEEK_END** The offset is set to the size of the file plus offset bytes.
When whence is as the last one, `lseek()` function allows the file offset to be set beyond the size of file while the file size still keeps the same. If data is latter write at this point, subsequent reads of the data in the gap (as a “hole”) return null bytes until data is actually written to this gap. [Reference](http://linux.die.net/man/2/lseek)
There are some special usage methods about `lseek()`:

1. `lseek(int fildes, 0, SEEK_SET)`:
   move the read or write position to the start of the file
2. `lseek(int fildes, 0, SEEK_END)`:
   move the read or write position to the end of the file
3. `lseek(int fildes, 0, SEEK_CUR)`:
   get the current read or write position of the file

With `lseek()`, you can implement the random I/O models of read and write easily.

### pread()

```
#include <unistd.h>
ssize_t pread(int fd, void *buf, size_t count, off_t offset);
```

Similar to `read()`, `pread()` attempts to read count bytes from file descriptor fd at offset into buffer starting at buf. Unlike `read()`, the offset here will be not changed after the call of `pread`
In many cases `pread()` is the only option when you’re dealing with threads reading from a database or such.
Compared with `read()`, `pread()` does more than `read()` on account of the time to positioning offset. From the work mechanism of `pread()`, we can find that it is like the combination of `read()` and `lseek()`. Nevertheless, performance of `pread()` is quite higher than the combination of `read()` and `lseek()`.
As mentioned above, `read()` function will be in mess when multiple threads or processes perform IO operations on the same open file because it will increase the current file offset. On the flip side, `pread()` do not change the position in the open file so it is more convenient to using in the scenario of multiple threads and processes.

### pwrite()

```
#include <unistd.h>
ssize_t pwrite(int fd, const void *buf, size_t nbytes, off_t offset);
				Returns: number of bytes written if OK, −1 on error
```

Calling `pwrite()` is equivalent to calling `lseek()` followed by a call to `write()`. Instead of calling `lseesk()` and `write()` separately, the combination of `lseek()` and `write()` is atomic operation in `pwrite()`.
