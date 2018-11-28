### 1. Mordern GCDs (WWDC2017)
Parallelism VS. Concurrency

* Parallelism:
    - Dispatch.concurrentPerform
```objc
dispatch_apply(DISPATCH_APPLY_AUTO, 1000, ^(size_t i) { … }  
//Not necessarily the number of cores, other task may occupying the core, automatic load balancing
```

* Concurrency:
    - Context switching - Instrument system trace  (system trace in depth in WWDC 16)
        - A higher priority thread needs CPU
        - A thread finishes its current work
        - Waiting to acquire a resource
        - Waiting for an asynchronous request to complete
    - Excessive context switching
        - Repeatedly waiting for exclusive access to contended resources
        
* `os_ufair_lock` may reduce lock-incurred context switch

|                 | Unfair         | Fair            |
|-----------------|----------------|-----------------|
| Available types | os_unfair_lock | pthread_mutex_t, NSLock, DispatchQueue.sync |
| Contended lock re-aquisition | Can steal the lock | Context switches to next waiter |
| Subject to waiter starvation | Yes | No |

* Lock ownership:

| Single Owner | No Owner | Mulitple Owners |
|---|---|---|
| Serial queues | dispatch_semaphore | Private concurrent queues |
| DispatchWorkItem.wait | dispatch_group | pthread_rwlock |
| os_unfair_lock | pthread_cond, NSCondition | |
| pthread_mutex, NSLock | Queue suspension | |

* Repeatedly switching between independent operations
    - Serial Dispatch Queue
        ```objc
        // 1 & 2 may be run on worker thread a, while 3 will be transferred to the calling thread and run there.
        dispatch_async(q, 1); 
        dispatch_async(q, 2); 
        dispatch_sync(q, 3)   
        ```
    - Dispatch Source
    - Target Queue Hierarchy
        - set target queue for a queue
    - Quality of Service
        - UI, IN, UT, BG
    
* One Queue hierarchy per subsystem, one target queue for Network, one target queue for DB
    - Good Granularity of Concurrency
        - Fixed number of serial queue hierarchies
        - Coarse workitem granularity between hierarchies
        - Finer workitem granularity inside a hierarchy
    
* Repeatedly bouncing an operation between threads
    - No mutation Past Activation
    - Protect the target queue hierarchy
        - Build queue hierarchy from bottom to top
        - Opt into “static queue hierarchy"
    


### 2. dispatch_queues discussions

> dispatch_get_global_queue() is in practice on of the worst thing that the dispatch API provides, because despite all the best efforts of the runtime, there aren't enough information at runtime about your operations/actors/... to understand what your intent is and optimize for it. Which means that the language should make sure that (1) the anti-pattern is not the default behavior and (2) the interface provides a way to give and propagate the hints (dependency relationships, ordering, execution context, priorities, ...) the runtime will potentially need upfront.
> 
> https://lists.swift.org/pipermail/swift-evolution/Week-of-Mon-20170828/039368.html
> 
> 
> In general suspending/resuming work is very difficult to handle for a runtime (implementation wise), has large memory costs, and breaks priority inversion avoidance. dispatch_suspend()/dispatch_resume() is one of the banes of my existence when it comes to dispatch API surface. It only makes sense for dispatch source "I don't want to receive these events anymore for a while" is a perfectly valid thing to say or do. But suspending a queue or work is ripping the carpet from under the feet of the OS as you just basically make all work that is depending on the suspended one invisible and impossible to reason about.
> 
> [...]
> 
> FWIW dispatch sources, and more importantly dispatch mach channels (which is the private interface that is used to implement XPC Connections) have a design that try really really really hard to not fall into any these pitfalls, are priority inheritance friendly, execute on *distributed* execution contexts, and have a state machine exposed through "on$Event" callbacks. We should benefit from the many years of experience that are condensed in these implementations when thinking about Actors and the primitives they provide.
> 
> https://lists.swift.org/pipermail/swift-evolution/Week-of-Mon-20170828/039373.html
> 
> 
> The place where GCD "fails" at is that if you target your individual serial queues to the global concurrent queues (a.k.a. root queues) which means "please pool, do your job", then yes it doesn't scale, because we take these individual serial queues as proxies for OS threads.
> 
> [...]
> 
> In GCD there's a very big difference between the one queue at the root of your graph (just above the thread pool) and any other that is within. The number that doesn't scale is the number of the former contexts, not the latter.
> 
> [...]
> 
> The real problem is that if you go async you need to be async all the way. Node.js and other similar projects have understood that a very long time ago. If you express dependencies between asynchronous execution context with a blocking relationship such as above, then you're just committing performance suicide. GCD handles this by adding more threads and overcommitting the system.
> 
> https://lists.swift.org/pipermail/swift-evolution/Week-of-Mon-20170828/039405.html
> 
> 
> The real world is far messier though. In practice, real world code blocks all of the time. In the case of GCD tasks, this is often tolerable for most apps, because their CPU usage is bursty and any accidental “thread explosion” that is created is super temporary. That being said, programs that create thousands of queues/closures that block on I/O will naturally get thousands of threads. GCD is efficient but not magic.
> 
> [...]
> 
> I think the real problem is that programmers cannot pretend that resources are infinite.
> 
> https://lists.swift.org/pipermail/swift-evolution/Week-of-Mon-20170828/039410.html
> 
> 
> NSOperation has several implementation issues, and using it to encapsulate asynchronous work means that you don't get the correct priorities (I don't say it cant' be fixed, I honnestly don't know, I just know from the mouth of the maintainer that NSOperation makes only guarantees if you do all your work from -[NSOperation main]).
> 
> https://lists.swift.org/pipermail/swift-evolution/Week-of-Mon-20170828/039415.html
> 
> 
> In dispatch, there are 2 kind of queues at the API level in that regard:
> - the global queues, which aren't queues like the other and really is just an abstraction on top of the thread pool
> - all the other queues that you can target on each other the way you want.
> 
> It is clear today that it was a mistake and that there should have been 3 kind of queues:
> - the global queues, which aren't real queues but represent which family of system attributes your execution context requires (mostly priorities), and we should have disallowed enqueuing raw work on it
> - the bottom queues (which GCD since last year tracks and call "bases" in the source code) that are known to the kernel when they have work enqueued
> - any other "inner" queue, which the kernel couldn't care less about
> 
> In dispatch, we regret every passing day that the difference between the 2nd and 3rd group of queues wasn't made clear in the API originally.
> 
> I like to call the 2nd category execution contexts, but I can also see why you want to pass them as Actors, it's probably more uniform (and GCD did the same by presenting them both as queues). Such top-level "Actors" should be few, because if they all become active at once, they will need as many threads in your process, and this is not a resource that scales. This is why it is important to distinguish them. And like we're discussing they usually also wrap some kind of shared mutable state, resource, or similar, which inner actors probably won't do.
> 
> https://lists.swift.org/pipermail/swift-evolution/Week-of-Mon-20170828/039420.html
> 
> 
> Private concurrent queues are not a success in dispatch and cause several issues, these queues are second class citizens in GCD in terms of feature they support, and building something with concurrency *within* is hard. I would keep it as "that's where we'll go some day" but not try to attempt it until we've build the simpler (or rather less hard) purely serial case first.
> 
> [...]
> 
> This is why our belief (in my larger K(ernel) & R(untime) team) is that instead of trying to paper over the fact that there's a real OS running your stuff, we need to embrace it and explain to people that everywhere in a traditional POSIX world they would have used a real pthread_create()d thread to perform the work of a given subsystem, they create one such category #2 bottom queue that represents this thread (and you make this subsystem an Actor), and that the same way in POSIX you can't quite expect a process to have thousands of threads, you can't really have thousands of such top level actors either. It doesn't mean that there can't be some anonymous pool to run stuff on it for the stuff that is less your raison d'être, it just means that the things your app that are really what it was built to do should be organized in a resource aware way to take maximum advantage of the system. Throwing a million actors to the system without more upfront organization and initial thinking by the developers is not optimizable.
> 
> https://lists.swift.org/pipermail/swift-evolution/Week-of-Mon-20170828/039429.html
> 
> 
> 
> Some Frameworks have a strong need for a serial context, some don't
> 
> It completely depends on the framework. If your framework is, say, a networking subsystem which is very asynchronous by nature for a long time, then yes, having the framework setup a #2 kind of guy inside it and have callbacks from/to this isolated context is just fine (and incidentally what your networking stack does).
> 
> However for some frameworks it makes very little sense to do this, they're better served using the "location" provided by their client and have some internal synchronization (locks) for the shared state they have. Too much framework code today creates their own #2 queue (if not queue*s*) all the time out of fear to be "blocked" by the client, but this leads to terrible performance.
> 
> [ disclaimer I don't know that Security.framework works this way or not, this is an hypothetical ]
> 
> For example, if you're using Security.framework stuff (that requires some state such as say your current security ephemeral keys and what not), using a private context instead of using the callers is really terribly bad because it causes tons of context-switches: such a framework should really *not* use a context itself, but a traditional lock to protect global state. The reason here is that the global state is really just a few keys and mutable contexts, but the big part of the work is the CPU time to (de)cipher, and you really want to parallelize as much as you can here, the shared state is not reason enough to hop.
> 
> It is tempting to say that we could still use a private queue to hop through to get the shared state and back to the caller, that'd be great if the caller would tail-call into the async to the Security framework and allow for the runtime to do a lightweight switch to the other queue, and then back. The problem is that real life code never does that: it will rarely tail call into the async (though with Swift async/await it would) but more importantly there's other stuff on the caller's context, so the OS will want to continue executing that, and then you will inevitably ask for a thread to drain that Security.framework async.
> 
> In our experience, the runtime can never optimize this Security async pattern by never using an extra thread for the Security work.
> 
> 
> Top level contexts are a fundamental part of App (process) design
> 
> It is actually way better for the app developer to decide what the subsystems of the app are, and create well known #2 context for these. In our WWDC Talk we took the hypothetical example of News.app, that fetches stuff from RSS feeds, has a database to know what to fetch and what you read, the UI thread, and some networking parts to interact with the internet.
> 
> Such an app should upfront create 3 "#2" guys:
> - the main thread for UI interactions (this one is made for you obviously)
> - the networking handling context
> - the database handling context
> 
> The flow of most of the app is: UI triggers action, which asks the database subsystem (brain) what to do, which possibly issues networking requests.
> When a networking request is finished and that the assets have been reassembled on the network handling queue, it passes them back to the database/brain to decide how to redraw the UI, and issues the command to update the UI back to the UI.
> 
> 
> At the OS layer we believe strongly that these 3 places should be made upfront and have strong identities. And it's not an advanced need, it should be made easy. The Advanced need is to have lots of these, and have subsystems that share state that use several of these contexts.
> 
> 
> For everything else, I agree this hypothetical News.app can use an anonymous pools or reuse any of the top-level context it created, until it creates a scalability problem, in which case by [stress] testing the app, you can figure out which new subsystem needs to emerge. For example, maybe in a later version News.app wants beautiful articles and needs to precompute a bunch of things at the time the article is fetched, and that starts to take enough CPU that doing this on the networking context doesn't scale anymore. Then you just create a new top-level "Article Massaging" context, and migrate some of the workload there.
> 
> 
> Why this manual partitionning?
> 
> It is our experience that the runtime cannot figure these partitions out by itself. and it's not only us, like I said earlier, Go can't either.
> 
> The runtime can't possibly know about locking domains, what your code may or may not hit (I mean it's equivalent to the termination problem so of course we can't guess it), or just data affinity which on asymmetric platforms can have a significant impact on your speed (NUMA machines, some big.LITTLE stuff, ...).
> 
> The default anonymous pool is fine for best effort work, no doubt we need to make it good, but it will never beat carefully partitioned subsystems.
> 
> https://lists.swift.org/pipermail/swift-evolution/Week-of-Mon-20170904/039461.html
