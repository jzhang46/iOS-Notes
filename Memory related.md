* Apple A10 cpu cache size:
    - L1 cache: 64KB per-core, one for data and  one for instructions
    - L2 cache: 3MB, shared by both CPU cores
    - L3 cache: 4MB

* MAC上内存相关命令
```sh
//得到当前heap上所有对象的类型、大小及其相应地址等
$heap -showSizes <pid>                //print all object type and sizes
$heap -address=all <pid>               //print all objects in heap and their addresses

//得到进程生命周期内所有的内存分配事件、调用栈
$malloc_history -allEvents <pid>     //print all ALLOC/FREE events, including call stack
$malloc_history -callTree -collapseRecursion <pid>     //print memory alloc call trees
$malloc_history <pid> <address>   //print the ALLOC/FREE event of the <address>, including call stack

//得到当前内存中所有object申请时的call stack
$vmmap -v -stacks <pid>               //print all call stacks of all objects in the heap
```

