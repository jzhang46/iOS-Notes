### 1. dispatch_semaphore VS. pthread_mutex_t
* dispatch_semaphore is an antipattern for synchronization
* dispatch_semaphores are prone to priority inversions, very much like OSSpinLock
    * It doesn't record the holder of the lock, so the kernel doesn't know which threads to donate priority to.
    
There are two things for synchronization: 1) avoid spinning, 2) tracking the lock owner. Spinlock is 0/2, semaphore 1/2.

If some threads are waiting for semaphore, but the semaphore's holding thread has lower priority than other threads (queues), it may hang.

The prio-donation/QoS is intended to improve front app responsiveness.

> 
> Question: The locks that don't record the thread holding the mutex are very common eg on Linux, Android, Win and WebKit.
> 
> Answer: 
>    1) Linux support for perthread priority is never used
>    2) All webkit threads run at the same priority, because they also own their thread pool
>     

原贴：https://twitter.com/cocoawithlove/status/739104279202406400


### 2. os_unfair_lock

Run on my latest gen rMBP, max specs.
* os_unfair_lock is new in iOS 10 and Sierra and is the fastest option.

* OSSpinLock and dispatch semaphores don't donate priorities and should not be used.

* NSLock is basically pthread_mutex + objc_msgSend

* queues can be slow

* @synchronized needs to lock twice because it works with an arbitrary object that is not itself a lock object. (thanks, Greg!)
    * testUnfairLock (4.014 seconds)
    * testSpinLock (4.064 seconds)
    * testDispatchSemaphore (46.611 seconds)
    * testPthreadMutex (64.438 seconds)
    * testNSLock (68.508 seconds)
    * testQueue (67.629 seconds)
    * testSynchronized (70.172 seconds)
At PSPDFKit we now use a wrapper for os_unfair_lock that falls back to pthread_mutex on iOS 9.

TODO: Rewrite in ObjC++ to make sure Swift doesn't change any of the results here.

TODO: Measure std::mutex, std::recursive_mutex and std::lock_guard

原贴：
https://gist.github.com/steipete/36350a8a60693d440954b95ea6cbbafc

### 3. Spinlock Priority inversion
* Under QOS, low priority tasks may get starved when there’s always high priority tasks running.
* If low priority holds a spinlock and yields, the high priority task will keep spinning when trying to acquire the lock, and the low priority task will not get a chance to release the lock.

### 4. ThreadSanitizer原理
对于每个8 byte的内存块，维护几个数据：

**加锁时：**
1. 将锁中记录的其他线程访问此内存块的时间戳更新到当前线程的thread local中

**访问内存时：**
1. thread local中当前线程的time stamp += 1
2. 用thread local中记录的其他线程的time stamp跟 shadow sate中记录的其他线程的time stamp进行比较，如果发现比shadow state中的旧，则说明有data race，报warning
3. 更新shadow state中当前线程的time stamp
4. 更新当前所占用的锁中对应于此内存块的当前线程访问时间戳

**注：**
* 每次内存访问，当前线程的time stamp都会 +1
* shadow state为全局数据
* 如果没加锁就访问了内存，则当前线程的thread local中其他线程的time stamp就不会被同步过来，就会导致第2.步中比较时thread local里的time stamp值比shadow state旧

参考来源：WWDC 2016 视频

### 5. How do I get put into the real time scheduling class?

Listing 1  The following code will move a pthread to the real time scheduling class

```objc
#include <mach/mach.h>
#include <mach/mach_time.h>
#include <pthread.h>
 
void move_pthread_to_realtime_scheduling_class(pthread_t pthread)
{
    mach_timebase_info_data_t timebase_info;
    mach_timebase_info(&timebase_info);
 
    const uint64_t NANOS_PER_MSEC = 1000000ULL;
    double clock2abs = ((double)timebase_info.denom / (double)timebase_info.numer) * NANOS_PER_MSEC;
 
    thread_time_constraint_policy_data_t policy;
    policy.period      = 0;
policy.computation = (uint32_t)(5 * clock2abs); // 5 ms of work
    policy.constraint  = (uint32_t)(10 * clock2abs);
    policy.preemptible = FALSE;
 
    int kr = thread_policy_set(pthread_mach_thread_np(pthread_self()),
                   THREAD_TIME_CONSTRAINT_POLICY,
                   (thread_policy_t)&policy,
                   THREAD_TIME_CONSTRAINT_POLICY_COUNT);
    if (kr != KERN_SUCCESS) {
        mach_error("thread_policy_set:", kr);
        exit(1);
    }
}
```

The period, computation, constraint, and preemptible fields do have an effect, and more can be learned about them at:  
Using the Mach Thread API to Influence Scheduling

### 6. Which timing API(s) should I use?

As mentioned above, all the timer methods end up in the same place inside the kernel. However, some of them are more efficient than the others. At the time this note was written, mach_wait_until() is the api we would recommend using. It has the lowest overhead of all the timing API that we measured. However, if you have specific needs that aren't met by mach_wait_until(), for example you need to wait on a condvar, then feel free to use the appropriate timer API.

Listing 2  This example code demonstrates using mach_wait_until() to wait exactly 10 seconds.
```objc
#include <mach/mach.h>
#include <mach/mach_time.h>
 
static const uint64_t NANOS_PER_USEC = 1000ULL;
static const uint64_t NANOS_PER_MILLISEC = 1000ULL * NANOS_PER_USEC;
static const uint64_t NANOS_PER_SEC = 1000ULL * NANOS_PER_MILLISEC;
 
static mach_timebase_info_data_t timebase_info;
 
static uint64_t abs_to_nanos(uint64_t abs) {
    return abs * timebase_info.numer  / timebase_info.denom;
}
 
static uint64_t nanos_to_abs(uint64_t nanos) {
    return nanos * timebase_info.denom / timebase_info.numer;
}
 
void example_mach_wait_until(int argc, const char * argv[])
{
    mach_timebase_info(&timebase_info);
    uint64_t time_to_wait = nanos_to_abs(10ULL * NANOS_PER_SEC);
    uint64_t now = mach_absolute_time();
    mach_wait_until(now + time_to_wait);
}
```

原贴地址：
https://developer.apple.com/library/content/technotes/tn2169/_index.html

