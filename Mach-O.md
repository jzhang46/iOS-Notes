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
