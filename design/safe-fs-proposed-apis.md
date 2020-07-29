# SAFE FS Proposed APIs
SAFE FS is the filesystem API for the SAFE Network and provides a standard way for applications to use the network as remote filesystem. This document describes the APIs and shows how they would support SAFE FUSE to provide access to safe storage mounted on the local filesystem.

The APIs include both a POSIX style API for relatively advanced applications, and a much simpler set of APIs for basic operations (such as single function call file save).

**Status:** All APIs are at proposal stage and subject to change. No working API exists.

### Approach to FS API Design
It is envisaged that we will begin with a subset of API functions and features, using the requirements of FUSE to develop a minimum initial POSIX style API. Later this can be extended to include more features of the POSIX filesystem. Where possible, the initial design and implementation should aim to provide for later features even if they are not implemented or exposed in the 'first' implementation.

### FUSE Operations and SAFE FS APIs
API features and design are guided by the POSIX standard and the wish to provide access to safe storage mounted on the local filesystem using a new SAFE FUSE application (which happily for us, assumes a POSIX style API). At the same time, the SAFE FS API should attempt to make common operations as simple as possible, and in order to do this we can provide two groups of API functions. A simplified API, and a more comprehensive POSIX stye API. 

The simplified SAFE FS API is outlined next, followed by the POSIX SAFE FS APIs, with the latter in two sections. One based on the high-level path based FUSE APIs, and then one based on the low-level inode FUSE API.

## SAFE FS Simplified API
The following APIs are proposed in order to make the writing of common file handling applications as easy as possible. The implementation will be built on top of the POSIX style APIs, and may be deferred until later when those are stable.

### File or Directory APIs
- **get_status()** - if file or directory exists, returns type (file, directory or symlink) and readily accessible metadata (e.g. size, time of last change etc.)

### File APIs

- **save_file(), copy_file(), load_file()**

### Directory APIs

- **list_directory()** - returns a FileInfo object for every File/Directory in the directory
- **copy_directory(), move_directory(), remove_directory()** - with an optional flag to force the deletion of a non-empty directory tree for remove_directory() only.


## SAFE FS POSIX Style API

As a default, the SAFE FS API feature can be assumed to be equvalent to the FUSE operation which it supports. Naming of SAFE FS APIs should reflect this, and I propose we use exactly the same names for the public API functions but inserting an underscore between 'words'. This maintains correspondence between each FUSE OP and its SAFE FS API, and with the naming of other SAFE APIs. So `getattr()` would be `get_attr()` in the SAFE FS API.

NOTE: The following information was taken from the FUSE documentation generated from the source code, but there is also a good description of both APIs including side-by-side comparission here:
- *FUSE Library Options and High- and Low-Level APIs* (CS Department, Stony Brook University [website](https://www.fsl.cs.stonybrook.edu/docs/fuse/fuse-article-appendices.html))

#### Mapping the FUSE High Level API
The following table shows the FUSE operations of the high level (path based) FUSE API classified by priority, and suggests SAFE FS APIs that correspond closely to the FUSE operations.
____
| FUSE OP (High Level) | Priority | Description                                              | SAFE FS API |
| -------------------- | -------- | -------------------------------------------------------- | ----------- |
| `init()`             | first    | called on filesystem init                                | tbd         |
| `destroy()`          | first    | clean up the filesystem (called on filesystem exit)      | ?           |
| `statfs()`           | second   | 'stat' the filesystem                                    | ?           |
| ---                  | ---      | ---                                                      | ---         |
| `getattr()`          | first*   | 'stat' a path                                            | ?           |
| `opendir()`          | first    | open a directory path                                    | ?           |
| `readdir()`          | first    | list a directory                                         | ?           |
| `releasedir()`       | first    | releases a directory descriptor                          | ?           |
| `create()`           | first    | open a new file                                          | ?           |
| `open()`             | first    | open an existing file                                    | ?           |
| `read()`             | first    | read content from a file                                 | ?           |
| `write()`            | first    | write content to a file                                  | ?           |
| `truncate()`         | first    | truncate a path to a given size                          | ?           |
| `fsync()`            | first    | sync content to storage (option to include metadata)     | ?           |
| `fsyncdir()`         | third    | as `fsync()` but for a directory                         | ?           |
| `flush()`            | third    | possibly flush cached data (not same as `fsync()`)       | ?           |
| `mkdir()`            | second   | create a directory                                       | ?           |
| `rmdir()`            | second   | remove a directory                                       | ?           |
| `rename()`           | second   | rename a file (or directory?)                            | ?           |
| `symlink()`          | second   | create a new symlink                                     | ?           |
| `readlink()`         | second   | resolve a symlink                                        | ?           |
| `chmod()`            | third    | change mode of a path                                    | ?           |
| `chown()`            | third    | change ownership of a path                               | ?           |
| `release()`          | third    | releases a file descriptor                               | ?           |
| `link()`             | opt.     | create a new hard link to an existing file               | tbd         |
| `unlink()`           | opt.     | unlink a file path (delete if no hard links remain)      | tbd         |
| ---                  | ---      | ---                                                      | ---         |
| `utimens()`          | third    | change atime/ctime of a file                             | ?           |
| `setxattr()`         | third    | set extended attributes (platform dependent)             | ?           |
| `getxattr()`         | third    | set extended attributes (platform dependent)             | ?           |
| `listxattr()`        | third    | list extended attributes (platform dependent)            | ?           |
| `removexattr()`      | third    | remove extended attributes (platform dependent)          | ?           |
| ---                  | ---      | ---                                                      | ---         |
| `lock()`             | second   | perform a POSIX file locking operation                   | ?           |
| `flock()`            | opt.     | perform BSD file locking operation                       | tbd         |
| `access()`           | third    | called to check permissions before accessing a file      | ?           |
| ---                  | ---      | ---                                                      | ---         |
| `write_buf()`        | third    | like `write()` but no copying needs to take place        | ?           |
| `read_buf()`         | third    | like `read()` but no copying needs to take place         | ?           |
| `poll()`             | opt.     | poll for IO readiness event                              | tbd         |
| `fallocate()`        | opt.     | allocate storage for file                                | tbd         |
| `copy_file_range()`  | opt.     | copy data from one file to another (avoiding FUSE stack) | tbd         |
| `lseek()`            | opt.     | Find next data or hole after the specified offset        | tbd         |
| ---                  | ---      | ---                                                      | ---         |
| `ioctl()`            | opt.     | control a device                                         | n/a         |
| `mknod()`            | opt.     | make file node (not needed if `create()` is implemented) | n/a         |
| `bmap()`             | opt.     | map block index from file to device                      | n/a         |
____
Notes:
*`getattr()` is called very frequently so the information it returns may need to be cached to improve performance.

FUSE APIs have been prioritised as 'first', 'second', 'third' or 'opt.' (optional or not applicable). The first SAFE FS API must support at least all the 'firsts', and implement or anticipate support for 'second' and 'third' features where this can be done without much additional development work.

#### Mapping the FUSE Low Level API
The following table shows the FUSE operations of the high level (path based) FUSE API classified by priority, and suggests SAFE FS APIs that correspond closely to the FUSE operations.

____
| FUSE OP (Low Level) | Priority | Description                                              | SAFE FS API |
| ------------------- | -------- | -------------------------------------------------------- | ----------- |
| `init()`            | first    | called on filesystem init                                | ?           |
| `destroy()`         | first    | clean up the filesystem (called on filesystem exit)      | ?           |
| `statfs()`          | second   | 'stat' the filesystem                                    | ?           |
| ---                 | ---      | ---                                                      | ---         |
| `lookup()`          | first    | look up a directory entry by name and get its attributes | ?           |
| `forget()`          | first    | forget about an inode (reduces `lookup()` count)         | ?           |
| `forget_multi()`    | first    | forget about multiple inodes                             | ?           |
| ---                 | ---      | ---                                                      | ---         |
| `setattr()`         | third    | set file attributes (cf. `<sys/stat.h>`*)                | ?           |
| `getattr()`         | first*   | 'stat' a path                                            | ?           |
| `opendir()`         | first    | open a directory path                                    | ?           |
| `readdir()`         | first    | list a directory                                         | ?           |
| `releasedir()`      | first    | releases a directory descriptor                          | ?           |
| `create()`          | first    | open a new file                                          | ?           |
| `open()`            | first    | open an existing file                                    | ?           |
| `read()`            | first    | read content from a file                                 | ?           |
| `write()`           | first    | write content to a file                                  | ?           |
| `fsync()`           | first    | sync content to storage (option to include metadata)     | ?           |
| `fsyncdir()`        | third    | as `fsync()` but for a directory                         | ?           |
| `flush()`           | third    | possibly flush cached data (not same as `fsync()`)       | ?           |
| `mkdir()`           | second   | create a directory                                       | ?           |
| `rmdir()`           | second   | remove a directory                                       | ?           |
| `rename()`          | second   | rename a file (or directory?)                            | ?           |
| `symlink()`         | second   | create a new symlink                                     | ?           |
| `readlink()`        | second   | resolve a symlink                                        | ?           |
| `release()`         | third    | releases a file descriptor                               | ?           |
| `link()`            | second   | create a new hard link to an existing file               | ?           |
| `unlink()`          | second   | unlink a file path (delete if no hard links remain)      | ?           |
| ---                 | ---      | ---                                                      | ---         |
| `setxattr()`        | third    | set extended attributes (platform dependent)             | ?           |
| `getxattr()`        | third    | set extended attributes (platform dependent)             | ?           |
| `listxattr()`       | third    | list extended attributes (platform dependent)            | ?           |
| `removexattr()`     | third    | remove extended attributes (platform dependent)          | ?           |
| ---                 | ---      | ---                                                      | ---         |
| `getlk()`           | second   | test for a POSIX file lock                               | ?           |
| `setlk()`           | second   | acquire, modify or release a POSIX file lock             | ?           |
| `flock()`           | opt.     | perform BSD file locking operation                       | tbd         |
| `access()`          | third    | called to check permissions before accessing a file      | ?           |
| ---                 | ---      | ---                                                      | ---         |
| `write_buf()`       | third    | like `write()` but no copying needs to take place        | ?           |
| `read_buf()`        | third    | like `read()` but no copying needs to take place         | ?           |
| `poll()`            | opt.     | poll for IO readiness event                              | tbd         |
| `fallocate()`       | opt.     | allocate storage for file                                | tbd         |
| `copy_file_range()` | opt.     | copy data from one file to another (avoiding FUSE stack) | tbd         |
| `lseek()`           | opt.     | Find next data or hole after the specified offset        | tbd         |
| ---                 | ---      | ---                                                      | ---         |
| `ioctl()`           | opt.     | control a device                                         | n/a         |
| `mknod()`           | opt.     | make file node (not needed if `create()` is implemented) | n/a         |
| `bmap()`            | opt.     | map block index from file to device                      | n/a         |
| ---                 | ---      | ---                                                      | ---         |
| `retrieve_reply()`  | second   | callback function for the retrieve request               | ?           |
| `readdirplus()`     | third    | read directory with attributes                           | ?           |
| ---                 | ---      | ---                                                      | ---         |
____
Notes:
*`getattr()` is called very frequently so the information it returns may need to be cached to improve performance.

FUSE APIs have been prioritised as 'first', 'second', 'third' or 'opt.' (optional or not applicable). The first SAFE FS API must support at least all the 'firsts', and implement or anticipate support for 'second' and 'third' features where this can be done without much additional development work.

### Comparing the Low- and High-level FUSE APIs

Most operations have equivalents with identical names in both high-level (`fuse_operations`) and low-level (`fuse_lowlevel_ops`) FUSE APIs. Areas of difference are summarised below.

More information is available on both APIs and an excellent side-by-side comparission is available in *FUSE Library Options and High- and Low-Level APIs* (CS Department, Stony Brook University [website](https://www.fsl.cs.stonybrook.edu/docs/fuse/fuse-article-appendices.html))

#### Low-level Only Operations
| Operation    | Description                                      |
| ------------ | ------------------------------------------------ |
| `chmod()`    | change mode of a path                            |
| `chown()`    | change owner of a path                           |
| `truncate()` | truncate a path to a given size                  |
| `lock()`     | perform a POSIX file locking operation           |
| `utimens()`  | change atime/ctime of a file                     |
| `read_buf()` | like `read()` but no copying needs to take place |

#### High-level Only operations
| Operation          | Description                                              |
| ------------------ | -------------------------------------------------------- |
| `lookup()`         | look up a directory entry by name and get its attributes |
| `forget()`         | forget about an inode (reduces `lookup()` count)         |
| `forget_multi()`   | forget about multiple inodes                             |
| `setattr()`        | set file attributes (cf. `<sys/stat.h>`*)                |
| `getlk()`          | test for a POSIX file lock                               |
| `setlk()`          | acquire, modify or release a POSIX file lock             |
| `retrieve_reply()` | callback function for the retrieve request               |
| `readdirplus()`    | read directory with attributes                           |

*see [<sys/stat.h>](https://pubs.opengroup.org/onlinepubs/007908799/xsh/sysstat.h.html)

#### Commentary on differences between APIs

Low-level API operaions are provided with a structure or buffer for any returned information, and use the return value to indicate success or an error code.

Whereas the high-level operations are provided with a request handle which will be used when calling a reply function. Different operations are required to call specific reply functions to return specific data (e.g. `fuse_reply_entry()` to return information about a directory entry). For error conditions, the request handle will be passed to the FUSE error reply API (e.g. `fuse_reply_err()`). See the low-level FUSE operation definitions to learn which reply functions are allowed to be used by each low-level FUSE operation.

In the high-level API `utimens()`, `chown()`, `chmod()` provide modification of ctime/mtime, owner and permissions respectively. In the low-level API these are all applied using `setattr()`.

`lock()` provides POSIX file locking in the high-level API, which is provided by `getlk()` and `setlk()` operations in the low-level API.

`read_buf()` in high-level API - check for equivalent in low-level API.
`retrieve_reply()` in low-level API - to be investigated.

## References
- FUSE - Filesystem in Userspace ([Wikipedia](https://en.wikipedia.org/wiki/Filesystem_in_Userspace))

**FUSE Documentation**
- libfuse documentation ([github.io/doxygen](https://libfuse.github.io/doxygen/index.html))
  - `fuse_operations` structure and definitions in [libfuse](https://libfuse.github.io/doxygen/structfuse__operations.html)
  - `fuse_low_levelops` structure and definitions in [libfuse](https://libfuse.github.io/doxygen/structfuse__lowlevel__ops.html)
  - [list of example files](https://libfuse.github.io/doxygen/files.html) including minimal FUSE examples [hello.c](https://libfuse.github.io/doxygen/example_2hello_8c.html) and [hello_ll.c](https://libfuse.github.io/doxygen/example_2hello__ll_8c.html)
- *FUSE Library Options and High- and Low-Level APIs* (CS Department, Stony Brook University [website](https://www.fsl.cs.stonybrook.edu/docs/fuse/fuse-article-appendices.html))

**POSIX Documentation**
- `stat` structure for *fstat(), lstat(), stat()* in [<sys/stat.h>](https://pubs.opengroup.org/onlinepubs/007908799/xsh/sysstat.h.html)

**Linux Documentation**
- *nix system error codes: [<errno.h>](https://www-numi.fnal.gov/offline_software/srt_public_context/WebDocs/Errors/unix_system_errors.html)