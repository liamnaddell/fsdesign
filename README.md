# Design

## TODO

* Count avg directory entries
* count average filename length
* what devices are we targeting?
* Union filesystems
* Pretty pictures!
* https://www.qnx.com/developers/docs/8.0/com.qnx.doc.neutrino.getting_started/topic/s1_resmgr_Finding_server.html
* File systems as shared libraries
* Thomasf https://www.qnx.com/developers/docs/8.0/com.qnx.doc.neutrino.sys_arch/topic/resource_FILESYSresmgr.html

## Understanding the task

* `find . -name <name>`
* Maybe less, but need to handle maximum scale of ~1000 entries 
* Readdir based

### An implementation of our "program"

```c
#include <dirent.h>

...
int find(const char *filename, const char *dirname) {
    DIR *d = opendir(dirname);
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
3. What's `de`'s lifetime? The manual doesn't say!

## How might we implement this in libc?

I think it would be best to implement readdir and opendir as special cases of read(), and open(). As such, when you read(), or open() a directory, you should be able to read() directory entries.

One nice note of our "find" implementation is that we do NOT have to call `stat` on the file/directory. It is important to make sure to be able to handle the common case without a `stat` call.

```c

struct dirent {
    char d_type;
    /* good idea, shamelessly stolen from glibc */
    unsigned short d_reclen;
    /* name */
    const char *d_name;
};

#define BSIZE 512
typedef struct DIR {
    int fd;
    //buf for readdir
    //DESIGN NOTE: I did this to make readdir (correct word: reentrant?)
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
    //Token for VFS.
    int global_file_id;
};

//TODO:
int open(const char *pathname, int flags) {
    //reserved,pid,chid,index,flags
    //TODO: How do you know pid, channel? Do we just ask procnto?
    coid = ConnectAttach (0, VFS_PROCESS, VFS_CHANNEL, 0, 0);

    struct file f;

    int retval = MsgSend(coid,"Please open and stat {pathname} for me",msg_len,&f,sizeof(struct file));
    //handle error;

    int fd = libc_add_file(&f);
    return fd;
}

ssize_t read(int fd, void *buf, size_t count) {
    //a file descriptor is an index into a table of open files!
    struct file *fi = libc_get_file(fd);
    //...error handling...

    //NOTE: A real message would have message data. I'd imagine we'd want to handle stat,
    //   etc by specifying message type
    long a = MsgSend(fi->coid,"Please get file data" + fi->global_file_id,msg_len,buf,count);

    //TODO: ???
    return a;
}
```

In summary, readdir() is a wrapper around read(), which bulk receives directory entries from VFS. We improve efficiency by handling all the information in large chunks, and having the readdir() function only make syscalls when absolutely necessary.

##What happens at the VFS layer?

The VFS operates by dispatching FS messages to the appropriate drivers. In our case, we need to take read requests (on files and directories), and forward them, knowing also that QNX operates using unioned filesystems.

How might a VFS work? Well the inputs to a VFS are a set of mounts and mount points, and the VFS will use this information to dispatch reads/writes.

Where does the VFS live in the system? A: I believe it is in procnto aka /sbin/init.

As such, we need a mechanism for registering mounts.

When a user mounts a filesystem, we need to determine which filesystem driver is appropriate. The superblock should (e.g. will) make this clear.

```c
//File system driver makes itself known to procnto:
...
int register_driver() {
    long a = MsgSend(coid_with_procnto,"I am a filesystem driver",msg_len,NULL,0);
    return a;
}

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
}

//Handle incoming open()'s Assuming this is a message sent to us by someone's libc.
int handle_open(const char *full_path) {
    struct open_query_reply r;
    for (auto mount : mounttable.get_mounts()) {
        if (mount.path() prefixes full_path) {
            long a = MsgSend(mount.coid,"Open file {full_path - mount.path()}",n,&r,sizeof(open_query_reply));

            if (r.success) {
                return r.file_id;
            }
        }
    }
}

//Assuming we are receiving a read request for global_file_id
//would be sent by MsgSend
ssize_t handle_read(int global_file_id, void *buf, size_t size) {
    //Procnto will manage a global list of open files.
    //Global file id's are managed by procnto
    //local_file_id's are assigned by the filesystem.
    //procnto maintains a mapping from global_file_id -> local_file_id
    struct qnx_file *f = procnto_get_file(global_file_id);
    long a = MsgSend(f->driver->coid,"Please get this file data "+local_file_id,buf,size);
    return a;
}
```

##What happens inside the filesystem ResMgr?

* The filesystem resmgr will be structured like a normal user program. It will receive incoming FS messages from procnto, and dispatch those queries, using its exclusive access to a block device.
* To reiterate, a filesystem ResMgr manages a block device.

Q: How do FS drivers assert EXCLUSIVE access to a block device.


```c
//These read and open requests come from some passed message from procnto.
int handle_open(const char *full_path) {
    
}
ssize_t handle_read(int global_file_id, void *buf, size_t size) {
}
```
