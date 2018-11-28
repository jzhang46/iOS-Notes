### Crash Log - bad address
* A crash with a bad address that looks like a valid heap address rotated right 4 bits is a sign of use after free. That 4 bit rotate is an encoding of the malloc free list.

* Not mentioned in WWDC 2018 session 414: 
    * Another malloc free list signature is a bad address 0x…bead; 0x…bec0 and 0x…bec8 are common in objc_msgSend due to masking and offsets. That value is a remnant of the malloc free list encoding by he nanozone allocator on some platforms.
    * 


### EXC_GUARD exception
系统会在使用某些资源时给资源加上guard，系统访问这些guarded资源时会使用特殊的guarded api，如果其他模块访问了这些资源，就会触发EXC_GUARD exception.

> Guarded Resource Violation [EXC_GUARD]
> 
> The process violated a guarded resource protection. System libraries may mark certain file descriptors as guarded, after which normal operations on those descriptors will trigger an EXC_GUARD exception (when it wants to operate on these file descriptors, the system uses special 'guarded' private APIs). This helps you quickly track down issues such as closing a file descriptor that had been opened by a system library. For example, if an app closes the file descriptor used to access the SQLite file backing a Core Data store, Core Data would then crash mysteriously much later on. The guard exception gets these problems noticed sooner, and thus makes them easier to debug.
> Crash reports from newer versions of iOS include human-readable details about the operation that caused the EXC_GUARD exception in the Exception Subtype and Exception Message fields. In crash reports from macOS or older versions of iOS, this information is encoded into the first Exception Code as a bitfield which breaks down as follows:
> * [63:61] - Guard Type: The type of the guarded resource. A value of 0x2 indicates the resource is a file descriptor.
> * [60:32] - Flavor: The conditions under which the violation was triggered.
> If the first (1 << 0) bit is set, the process attempted to invoke close() on a guarded file descriptor.
> 
> If the second (1 << 1) bit is set, the process attempted to invoke dup(), dup2(), or fcntl() with the F_DUPFD or F_DUPFD_CLOEXEC commands on a guarded file descriptor.
> 
> If the third (1 << 2) bit is set, the process attempted to send a guarded file descriptor via a socket.
> 
> If the fifth (1 << 4) bit is set, the process attempted to write to a guarded file descriptor.
> * [31:0] - File Descriptor: The guarded file descriptor that the process attempted to modify.
