### Anti-debugging
- 使用ptrace
```objc
#if !defined(PT_DENY_ATTACH)
#define PT_DENY_ATTACH 31
#endif

typedef int (*ptrace_ptr)(int request, pid_t pid, caddr_t addr, void *data);

- (void)anti_debug {
    void *handle = dlopen(NULL, RTLD_GLOBAL|RTLD_NOW);
    ptrace_ptr p_ptr = dlsym(handle, "ptrace");
    p_ptr(PT_DENY_ATTACH, 0, 0, 0);
    dlclose(handle);
}
```

- 使用sysctl
```objc
#include <assert.h>
#include <stdbool.h>
#include <sys/types.h>
#include <unistd.h>
#include <sys/sysctl.h>

static bool AmIBeingDebugged(void)
// Returns true if the current process is being debugged (either
// running under the debugger or has a debugger attached post facto).
{
  int                junk;
  int                mib[4];
  struct kinfo_proc  info;
  size_t              size;
  
  // Initialize the flags so that, if sysctl fails for some bizarre
  // reason, we get a predictable result.
  
  info.kp_proc.p_flag = 0;
  
  // Initialize mib, which tells sysctl the info we want, in this case
  // we're looking for information about a specific process ID.
  
  mib[0] = CTL_KERN;
  mib[1] = KERN_PROC;
  mib[2] = KERN_PROC_PID;
  mib[3] = getpid();
  
  // Call sysctl.
  
  size = sizeof(info);
  junk = sysctl(mib, sizeof(mib) / sizeof(*mib), &info, &size, NULL, 0);
  assert(junk == 0);
  
  // We're being debugged if the P_TRACED flag is set.
  
  return ( (info.kp_proc.p_flag & P_TRACED) != 0 );
}
```

### How debuggers work
ptrace API：
- 可以使子进程指定允许父进程trace自己
- 可以在父进程中查看、修改子进程寄存器、内存等内容，或者单步执行指令

breakpoint是用int 3软中断实现，在断点的代码位置将原位置上的一个字节的代码用0xCC替换（0xCC即为int 3），子进程cpu执行到0xCC处即停止，并发送sigtrap，父进程收到此消息即可进行相应的deug操作


参考资料：
> https://eli.thegreenplace.net/2011/01/23/how-debuggers-work-part-1/  
> https://eli.thegreenplace.net/2011/01/27/how-debuggers-work-part-2-breakpoints/  
> https://eli.thegreenplace.net/2011/02/07/how-debuggers-work-part-3-debugging-information
