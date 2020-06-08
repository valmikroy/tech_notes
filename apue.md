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













