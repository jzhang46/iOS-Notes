### Environment variables handled in +[NSObject load]:
* NSDebugEnabled
* NSZombieEnabled
* NSDeallocateZombies
* NSDisableAutoreleasePoolCache
* NSDOLoggingEnabled
* NSUnbufferedIO 

### Log NSURLSession errors:  
* CFNETWORK_DIAGNOSTICS=1

### Activity tracing
OS_ACTIVITY_MODE=<Disable>|<stream>

### Malloc environment variables
```
MallocStackLogging
If set, malloc remembers the function call stack at the time of each allocation. 

MallocStackLoggingNoCompact
This option is similar to MallocStackLogging but makes sure that all allocations are logged, no matter how small or how short lived the buffer may be.

MallocScribble
If set, free sets each byte of every released block to the value 0x55.

MallocPreScribble
If set, malloc sets each byte of a newly allocated block to the value 0xAA. This increases the likelihood that a program making assumptions about freshly allocated memory fails. 

MallocGuardEdges
If set, malloc adds guard pages before and after large allocations.

MallocDoNotProtectPrelude  
Fine-grain control over the behavior of MallocGuardEdges: If set, malloc does not place a guard page at the head of each large block allocation.

MallocDoNotProtectPostlude
Fine-grain control over the behavior of MallocGuardEdges: If set, malloc does not place a guard page at the tail of each large block allocation.

MallocCheckHeapStart
Set this variable to the number of allocations before malloc will begin validating the heap. If not set, malloc does not validate the heap.

MallocCheckHeapEach
Set this variable to the number of allocations before malloc should validate the heap. If not set, malloc does not validate the heap.

```
