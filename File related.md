### 1. 禁用 Unified Buffer Cache (UBC)

```
//禁用 Unified Buffer Cache (UBC), 需要保证文件之前不在cache中，否则还是会用cache
fcntl(fd, F_NOCACHE, 1) 
```

`F_GLOBAL_NOCACHE`是直接将所有对此文件的访问都置为NO cache，但好像现在行为是跟`F_NOCACHE`是一样的。

原贴地址：
https://forums.developer.apple.com/thread/25464

苹果员工评论：
> 
> eskimoNov 9, 2015 1:02 AM(in response to Ken Thomases)
>
> > I belive that F_NOCACHE prevents the file's data from being added to the cache if it's not already there.  However, if it's already in the cache, the cache will be used for reading.
>
> That’s correct.  If things didn’t behave that way then the file system would be inconsistent depending on whether you set F_NOCACHE or not, which would be a bit of a nightmare (especially when you consider memory mappings).
> Some things to keep in mind while testing:
> * Set F_NOCACHE on your writes as well as your reads.
> * For each round of testing, deleting the file before you start.  That guarantees that no remnants of that file will remaining in the cache.
> * F_NOCACHE has implementation restrictions that can cause it to end up using the cache.  To increase the chances of that not happening, do the following:
> * use a page aligned buffer (valloc is your friend)
> * make your I/O length a multiple of the page size
> * make your I/O offset (that is, the offset into the file) a multiple of the page size
> Finally, be aware that F_NOCACHE is a hint, not an absolute requirement, and there are circumstances under which it will use the cache, and those circumstances can change from release to release.
> 
> > If one wanted to write a form of IO benchmarking program, where you are trying to measure the hardware-centric not-host-buffered IO performance, what system calls would be used?
>
> F_NOCACHE works fine for that (or it did the last time I tested it, which was quite some time ago).  There are two things to watch out for:
> * You have to make sure that the file you’re interacting doesn’t get into the cache.  The easiest way to do that is to create your own test file and write to it with non-cached I/O.  You can test whether this is working as expected using mincore.
> * You need to watch out for the file being discontiguous.  F_PREALLOCATE can help with that

### 2. Temp file creation 生成临时文件
```objc
NSString *templateStr = [NSTemporaryDirectory() stringByAppendingPathComponent: @"live_crash_report.XXXXXX"];
char *path = strdup([templateStr fileSystemRepresentation]);
    
int fd = mkstemp(path); //取随机数填充XXXXXX，生成临时文件名，并以mode 0600 打开
if (fd < 0) {
        plcrash_populate_posix_error(outError, errno, NSLocalizedString(@"Failed to create temporary path", @"Error opening temporary output path"));
        free(path);

        return nil;
}
```
