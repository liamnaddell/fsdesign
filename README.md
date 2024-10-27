# Design

## Q/A:

How does this design implement a speedy `find` command:
1. We bulk transfer directory entries between libc, procnto, and the file system driver to minimize syscalls
2. All opens are openat's, so each 1000 entry directory need only be searched once.
    1. `find` will perform a brute force search of the directory entries once it has opened the directory
    2. Opening directories requires a filesystem to do a brute force search of the tree it is attempting to open. If each directory contains 1000 entries, a mere 5-level-deep tree would require each open to search 1000^5 entries, which is completely unacceptable. By designing `open` to be converted to an `openat(CWD,rel_path)`, we allow a properly implemented `find` to linearly search the entire tree.
3. The filesystem itself is based off of a design similar to the `ext2/3` filesystem series. This design is likely sufficient to implement a speedy `readdir`, as it's main performance impact is a slightly longer read time vs an extent+b-tree based design like xfs, btrfs, or ext4. Slightly longer here meaning 5-10 extra block reads from disk, assuming no caching.

## TODO

* Pretty pictures!
* https://www.qnx.com/developers/docs/8.0/com.qnx.doc.neutrino.getting_started/topic/s1_resmgr_Finding_server.html
* File systems as shared libraries
* Thomasf https://www.qnx.com/developers/docs/8.0/com.qnx.doc.neutrino.sys_arch/topic/resource_FILESYSresmgr.html
* Caching
* Synchronization.
* Handle deleting dir entries

## Understanding the task

* `find . -name <name>`
* Maybe less, but need to handle maximum scale of ~1000 entries 
* Readdir based

### An implementation of our "program"

```c
#include <dirent.h>

...
int find(const char *filename, const char *dirname) {
    //chdir is acting as a simplified version of "openat"
    chdir(dirname);
    DIR *d = opendir(".");
    //NOTE: Ignoring error handling
    struct dirent *de = readdir(d);
    while (de != NULL) {
        //IGNORING: poor filesystem support for d_type
        //IGNORING: searching symbolic links, character devices, fifos, etc...
        //IGNORING: a useful "find" tool that prints the full directory path.
        if (de->d_type == DT_DIR) {
            find(filename,d->d_name);
        } else if (de->d_type == DT_REG) {
            if (strcmp(de->d_name,filename) == 0) {
                printf("Found filename %s in directory %s\n",filename,dirname);
            }
        }
        de=readdir(d);
    }
    return 0;
}
...
```

1. We implemented DFS here, but others might implement BFS search, it's allegedly faster.
2. Strcmp is probably not that expensive for O(~10-50) average case elements.
3. What's `de`'s lifetime? The manual doesn't say! I assume de lives until the next readdir from within the same thread.

## How might we implement this in libc?

I think it would be best to implement readdir and opendir as special cases of read(), and open(). As such, when you read(), or open() a directory, you should be able to read() directory entries.

One nice note of our "find" implementation is that we do NOT have to call `stat` on the file/directory. It is important to make sure to be able to handle the common case without a `stat` call.

```c

//NOTE: The directory entry structure is defined by procnto. This is because we need each filesystem to report directory entry data in the exact same format, regardless of the underlying differences.
struct procnto_dirent {
    char d_type;
    unsigned short d_reclen; /* good idea from glibc! */
    const char *d_name; /* name */
};

#define BSIZE 512
typedef struct DIR {
    int fd;
    //buf for readdir
    //NOTE: we could also mallocate this when reading the 
    // first dir entry, and de-allocate on the last one
    char rd_buf[BSIZE];
    //dir entry for readdir to return pointers to.
    struct dirent de;
    //Our current progress in rd_buf;
    char *rd_progress;
};


DIR *opendir(const char *filename) {
    //O_DIRECTORY causes error if not directory
    int fd = open(filename,O_DIRECTORY);
    //...handle errors...
    
    DIR *d = malloc(sizeof(DIR));
    //...handle errors...

    d->fd = fd;
    d->rd_progress = NULL;
    return d;
}

struct dirent *readdir(DIR *dirp) {
    //1. If we ran out of data to return, read() some more from fd
    //2. If we have data to return, "parse" it from rd_buf into dirp->de, 
    //      and return a pointer to dirp->de
    //3. use rd_progress to facilitate this.
    struct dirent fake_de;

    if (dirp->rd_progress != NULL) {
        //read the data that's there
        fake_de = (struct dirent *) dirp->rd_progress;
        dirp->de = *fake_de;
        dirp->rd_progress += sizeof(struct dirent);

        //increment read progress
        assert(dirp->rd_buf+BSIZE > dirp->rd_progress)
        if (dirp->rd_buf - dirp->rd_progress < sizeof(struct dirent)) {
            dirp->rd_progress = NULL;
        }
        return &dirp->de;
    }

    //NOTE: Not handling read() returning less than BSIZE to simplify logic.
    ssize_t res = read(dirp->fd,dirp->buf,BSIZE);

    return &dirp->de;
}

//LIBC internal.
//must be shallowly copyable.
struct file {
    unsigned char type;
    //... Open might as well also call stat() while we are at it...
    //Once the "stat" is read, we discard the cached stat results.
    //So if the user calls stat again, they will be supplied with fresh data
    //Token for VFS.
    int global_file_id;
};

int open(const char *pathname, int flags) {
    //reserved,pid,chid,index,flags
    //VFS_PROCESS = procnto
    coid = ConnectAttach (0, VFS_PROCESS, VFS_CHANNEL, 0, 0);

    struct file f;
    //we store CWD as a "struct DIR *"
    int dir_id = libc_get_cwd()->global_fd;
    //This is basically an openat + stat.
    int retval = MsgSend(coid,"Please open and stat {pathname} for me, I am at directory {dir_id}",msg_len,&f,sizeof(struct file));
    //handle error;

    int fd = libc_add_file(&f);
    return fd;
}

ssize_t read(int fd, void *buf, size_t count) {
    //a file descriptor is an index into a table of open files!
    struct file *fi = libc_get_file(fd);
    //...error handling...

    long a = MsgSend(fi->coid,"Please get file data: {fi->global_file_id}, starting at {fi->loc}",msg_len,buf,count);

    return a;
}
```

In summary, readdir() is a wrapper around read(), which bulk receives directory entries from VFS. We improve efficiency by handling all the information in large chunks, and having the readdir() function only make syscalls when absolutely necessary.

We also cache stat calls, assuming that `find` will likely want this information in case of a more complex search.

## What happens at the VFS layer?

The VFS operates by dispatching FS messages to the appropriate drivers. In our case, we need to take read requests (on files and directories), and forward them, knowing also that QNX operates using unioned filesystems.

How might a VFS work? Well the inputs to a VFS are a set of mounts and mount points, and the VFS will use this information to dispatch reads/writes.

Q: Where does the VFS live in the system? A: I believe it is in procnto aka /sbin/init.

As such, we need a mechanism for registering mounts.

When a user mounts a filesystem, we need to determine which filesystem driver is appropriate. The superblock should (e.g. will) make this clear.

NOTE: I wrote this design before looking up the QNX documentation, and seeing how QNX actually implemented this. I think my guesses for how VFS works were pretty good, but the process of registering drivers and sending superblocks diverged significantly from how QNX seems to have implemented this.

```c
//File system driver makes itself known to procnto:
...
int register_driver() {
    long a = MsgSend(coid_with_procnto,"I am a filesystem driver",msg_len,NULL,0);
    return a;
}
```

```c
//Procnto creates a list of filesystems:
int handle_new_filesystem() {
    while (true) {
        struct _msg_info mi;
        //check permissions, etc.
        rcvid_t r = MsgReceive(driver_register_chid, NULL, 0, &mi);

        procnto_register_filesystem_driver(mi.pid,mi.coid);
        MsgReply(r,0,NULL,0);
    }
}

//NOTE: This presumes the filesystem driver is active + responding to messages at all times...
//NOTE: This might force one driver process to handle multiple filesystems across devices, maybe not what we want.
int handle_mount_request(device d, const char *loc) {
    bool found_device_driver = false;

    struct fs_query_reply r;
    for (auto fs_driver in procnto_filesystem_drivers()) {
        long a = MsgSend(fs_driver.coid,"Does this superblock look like yours?" + d.blocks[0],n,&r,sizeof(struct fs_query_reply));
        if (a>0 && r.reply == "Yes, this is my filesystem") {
            procnto_add_mount(loc,fs_driver);
        } else if (r.reply == "Corrupted") {
            printf("fs_driver: %s, invalid or corrupted superblock\n",fs_driver.name);
            return -1;
        }
        //Reply is "Not Mine"
    }
    printf("If this device contains a filesystem, no installed filesystem drivers recognize it\n");
    return -1;
}

//Handle incoming openat+stat()'s Assuming this is a message sent to us by someone's libc.
int handle_openstatat(int starting_directory_id, const char *relative_path) {
    struct open_query_reply r;
    struct procnto_dir *d = procnto_get_file(starting_directory_id);
    if (d == NULL || !d->type == TYPE_DIR) {
        return -1;
    }

    //Remove ".." and "." and "//"
    //NOTE: We have to be careful to handle openat(".","..") properly. In this case, it is non-trivial to remove "..".
    // We do know that procnto has to handle this, since ".." might lie across the FS boundary, so by the time these paths make it to the fs driver, 
    // they are non-relative.
    const char *simplified_path = remove_relative(relative_path);
    for (auto mount : mounttable.get_mounts()) {
        //we know which filesystem d resides on, and where "d" is in the VFS.
        //However, relative path could be for a mount that starts under d.
        if (mount.path() prefixes d->path + relative_path) {
            //This is a unioned filesystem implementation, which prioritizes files close to the root in case of name conflicts.
            //It might be better to prioritize lowest mounted filesystem first.
            //Again, we are effectively sending an openat + stat.
            long a = MsgSend(mount.coid,"Open and stat file {full_path - mount.path()} starting at {d->local_id}",n,&r,sizeof(open_query_reply));

            if (r.success) {
                return r.file_id;
            }
        }
    }
}

//Assuming we are receiving a read request for global_file_id
//would be sent by MsgSend
ssize_t handle_read(int global_file_id, void *buf, size_t size, size_t loc) {
    //Procnto will manage a global list of open files.
    //Global file id's are managed by procnto
    //local_file_id's are assigned by the filesystem.
    //procnto maintains a mapping from global_file_id -> local_file_id
    struct qnx_file *f = procnto_get_file(global_file_id);
    //TODO: I'm not distinguishing here between "read" and "read_at". It's likely we want read() to only exist within libc, and read_at() to exist everywhere else.
    long a = MsgSend(f->driver->coid,"Please get this file data {f->local_file_id}, starting at {loc}",buf,size);
    return a;
}
```

## What happens inside the filesystem ResMgr?

* The filesystem resmgr will be structured like a normal user program. It will receive incoming FS messages from procnto, and dispatch those queries, using its exclusive access to a block device.
* To reiterate, a filesystem ResMgr manages a block device.

Q: How do FS drivers assert EXCLUSIVE access to a block device.
A: They link to io-blk.

We have 2 ideas: An extent based design, or an indexed based design.

Extents:
* Pro: More space efficient in the inode, more contiguous => more speed.
* Pro: SATA seems to allow you to request multiple contiguous blocks, so filesystems which allocate contiguously are likely to have high performance.
* Con: Expensive to track. Btrfs, xfs, and ext4 all use b-trees to track extents, which is probably too difficult of a design to implement here.

Indexed files:
* More simple to implement
* Even small block size (512B) is likely to store >100 directory entries. So one inode with ~10-13 direct pointers is likely to store at least 1000 as wanted.
* No external fragmentation.
* Con: data is scattered all around the disk which can have speed impacts.

NOTE: This code is adapted from my previous group project, PINTOS.
* Our original design did not have an inode table, however, I have added one for more realism.
* Code also used to contain sparse inodes to balence this strange design, these were removed, as inodes are no longer block sized.
* Sparse inodes are a nightmare to program for.
* Our original design had very easy support for openat(), since the filesystem could just grab the current user's CWD, then base relative queries from there. Microkernel cannot do this as easily because the relevant data lies across the process boundary.

```c
struct superblock {
  // 0xdeadbeef
  uint32_t magic;
  // location of free map on disk
  block_sector_t bitmap;
  // length of the bitmap in bytes.
  uint32_t bitmap_size;
  // The amount of inodes
  uint32_t num_inodes;
  // Location of inode table on disk.
  //  Inode table is a giant list of inodes.
  block_sector_t itable;
  //Used for finding "/"
  inum_t root_inode_num;
  uint8_t block_size;
};

#define I_FILETYPE (1 << 0)
#define I_FILE 0
#define I_DIR (1 << 0)
#define I_ALIVE (1 << 1)
struct inode_disk
{
    //Length in "number of blocks"
    uint32_t num_blocks;
    //NOTE: We might not need this back pointer. We only used it for implementing openat(".","..");
    block_sector_t parent; /* Our directory parent inode */
    block_sector_t directs[16];
    block_sector_t indirect; /* Single-level indirect */
    block_sector_t d_indirect; /* double-level indirect */
    block_sector_t t_indirect; /* triple-level indirect */
    inum_t inum; /* used for responding to fstat queries */

    //File length = num_blocks * BLOCK_SIZE + rem_size
    uint8_t rem_size;
    uint8_t type;
    //inode size is 128B, so we can fit 4 into a block.
    char wasted_space[42];
};

struct dir_entry
{
    //inode sector is zero if dir_entry is not in use
    block_sector_t inode_sector; /* Sector number of header. */
    //dir entry file names are stored as packed as possible on disk
    //name is null terminated, and cannot be greater than MAX_NAME_LEN
    //max_name_len is set so that dir_entry is smaller than a block.
    char name[];
};

/*
NOTE: This is the first design for dir entries that came to mind. There are a lot of unfortunate design tradeoffs here:
 * If we store directory entries packed, then it's harder to delete directory entries without introducing misalignment
 * If we store directory entries padded (char name[MAX_SIZE]) we are likely to waste a ton of space. It seems like most 
    filesystems are ok with this tradeoff though
 * We can also improve searching using hashing. I think you need to maintain directory entries as a tree for this to be efficient though.
*/


//These read and open requests come from some passed message from procnto.
//NULL indicates "/".
struct openstatat_response handle_openstatat(struct dir *at, const char *path) {
    char *splits = path_split(full_path);
    //From "at", we can recursively descend, each segment of "path" until we reach the base
    //NOTE: Path does not contain ".." because procnto strips these by necessity.
    //Once we find + open the file, we can easily stat by reading the inode fields.
    //We return this data as a response struct.
}

//"Local file ID" => "inum"
//We will send this data back as a message to procnto.
ssize_t handle_read(int inum, size_t pos, void *buf, size_t size) {
    struct file *f = NULL;
    struct dir *d = NULL;
    size_t s = 0;
    fs_try_open(inum,f,d);
    if (f != NULL) {
        return inode_read_at(f->inode,pos,buf,size);
    } else if (d != NULL) {
        /*
         * NOTE: we MUST give directory entries to procnto in a format PROCNTO
         *  can understand. Not our own internal format.
         *
         * If pos does not point to the beginning of a dir entry, this will fail.
         * Each dir entry returned is valid.
         */
        return inode_read_direntries(f->inode,pos,buf,size);
    }
    return -1;
}
NOTE: I did not include inode reading/writing code, since its not super complex, but it is very tedious. Essentially, for reading you can divide the read start location by the block size, to figure out which block to read from, then pointer-chase if needed to perform the read. For writing, you can apply the same process. What's unfortunate is that you need to handle each kind of indirection differently, which balloons the code size. Our project was even worse because of sparse inodes.
```

## Caching and Synchronization.

We can break synchronization down into two challenges, "inode data synchronization" and "File system synchronization".

## Inode data synchronization

### Files

Judging from what I have read online, the main synchronization requirement for UNIX is that almost every FS operation has to be atomic. On files, we can enforcing block-sized write restrictions, and only allowing one writer for each region at a time. The only caveat is when multiple users try to extend a file at the same time. We only allow one person to extend a file at once.

### Directories

Directories are much more tricky to handle, since these inodes contain FS-internal data. Depending on how exactly you lay out directory entries, you can have cascading writes. In the case of a tree structure, you need to have locks for each node, which could be more efficient. It's possible that the best solution would be to only let one user unlink / link directory entries at once. The reason directories are more complex is that you do not know *where* you are going to need to write to in advance. So an unlink() is effecively a scan + remove, while a link (create or hard link) is a "find the next empty slot, and put something in it".

NOTE: unlink() does not delete file data, only the directory entry. File data is garbage collected on file closure by all who have the file open.

## File System synchronization.

The challenge of FS synchronization is keeping the entire directory tree stable, including at the VFS layer.
* We can have people moving files or directories around while another user has them open. This is actually safe when working with file descriptors, but once paths are introduced, the challenge becomes unsolvable.
* Procnto/VFS needs to use paths when resolving where a file lies with respect to the filesystem boundary. This could introduce issues if not handled properly.
* Security: Time-of-check-time-of-use has been difficult to solve on unix. I think windows solved toctou by locking entire FS trees for exclusive access. This would have to be handled as a unix extension.

## Caching

Caching is an area of FS design I feel less comfortable with, however, the design I would choose for caching would be that the cache sits as a layer between block read/write requests, and the filesystem itself. 
* The cache acts as a hash map from block pointers to block data.
* I would guess that you only want to cache in units of 4096 (page size) for convenience and alignment when allocating.
* If the FS only wants 512 bytes, you load 4096 bytes into memory. Ex, a read request for block 0 would also load the data for blocks 1,2, and 3. 
* For low time complexity, you need the cache to act as a hash map. For cache replacement, you need a separate data structure.
    * I would implement this by maintaining a list of cache entries. At this point, you can pick a standard replacement algorithm such as Clock, LRU, etc to handle eviction.
* Evicition writes the block to disk.
* Major design tradeoff: How often do you evict to disk (without intervention)?
    * You can write ALL filesystem data to disk as soon as possible. Pro: Less likely to lose data, Con: Not able to buffer writes as well.
    * You can periodically flush writes from cache to disk. This is more efficient, but might lose data.
    * You can wait for eviction. I'm more suspicious of this design. I would worry that entries could remain in the cache for a long time because of reads, which would result in data loss.
* All filesystem data (data that lies OUTSIDE the inode) must be written immediately. All filesystems rely on atomic block-level writes to ensure consistency.
* flush must work correctly.
* You need to prevent the cache from being swapped to disk.

### Personal interest question:

Disks also have caching now. I'm quite curious what happens when you have nested caching systems. Do nested caches constructively interfere? destructively interfere? I know CPU's have multilevel caches, and when data gets evicted from L1 cache, it moves to L2 cache, etc. With a disk, the disk has no idea what caching system the FS is using, and vice-versa. I wonder if there's a lot of cache flushes for blocks which are NEVER! going to be read/written to again, that cause cache thrashing at the disk level.
