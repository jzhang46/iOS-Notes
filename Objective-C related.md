### @def

* @defs 用于返回一个 Objective-C 类的 struct 结构，这个 struct 与原 Objective-C 类具有相同的内存布局。
* Getting the isa pointer of an NSObject base instance is relatively easy, if we use the useful @defsconstruct:
```objc
static inline Class _isa( NSObject *obj) {
    return( ((struct { @defs( NSObject) } *) obj)->isa);
}
```

### NSSelectorFromString(@"aaa")
* Internally calls `sel_registerName("aaa")`;

### system trace Point of Interest (WWDC16)
* Point of Interest:
    - kdebug_signpost
    - kdebug_signpost_start(code, arg1, arg2, arg3, arg4)
    - kdebug_signpost_end(code, arg1, arg2, arg3, arg4)
        - code是一个id，用于区分不同Point of interest
        - ar1, arg2, arg3为预留字段
        - arg4用于区分颜色
            - 0=Blue; 1=Green; 2=Purple; 3=Orange; 4=Red
```objc
#import <sys/kdebug_signpost.h>
- (void)func {
    kdebug_signpost_start(10, 0, 0, 0, 0);
    //code
    kdebug_signpost_end(10, 0, 0, 0, 0);
}
```

### __kindof关键字
objc泛型里如果不用__kindof指定泛型参数，add subclass object可以，但取值赋给子类变量时就会有warning

加上__kindof以后就好了。

```objc
NSMutableArray<UIView *> *subviews = [[NSMutableArray alloc] init];

[subviews addObject:[[UIView alloc] init]]; // Works
[subviews addObject:[[UIImageView alloc] init]]; // Also works

UIView *sameView = subviews[0]; // Works
UIImageView *sameImageView = subviews[1]; // Incompatible pointer types initializing 'UIImageView *' with an expression of type 'UIView *'

NSLog(@"%@", NSStringFromClass([sameView class])); // UIView
NSLog(@"%@", NSStringFromClass([sameImageView class])); // UIImageView
```

Now, this produces a compile time warning, but does not crash at runtime. The key difference between this and the same example where the array's generic type is marked as __kindof, is that the compiler won't complain if you try to access ones of its elements, and store the result in a variable who's type is UIView or one of its subclasses.

```objc
NSMutableArray<__kindof UIView *> *subviews = [[NSMutableArray alloc] init];

[subviews addObject:[[UIView alloc] init]]; // Works
[subviews addObject:[[UIImageView alloc] init]]; // Also works

UIView *sameView = subviews[0]; // No problem
UIImageView *sameImageView = subviews[1]; // No complaints now!

NSLog(@"%@", NSStringFromClass([sameView class])); // UIView
NSLog(@"%@", NSStringFromClass([sameImageView class])); // UIImageView
```
