### Personal interest question:

Disks also have caching now. I'm quite curious what happens when you have nested caching systems. Do nested caches constructively interfere? destructively interfere? I know CPU's have multilevel caches, and when data gets evicted from L1 cache, it moves to L2 cache, etc. With a disk, the disk has no idea what caching system the FS is using, and vice-versa. I wonder if there's a lot of cache flushes for blocks which are NEVER! going to be read/written to again, that cause cache thrashing at the disk level.
