# SAFE FS Design and implementation
This is a tentative design with notes on selected areas of implementation to help identify options for implementing a SAFE Network filesystem and the [SAFE FS API](./safe-fs-proposed-apis.md) (which is also at an early stage).

To Do list of areas to discuss:
- summary of POSIX filesystem:
  - [x] inodes, directories and files
  - [x] symlinks
  - [x] hard-links
  - [ ] file locking (needs more work)
- Implementation options for:
  - [x] inodes with support for hard-links
  - [ ] file locking
  - [ ] versioning for private and published data (and the perpetual web)

- [ ] **To do:** File locking needs further investigation.

# Summary of POSIX filesystem
## inodes, Directories and Files
**inodes** are structures which hold metadata about the objects in a filesystem. In Unix/Linux operating systems the filesystem in a device partition is typically implemented using a fixed number (2^64) of inodes, which altogether hold the structure and metadata for all files and directories. Each inode corresponds to an object or entry in the filesystem (e.g. a file, directory or symlink) and is referred to in the POSIX filesystem APIs by an inode number (an unsigned 64 bit integer). Typical metadata stored in an inode includes creation and modification time, operating system owner, group and mode (access controls).

**directories** are inode objects which have a list of directory entries which map names to inode numbers. A file or directory is therefore independent of its name or location, and either can be changed without touching the entry's inode object, and just modifying an entry in a directory.

**files** are inodes which have content, typically a list of block locations on a storage device (disk or memory). These locations can change when a file is modified, a device is defragmented etc, and causing the inodes list of block locations to be updated.

The **filesystem** always has at least one directory known as 'root' and is the ultimate directory, or the base path for all other directories and files in the filesystem. The filesystem 'root' may appear at different paths on computer device (e.g. at '/' or '/tmp' etc.), the mount point.

## Symlinks
A **symlink** (short for 'symbolic link') is an inode which holds a path which acts like a pointer to another location. This can be used to make a file or directory appear in more than one filesystem path, and for many applications this will look as if the file or directory pointed to by the symlink, is at the location of the symlink.

Changes to the destination will therefore be reflected when anything accesses it via the symlink.

If the desination is renamed, moved or deleted, the destination of the symlink will no longer point to anything and the destination file or directory will no longer be accessible via the symlink.

This is different to **hard-links** described next.

See: Symbolic link ([Wikipedia](https://en.wikipedia.org/wiki/Symbolic_link))

## Hard-links
A **hard link** is a named entry in a directory which holds an **inode** number. So in practice all files visible in the system have at least one hard-link, from the directory in which they appear. But in POSIX systems, more than one directory entry can refer to the same inode number, in which case the same inode appears in more than one location.

Each inode keeps track of the number of such links as part of its metadata, and will be deleted from the filesystem when this count reaches zero. However, this link count includes links from open processes accessing the file not just links from the filesystem itself. This feature means that an inode which has been removed completely from the filesystem is kept until any active processes have closed their access to the file, or been terminated, and its link count reaches zero.

### Do we need hard-links?
Hard links mostly go unseen by applications, but are very useful in multithreaded operating systems. For example, they ensure that a program, library or script can continue working even if it is deleted from the filesystem by another process - rather than causing the running program to behave unpredictably. This can be useful when developing and updating programs or scripts, and for updating system libraries without the need to shut the system down.

Hard-links can also be useful for users to create 'views' of data held in the filesystem and are more robust than symlinks, which are pointers to a location rather than an object. This means that no directory entry that is a hard link is preferred over any other, and can be removed or moved without affecting other links, allowing multiple directory hierarchies to co-exist. Uses include sharing access to files and directories in different contexts, and making snapshots of large directory trees without duplicating files (cf. [rsnapshot](https://github.com/rsnapshot/rsnapshot)).

**Summary:** hard-links are very useful in some circumstances, but easily overlooked so we should be cautious about omitting them in the long term.

## File Locking
File locking allows multiple programs or processes to access the same file or directory in a co-operative way to avoid unwanted side effects from data being changed unexpectedly by another process.

Several locking mechanisms exist, even on a single operating system, and I think none of them are able to prevent an unco-operative process from interfering with a locked file.

- [ ] **To do:** File locking needs further investigation.

Ref: File locking ([Wikipedia](https://en.wikipedia.org/wiki/File_locking))
# SAFE FS Implementation

## FUSE low-level APIs
We propose using the low-level FUSE APIs so that we can support hard-links and have prototype code showing how this might be implemented using the TreeCDRT.

## TreeCDRT and SAFE FUSE Implementation
Internally the TreeCDRT uses UUIDs to provide global node identifiers across all replicas. Directory and symlink don't need inode objects and so their data and metadata is held on the corresponding tree-node.

File metadata is held on separate inode objects so that multiple directory entries can reference the same file. Each inode tree-node holds the file metadata, and keeps track of the number of hard-link references to the file. If this count falls to zero, there are no longer any links in the filesystem to the node and it can be discarded.

Tree-nodes all have a UUID making directories, symlinks and file inodes globally unique across all replicas, which is a pre-requisite for the tree-move algorithm.

In a local filesystem objects are created to represent inodes when needed and exposed to the FUSE implementation using inode numbers or ino's (u64 values). These objects are reference counted on `lookup()`, dereferenced when `forget()`, and can be discarded when the reference count falls to zero (similar to a file handle). u64 ino's are local to the filesystem and because they are ephemeral can be allocated sequencially and will take too long to run out for there to be any chance of this happening. (N.B. This is how a networked filesystem implementation of FUSE works).

### Tree CDRT Algorithm
Details of the algorithm are included in the following paper and online presentation:

-  *A highly-available move operation for replicated trees and distributed filesystems* by Kleppmann et al (PDF: [online](https://martin.kleppmann.com/papers/move-op.pdf))
-  *CRDTs: The Hard Parts* by Martin Kleppmann (https://martin.kleppmann.com/2020/07/06/crdt-hard-parts-hydra.html). The section on "Moving sub-trees of a tree" begins at 35:50, but the whole video is a very good introduction to the current state of CDRTs. Notes and timings by `@happybeing` to help locate information in the video are online [here](https://forum.safedev.org/t/filetree-crdt-for-safe-network/2833/2?u=happybeing).

#### Pruning the `FileTree` CDRT Log
It may be desirable to prune the CDRT log as it grows with each mutation of the FileTree and its files. We should though ensure this is only done when a user is willing to lose versioning earlier than a given point, or at given granularity (if we allow pruning of intermediate versions).

It would be possible to maintain the full CRDT log while also reducing the portion of the log which is present in the FileTree. Each FileTree can have an associated archive of its log which has pruned entries added to it, so that the complete log can always be obtained, and all versions revisited if needed. The overhead of retrieving the pruned entries should be acceptable if pruning is managed in a way to make this a rare activity.

## File Locking

TODO - decide what to support and how to implement

## Versioning for Private and Published Data
How to support versioning for perpetual data within SAFE FS is currently unknown, so what follows are some thoughts for that discussion.

I think we need two things:
1) Perpetual versioned access to public data. This is inherent in the underlying data types, but not yet understood for a `FileTree` and the latter may not be necessary for the former.
2) Querying the version of a `FileTree` and the ability to access the complete content at every earlier version.

If it proves difficult to satisfy both using the same approach, it may be worth achieving these two goals by different means.

For example 2) may require mounting of and access to a `FileTree` as a complete read-only filesystem, and this would be a large burden if any web or other client has to do this (for a potentially large object) when retrieving every version it wishes to access.

To achieve 1), we could take a snapshot of a `FileTree` or a subset of a `FileTree` that is to be published, and store this in a similar way to the versioned `FilesContainer`, the forerunner to this implementation. This scheme would be suitable for general publishing tasks such as websites via versioned NRS container entry, accessible from devices with modest resources. It will mean that a `FileTree` cannot become a *Published* `FileTree` (like for example SAFE `Sequence`, although access to it can still be shared. To publish, part or all of a FilesTree would be copied to a read-only, versioned container (such as a `FilesContainer`. As this will be read-only, hard-links could be simulated by referencing the same immutable file in multiple directory entries).

For 2), if we are unconstrained by the need for access on modest clients, it becomes accceptable to work like a versioned backup, mounting the whole tree at a particular version and having the full SAFE FS APIs operating on that version (possibly restricted to read-only fuctionality). 

Presumably this could be achieved using the rollback feature of the CRDT log, built into the `FileTree` itself with no additional work.

# References

- **Symbolic link** ([Wikipedia](https://en.wikipedia.org/wiki/Symbolic_link))
- **File locking** ([Wikipedia](https://en.wikipedia.org/wiki/File_locking))

* **inode** - useful summary of Unix/POSIX implementation of inodes and hard-links ([Wikipedia](https://en.wikipedia.org/wiki/Inode))
  - Note: maybe only MacOS allows hard-links to directories? (see "Implications")

* *UNIX/LINUX File System - Directories, inodes, Hard-links* by Ian D. Allen (2019, [course notes](http://teaching.idallen.com/cst8207/13w/notes/450_file_system.html))

