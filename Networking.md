
### 1. 网络请求返回状态调试工具：
> 
> http://httpstat.us
> 
> 返回200：
> http://httpstat.us/200

### 2. tcpdump抓包
```sh
//Start active device
$rvictl -s 7d0982cc809a1ee90ac726267f1f02732ac94320        //device udid

//List ongoings active devices
$rvictl -l

//tcp dump
$sudo tcpdump -i rvi0 -w ~/Desktop/output.pcap

//Stop
$rvictl -x 7d0982cc809a1ee90ac726267f1f02732ac94320    

//生成的 output.pcap 可用WireShark来查看。
```

### 3. TCP_NODELAY & TCP_QUICKACK

```
By default TCP users Nagle’s algorithm to collect small outgoing packets to send all at once. This have a detrimental effect on latency.

Using TCP_NODELAY and TCP_CORK to improve network latency
1. Applications that require lower latency on every packet sent should be run on sockets with TCP_NODELAY enabled. It can be enabled through the setsockopt command with the sockets API:
# int one = 1;

# setsockopt(descriptor, SOL_TCP, TCP_NODELAY, &one, sizeof(one));

2. For this to be used effectively, applications must avoid doing small, logically related buffer writes. Because TCP_NODELAY is enabled, these small writes will make TCP send these multiple buffers as individual packets, which can result in poor overall performance.
If applications have several buffers that are logically related and that should be sent as one packet it could be possible to build a contiguous packet in memory and then send the logical packet to TCP, on a socket configured with TCP_NODELAY.
Alternatively, create an I/O vector and pass it to the kernel using writev on a socket configured with TCP_NODELAY.
3. Another option is to use TCP_CORK, which tells TCP to wait for the application to remove the cork before sending any packets. This command will cause the buffers it receives to be appended to the existing buffers. This allows applications to build a packet in kernel space, which may be required when using different libraries that provides abstractions for layers. To enable TCP_CORK, set it to a value of 1 using the setsockopt sockets API (this is known as "corking the socket"):
# int one = 1;

# setsockopt(descriptor, SOL_TCP, TCP_CORK, &one, sizeof(one));

4. When the logical packet has been built in the kernel by the various components in the application, tell TCP to remove the cork. TCP will send the accumulated logical packet right away, without waiting for any further packets from the application.
# int zero = 0;

# setsockopt(descriptor, SOL_TCP, TCP_CORK, &zero, sizeof(zero));

Related Manual Pages
For more information, or for further reading, the following man pages are related to the information given in this section.
* tcp(7)
* setsockopt(3p)
* setsockopt(2)
```

### 4. nc命令

```
//TCP的server端
$nc -l 1234    //监听1234端口

//TCP的client端
$nc 127.0.0.1 1234 //连接127.0.0.1的1234端口，后面可以输入文字
$nc 127.0.0.1 1234 < input.file    //连接1234端口，并将文件传过去

//详见： man nc

```

### 5. iOS connection per Host

`NSURLConnection`限制同一host只能同时建立4个连接，多了会超时（这应该是为什么频繁给几个server发请求的时候会出现超时）：
```objc
//iOS限制对同一个host最多4个连接，第五个请求会超时
- (void)startConnections {
    NSMutableArray *connections= [NSMutableArray array];
    for (int i= 0; i < 5; i++) {
        NSURL *url= [NSURL URLWithString:@"https://push.lightstreamer.com/lightstreamer/create_session.txt?LS_user=&LS_adapter_set=DEMO&LS_ios_version=1.0"];
        NSMutableURLRequest *req= [NSMutableURLRequest requestWithURL:url];
        req.timeoutInterval = 15;
        NSURLConnection *conn= [[NSURLConnection alloc] initWithRequest:req delegate:self startImmediately:NO];
        [connections addObject:conn];
    }
    
    for (NSURLConnection *conn in connections)
        [conn start];
}

- (void)viewDidLoad {
    [super viewDidLoad];
    
    [self startConnections];
}


- (void)didReceiveMemoryWarning {
    [super didReceiveMemoryWarning];
    // Dispose of any resources that can be recreated.
}

- (void)connection:(NSURLConnection *)connection didReceiveResponse:(NSURLResponse *)response
{
    NSLog(@"Connection %p did receive response %ld", connection, [(NSHTTPURLResponse *) response statusCode]);
}

- (void)connection:(NSURLConnection *)connection didReceiveData:(NSData *)data
{
    NSLog(@"Connection %p did receive %ld bytes:\n%@", connection, [data length], [[NSString alloc] initWithData:data encoding:NSUTF8StringEncoding]);
}

- (void)connection:(NSURLConnection *)connection didFailWithError:(NSError *)error
{
    NSLog(@"Connection %p did fail with error %@", connection, error);
}
```

`NSURLSession`的`NSURLSessionConfiguration`可以设置`httpMaximumConnectionsPerHost`,调大以后就没问题了：
```objc
- (void)startConnections_session {
    NSURL *url= [NSURL URLWithString:@"https://push.lightstreamer.com/lightstreamer/create_session.txt?LS_user=&LS_adapter_set=DEMO&LS_ios_version=1.0"];
    NSURLRequest *request = [NSURLRequest requestWithURL:url];
    NSURLSessionConfiguration *configuration = [NSURLSessionConfiguration defaultSessionConfiguration];
    configuration.timeoutIntervalForRequest = 10;
    [configuration setHTTPMaximumConnectionsPerHost:10];
    NSURLSession *session = [NSURLSession sessionWithConfiguration:configuration];
    for (int i = 0; i < 6; i++) {
        NSURLSessionDataTask *task = [session dataTaskWithRequest:request
                                                completionHandler:
                                      ^(NSData *data, NSURLResponse *response, NSError *error) {
                                          NSLog(@"Task %d did receive response %ld, error: %@", i, [(NSHTTPURLResponse *) response statusCode], error);
                                      }];
        
        [task resume];
    }
}

```
