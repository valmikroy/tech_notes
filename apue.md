 # Notes on APUE

Describing Unix systems in a stricly linear fashion, without any forward references to terms that haven't been described yet, is nearly impossible.

### Ch1 - Unix System Overview 

The order in which any computer becomes usable

- Hardware
- Kernel 
- System calls wrapped in common library functions
- Shell
- Applications 



Directory is just an file with the list of entries and every directory has `dot` refering to its own directory and `dot dot` refering to parent directory.

Every process has its own working directory.

Unbuffered I/O is provided by the functions `open, read, write, lseek, and close`. These functions all work with file descriptors. 

All standard IO functions from `stdio.h` provides buffered I/O liker `printf` , `getc` and so on.

Any unix shell is taking a input which we type of the prompt through STDIN and then fork-exec that input to execute the program binary and then again return back to STDIN input prompt. 

Threads of the process are identified by thread ids and they are local to process itself.

`errorno` is set when any of the unix system function returns negative value. 

- this variable is global and defined as  `extern int errno`

- for thread friendly environment is it point to the function which return error number value 

  ```
  extern int *_ _errno_location(void); 
  #define errno (*_ _errno_location())
  ```

- never check value of `errno` unless you catch an error because its value never gets cleared.

- never setup value of `errno` to 0

- Various error strings starting with `E` are mapped to error numbers in `<errno.h>`

- nonfatal errors indicates that delay a little and try again later. Examples are `EAGAIN`, `EWOULDBLOCK` or others.



16 Supplementry group IDs are allowed to tied to single user.

Singnals are asynchronous events which occurs during program execution. They can be ignored unless they have a default assigned action or they can be caught by providing them some actions.



Process execution time split in wall clock, user and system space CPU time. Watch a difference between wall clock and sum of user and system space CPU time.



`malloc` is implemented on top of `sbrk` system call.



threads

- threads are just like processes but they share some things (memory, fds...) with other instances of the same *group*.
- a *tid* is actually the identifier of the schedulable object in the kernel (thread), while the *pid* is the identifier of the group of schedulable objects that share memory and fds (process).




### Ch3 - File I/O

File descriptors starts from 0 to `OPEN_MAX` where this upperlimit is controlled by `ulimit -n` for each user. System wide limit for each user is setup by sysctl param  `fs.file-max`.



```
#include <fcntl.h>
int open(const char *pathname, int oflag, ... /* mode_t mode */ );

Returns: file descriptor if OK, 1 on error
```



some interesting `oflags` apart from standard one

- O_DSYNC -  Have each **write wait** for physical I/O to complete, but **don't wait for file attributes** to be updated **if they don't affect the ability to read** the data just written.
- O_RSYNC - Have each **read** operation on the file descriptor **wait** until any **pending writes** for the same portion of the file are complete.
- O_SYNC - Have each **write wait** for physical I/O to complete, **including** I/O necessary to update **file attributes** modified as a result of the write.



Legacy create() function

```
#include <fcntl.h>
int creat(const char *pathname, mode_t mode);

Returns: file descriptor opened for write-only if OK, 1 on error
This exists for legacy reason and open() way to achieve the same  is

open (pathname, O_WRONLY | O_CREAT | O_TRUNC, mode);

But recommended may to do such activity in both read & write mode with following

open (pathname, O_RDWR | O_CREAT | O_TRUNC, mode);
```



File close 

```
#include <unistd.h> 
int close(int filedes);

Returns: 0 if OK, 1 on error
```



Seek through the file

```
#include <unistd.h>
off_t lseek(int filedes, off_t offset, int whence);

Returns: new file offset if OK, 1 on error
```



Seek can be 

- from the begining of the file + offset whence `SEEK_SET`
- from the current position + offset whence `SEEK_CUR`
- from the end of file + offset whence `SEEK_END`



check the current position or/and check if FD is seekable, that means not PIPE, FIFO or socket.

```
off_t currpos;
currpos = lseek(fd, 0, SEEK_CUR);
```

above return current position zero seeking or an error `ESPIPE` if FD is not seekable.

max size of `off_t` will be signed integer  `INT_MAX`  = 2^31 -  1  = 2147483647 (2TB)

Supported max file sizes by various operating systems are going to TBs and EBs. One strange thing is that max filesystem size set up lesser than max file size.



- seek position affects the size of the file written in the metadata which is always increments as the byte count between files start position and the seek counter increases. 
- actual data get written in the form of  filesystem blocks which are agnostick to seek counter. That means not every position of the seek pointer has to be filled with data and associated filesystem blocks.
- Above leads to holes in the file where output of the size shown by  `ls` and `du` differs. You can see holes in the file with `od -c`.

```
#include <unistd.h>
ssize_t read(int filedes, void *buf, size_t nbytes);

Returns: number of bytes read, 0 if end of file, 1 on error
```

Read FD

```
#include <unistd.h>
ssize_t read(int filedes, void *buf, size_t nbytes);

Returns: number of bytes read, 0 if end of file, 1 on error
```

Efficient reading depends on the chunk size defined in the last argument. This given optimial performance if setup to filesystem block size while reading from the disk. It can be set to larger till it can rip the benefits of filesystem's  read ahead mechanism.



Write FD

```
#include <unistd.h>
ssize_t write(int filedes, const void *buf, size_t nbytes);

Returns: number of bytes written if OK, 1 on error
```

File descriptor table organization

![FD table organization](images/FD%20tables%20organization.png)


There are some atomic operations can be performed in unix environment, like `O_APPEND` will `open` the file and  `lseek` till the end of it in the atomic manner.

`ioctl` is something to provide custom actions for the FD. Like in the past CD-ROM drive eject was the ioctl action provided through device driver for the `/dev/cdrom`.


```
#include <unistd.h>
int dup(int filedes); // returns lowest availble FD number duplicating filedes
int dup2(int filedes, int filedes2);

Both return: new file descriptor if OK, 1 on error
```

These functions get used for IO rdirection. `|` and `>` are some of them use `dup` functions. 



Null character devices
- `/dev/null` – accepts and discards all input written to it; provides an end-of-file indication when read from.
- `/dev/zero` – accepts and discards all input written to it; produces a continuous stream of null characters (zero-value bytes) as output when read from.




### APUE CH4 - Files and Directories 

- Various state functions like

  -  `stat` - To get stat information  of the file by filename 
  -  `fstat` - To get stat information of the open file using FD
  -  `lstat` - To get stat information of symbolic link and it's discovery 

- There is always a 

  - real user/group ID
  - effective user/group ID

  There is replacement happens of real ID value with effective ID if `set ID` bits are set of the executables.

  When setuid or setgid bit is set on the executable files so user who is executing those commands will get effective user or group ID setup according to the file's ownership. `/usr/bin/password` and `/usr/bin/mail` are examples.

- Directory 

  - Execute - allows to search through it and make you do `cd`
  - Read - allows to list files
  - Write - allows to create files

- File creation in the directory 

  - adapts group ID of the directory for the file if setgid is setup for the directory 
  - otherwise takes process group ID 

- `access()` uses effective User ID.

- `umask()`  - it is inverse mask set for the process to govern the permission bits of any file/directory creation.

- Sticky bit

  - Files - it was previously used for fast loading of the file 
  - Directory - it is used for restricted deletion of the file in that directory. Only owner can delete their own filess in the directory with `t` bit set. Example `/tmp`.

- chown() only superuser can change userID of the file. Non-superuser process can change group ID of the file.

- File sizes - Direcotry sizes are multiple of 16 and 512 , so symlink size is dpending on the pointed pathname.  Each entry in the dir has a fix size per unit, dir's size in in multiplication of the number of entries.

- File size - `du` counts actual blocks on the other hand `st_blksize` structure keeps a track of offset which is used by `ls`. `cat` command uses `read()` which returns `0` for any block are which has no data written.

- Directory `.` and `..` entries.

- Hardlink for Dir is not allowed

- call to `unlink()` and `open()` while using `tmpfile` , trick for mandotory clean up of temp files.

- Various stdlib functions (like `open()`, `exec()` ) follow symlink in the atomic fashion that is the reason we used use them in the deployments.

- call to `chdir()` does not change parent process's `pwd`. 

   





