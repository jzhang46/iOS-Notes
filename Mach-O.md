### 1. Dynamic linking (ELF VS. Mach-O)
* http://timetobleed.com/dynamic-linking-elf-vs-mach-o/

### 2. App launching
```sh
$ xcrun dyldinfo -rebase -bind -lazy_bind <YOUR_APP> //show what dyld need to do to fix up during launching
```

### 3. Show dylib dependency
```sh
$ xcrun dyldinfo -dylibs <YOUR_APP> 
```

### 4. Other cmds
```sh

//各个段、区的大小
$xcrun zie -x -l -m a.out
Segment __PAGEZERO: 0x100000000 (vmaddr 0x0 fileoff 0)
Segment __TEXT: 0x1000 (vmaddr 0x100000000 fileoff 0)
    Section __text: 0x37 (addr 0x100000f30 offset 3888)
    Section __stubs: 0x6 (addr 0x100000f68 offset 3944)
    Section __stub_helper: 0x1a (addr 0x100000f70 offset 3952)
    Section __cstring: 0xe (addr 0x100000f8a offset 3978)
    Section __unwind_info: 0x48 (addr 0x100000f98 offset 3992)
    Section __eh_frame: 0x18 (addr 0x100000fe0 offset 4064)
    total 0xc5
Segment __DATA: 0x1000 (vmaddr 0x100001000 fileoff 4096)
    Section __nl_symbol_ptr: 0x10 (addr 0x100001000 offset 4096)
    Section __la_symbol_ptr: 0x8 (addr 0x100001010 offset 4112)
    total 0x18
Segment __LINKEDIT: 0x1000 (vmaddr 0x100002000 fileoff 8192)
total 0x100003000

//某个段的具体内容
$xcrun otool -s __TEXT __text a.out 
a.out:
(__TEXT,__text) section
0000000100000f30 55 48 89 e5 48 83 ec 20 48 8d 05 4b 00 00 00 c7 
0000000100000f40 45 fc 00 00 00 00 89 7d f8 48 89 75 f0 48 89 c7 
0000000100000f50 b0 00 e8 11 00 00 00 b9 00 00 00 00 89 45 ec 89 
0000000100000f60 c8 48 83 c4 20 5d c3

//代码段的汇编
$xcrun otool -v -t a.out
a.out:
(__TEXT,__text) section
_main:
0000000100000f30    pushq   %rbp
0000000100000f31    movq    %rsp, %rbp
0000000100000f34    subq    $0x20, %rsp
0000000100000f38    leaq    0x4b(%rip), %rax
0000000100000f3f    movl    $0x0, 0xfffffffffffffffc(%rbp)
0000000100000f46    movl    %edi, 0xfffffffffffffff8(%rbp)
0000000100000f49    movq    %rsi, 0xfffffffffffffff0(%rbp)
0000000100000f4d    movq    %rax, %rdi
0000000100000f50    movb    $0x0, %al
0000000100000f52    callq   0x100000f68
0000000100000f57    movl    $0x0, %ecx
0000000100000f5c    movl    %eax, 0xffffffffffffffec(%rbp)
0000000100000f5f    movl    %ecx, %eax
0000000100000f61    addq    $0x20, %rsp
0000000100000f65    popq    %rbp
0000000100000f66    ret

//其他区的内容
$xcrun otool -v -s __TEXT __cstring a.out
a.out:
Contents of (__TEXT,__cstring) section
0x0000000100000f8a  Hello World!\n

//其他区内容
$xcrun otool -v -s __TEXT __eh_frame a.out 
a.out:
Contents of (__TEXT,__eh_frame) section
0000000100000fe0    14 00 00 00 00 00 00 00 01 7a 52 00 01 78 10 01 
0000000100000ff0    10 0c 07 08 90 01 00 00

//显示依赖的动态库
$xcrun -otool -L a.out
a.out:
    /System/Library/Frameworks/Foundation.framework/Versions/C/Foundation (compatibility version 300.0.0, current version 1056.0.0)
    /usr/lib/libSystem.B.dylib (compatibility version 1.0.0, current version 1197.1.1)
    /System/Library/Frameworks/CoreFoundation.framework/Versions/A/CoreFoundation (compatibility version 150.0.0, current version 855.11.0)
    /usr/lib/libobjc.A.dylib (compatibility version 1.0.0, current version 228.0.0)

//显示symbol table
$xcrun nm -nm a.out 
                 (undefined) external _NSFullUserName (from Foundation)
                 (undefined) external _NSLog (from Foundation)
                 (undefined) external _OBJC_CLASS_$_NSObject (from CoreFoundation)
                 (undefined) external _OBJC_METACLASS_$_NSObject (from CoreFoundation)
                 (undefined) external ___CFConstantStringClassReference (from CoreFoundation)
                 (undefined) external __objc_empty_cache (from libobjc)
                 (undefined) external __objc_empty_vtable (from libobjc)
                 (undefined) external _objc_autoreleasePoolPop (from libobjc)
                 (undefined) external _objc_autoreleasePoolPush (from libobjc)
                 (undefined) external _objc_msgSend (from libobjc)
                 (undefined) external _objc_msgSend_fixup (from libobjc)
                 (undefined) external dyld_stub_binder (from libSystem)
0000000100000000 (__TEXT,__text) [referenced dynamically] external __mh_execute_header
0000000100000e50 (__TEXT,__text) external _main
0000000100000ed0 (__TEXT,__text) non-external -[Foo run]
0000000100001128 (__DATA,__objc_data) external _OBJC_METACLASS_$_Foo
0000000100001150 (__DATA,__objc_data) external _OBJC_CLASS_$_Foo

```
