
### 1. Xcode commands

```sh
//列出所有模拟器
$ xcrun simctl list  

//找二进制文件的symbol, man symbols可以看help，/Applications/Xcode.app/Contents/Developer/usr/bin/symbols
$ xcrun symbols      

//显示当前xcode的路径
$xcode-select -print-path

//转换plist
$xcrun plutil -convert <xml1>|<binary1> xxx.plist

//列出所有的scheme
$xcodebuild -list <project_file>

//将buildsetting导出到.xcconfig文件中
$xcodebuild -scheme <scheme_name> -showbuildSettings >> mynew.xcconfig

//二进制文件中各个段的大小
$xcrun size -x -l -m <二进制文件>

//只输出buuild所用命令，不真正build
// -n 或 -dry-run
$xcodebuild -n -workspace <YOUR_WORKSPACE>.xcworkspace -scheme <YOUR_SCHEME>

//显示ios的sdk路径
$xcrun --sdk iphoneos --show-sdk-path
例：/Applications/Xcode.app/Contents/Developer/Platforms/iPhoneOS.platform/Developer/SDKs/iPhoneOS12.1.sdk
```

### 2. System commands

```sh

//显示导入symbol
$ nm -u  

//显示所有导出的global symbol
$ nm -g  

//在目录中递归找
$ grep -r <thing-to-find> <folder-to-look-into>

//远程拷贝，如果<destination>的目录不存在，会创建对应的folder
$ rsync -avzP <source_file_or_dir> <destination>


```

### 3. 进程相关
```
//可以显示文件系统和网络访问情况，以及缺页、pagein等
$ fs_usage
```


### 5. lldb
```
//显示所有可用平台
(lldb) platform list

//选择remote-ios为当前平台
(lldb) platform select remote-ios --sysroot --sysroot  /Volumes/Xcode/Xcode.app/Contents/Developer/Platforms/iPhoneOS.platform/Developer/SDKs/iPhoneOS5.1.sdk

//连接到远程的debugserver
(lldb) platform connect connect://192.168.1.8:6789

//列出调试的target
(lldb) target list

//设置断点
// -a <addr> 断在指定的内存地址
// -s <selector> 断在ObjC的消息selector
(lldb) breakpoint set -a 0x30036

//读取内存
// -o 把内存的东西看成Objective-C对象
(lldb) memory read <开始地址> <结束地址>

//给block设置断点，例子：
(lldb) b
UIKit`__250-[_UIViewServiceViewControllerOperator function:]_block_invoke+214

//查找指定的symbol
(lldb) image lookup -vs __snapshotViews

//查找某地址的symbol
(lldb) image lookup -a 0x123456

//查找所有名字中带有XXX_className的objc方法
(lldb) image lookup -rn XXX_className

//打印类定义
(lldb) exp -lobjc -O -- [UIDebuggingInformationOverlay _shortMethodDescription]
<UIDebuggingInformationOverlay: 0x11677e258>:
in UIDebuggingInformationOverlay:
Class Methods:
+ (void) prepareDebuggingOverlay; (0x115da4312)
+ (void) pushDisableApplyingConfigurations; (0x115da45c3)
+ (void) popDisableApplyingConfigurations; (0x115da45d0)
+ (id) overlay; (0x115da4482)
Properties:
@property (retain, nonatomic) UIEvent* lastTouch;  (@synthesize lastTouch = _lastTouch;)
@property (nonatomic) struct CGPoint drawingOrigin;  (@synthesize drawingOrigin = _drawingOrigin;)
@property (readonly, nonatomic) UIDebuggingInformationOverlayViewController* overlayViewController;
@property (retain, nonatomic) UIDebuggingInformationRootTableViewController* rootTableViewController;
@property (nonatomic) BOOL checkingTouches;  (@synthesize checkingTouches = _checkingTouches;)
@property (nonatomic) BOOL touchCaptureEnabled;  (@synthesize touchCaptureEnabled = _touchCaptureEnabled;)
@property (retain, nonatomic) NSMutableArray* touchObservers;  (@synthesize touchObservers = _touchObservers;)
@property (retain, nonatomic) UIWindow* inspectedWindow;  (@synthesize inspectedWindow = _inspectedWindow;)
Instance Methods:
- (void) .cxx_destruct; (0x115da527d)
- (id) hitTest:(struct CGPoint)arg1 withEvent:(id)arg2; (0x115da4b0d)
- (id) lastTouch; (0x115da5228)
- (id) inspectedWindow; (0x115da5203)
- (void) setInspectedWindow:(id)arg1; (0x115da5214)
- (void) setTouchCaptureEnabled:(BOOL)arg1; (0x115da51ce)
- (id) overlayViewController; (0x115da4300)
- (void) setCheckingTouches:(BOOL)arg1; (0x115da51ae)
- (void) toggleFullscreen; (0x115da4ad0)
- (void) toggleVisibility; (0x115da45dd)
- (void) setRootTableViewController:(id)arg1; (0x115da5076)
- (id) rootTableViewController; (0x115da5026)
- (BOOL) checkingTouches; (0x115da519e)
- (id) touchObservers; (0x115da51de)
- (void) setLastTouch:(id)arg1; (0x115da5239)
- (BOOL) touchCaptureEnabled; (0x115da51be)
- (void) setTouchObservers:(id)arg1; (0x115da51ef)
- (struct CGPoint) drawingOrigin; (0x115da524d)
- (void) setDrawingOrigin:(struct CGPoint)arg1; (0x115da5265)
- (id) init; (0x115da423e)
- (id) initWithFrame:(struct CGRect)arg1; (0x115da4f76)
(UIWindow …)

//反汇编oc函数
(lldb) disassemble -n "-[UIDebuggingInformationOverlay init]"
(lldb) disassemble -n "-[UIDebuggingInformationOverlay init]" -c10 //First 10 lines

//刷新UI
(lldb) expression -- (void)[CATransaction flush]

```

lldb换目录/机器debug
```sh
将User1机器上的代码部署到设备上，将dSYM文件拷贝给User2

然后在User2的机器上运行xcode，attach键盘进程，break下来，在lldb命令行中设置如下：

//设置symbol文件路径
lldb> target symbols add <User2机器上dSYM文件路径> 

//设置源机器与当前机器上workarea的对应关系
lldb> settings set target.source-map <User1机器上的代码路径> <User2机器上的代码路径>

然后就可以在xcode GUI中直接设置断点、单步调试等了。
```

### 其他命令

* 统计代码行数、文件数(`cloc`)
```sh
$brew install cloc
$clock work_area_path

$ cloc <CODE_FOLDER>
    3113 text files.
    2937 unique files.                                          
     360 files ignored.

github.com/AlDanial/cloc v 1.74  T=39.83 s (69.2 files/s, 16726.7 lines/s)
---------------------------------------------------------------------------------------
Language                             files          blank        comment           code
---------------------------------------------------------------------------------------
Objective C                            776          35073          18954         203497
C++                                    225          21145          12600         132774
C/C++ Header                          1320          28043          53283          97934
Objective C++                           22           4312           1573          24520
C                                       40           2021           4052          13687
JSON                                   335              1              0           8401
Markdown                                 5            295              0           1005
Bourne Shell                             7            140             84            887
INI                                     12              0              0            373
Python                                   4             72             40            342
XML                                      1              0              0            296
JavaScript                               3             47             12            249
Swift                                    4             47             30            230
make                                     1             29             19            105
HTML                                     1              0              0             73
Windows Module Definition                1              0              0              3
---------------------------------------------------------------------------------------
SUM:                                  2757          91225          90647         484376
---------------------------------------------------------------------------------------
```

* codesign error when call xcodebuild over ssh
    - Move keychain Keys from login/local to system

