# Design

## TODO

* Count avg directory entries
* count average filename length
* what devices are we targeting?

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
    long a = MsgSend(fi->coid,"Please get file data",msg_len,buf,count);

    //TODO: ???
    return a;
}
```

In summary, readdir() is a wrapper around read(), which bulk receives directory entries from VFS. We improve efficiency by handling all the information in large chunks, and having the readdir() function only make syscalls when absolutely necessary.

##What happens at the VFS layer?
##What happens inside the filesystem ResMgr?
