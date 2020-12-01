# NSURLProtocol

看项目代码时发现了这个类，没用过，也没听过。查阅一翻发现这个类的功能非常强大，能拦截 URL loading system 里面的所有网络请求，拦截之后可以自定义处理或特殊化处理请求。

我们项目中是用 NSURLProtocol 来截断请求，用于 webview 加载 .webp 图片。

一张图更清晰的认识 NSURLProtocol 在网络中的位置:

 ![pic](https://tva1.sinaimg.cn/large/0081Kckwgy1gl8dayz2vzj30b40bkafx.jpg)
 
 我们的项目中并没有使用 WKWebView。要使 WKWebView 支持 NSURLProtocol（[点击](https://blog.yeatse.com/2016/10/26/support-nsurlprotocol-in-wkwebview/)）。
 
 
### 使用 NSURLProtocol

NSURLProtocol 只是一个抽象类，只能使用子类。

```
@interface WebpURLProtocol : NSURLProtocol
```

NSURLProtocol 分五个步骤：
注册->拦截->转发->回调->结束




#### 注册

注册发生在 applicationdidFinish 方法中。

[NSURLProtocol registerClass:[WebpURLProtocol class]];



#### 拦截

重写 NSURLProcotol 的  

```
+ (BOOL)canInitWithRequest:(NSURLRequest *)request

```

return YES 则进行拦截该 request。

在此方法里面可以进行一系列自定义判断。但下面的判断是必须的，防止循环调用该方法。

```
// request 是否为已请求过的
if (nil != [self propertyForKey:WebpURLRequestHandledKey inRequest:request]) {
        return NO;
    }
```

同时项目中是通过还判断了URL的 host，和判断了是否为图片的请求，都是基于 request来进行判断的。

```
+ (NSURLRequest *)canonicalRequestForRequest:(NSURLRequest *)request
```
拦截后的 request 可以在此方法中进行修改，如设置请求头，项目中设置了headerFeild中 User-Agent 字段。


#### 转发

通过 ``` - (void)startLoading ``` 中实现 request 的重新转发，同时要标记该请求。

```
[NSURLProtocol setProperty:@YES forKey:TNWebpURLRequestHandledKey inRequest:mRequest];
```


实现转发可以基于 NSURLConnection，NSURLSession。项目中是通过 NSURLSessionConfiguration 来实现转发的。

```
 self.session = [NSURLSession sessionWithConfiguration:config delegate:self delegateQueue:sessionDelegateQueue];
       
[[self.session dataTaskWithRequest:mRequest] resume];
```


#### 结束

在 ```- (void)stopLoading  ``` 方法中实现结束一般都是 cancel 掉请求，将 NSURLSession 或 NSURLConnection 对象置为 nil。


#### 回调

项目中是通过 NSURLProtocol 的以下几个方法来进行回调的，回调给原来发网络请求的地方。项目中是通过 NSURLSession 重发的网络请求，则在 NSURLSession 对象的回调方法中实现。同时在 NSURLSession 回调方法中也实现了一些自定义方法（是对回调的data进行了一些处理）。

```
[[self client] URLProtocol:self wasRedirectedToRequest:redirectRequest redirectResponse:response] // 通知发生重定向
[self.client URLProtocol:self didFailWithError:error]; // 请求失败
[self.client URLProtocolDidFinishLoading:self];  // 完成
[self.client URLProtocol:self didReceiveResponse:response cacheStoragePolicy:NSURLCacheStorageNotAllowed]; // 收到response 
[self.client URLProtocol:self didLoadData:data]; // 加载到数据

```

下面这个方法是本次请求服务器通知要重定向，则会回调此方法

```
- (void)URLSession:(NSURLSession *)session task:(NSURLSessionTask *)task willPerformHTTPRedirection:(NSHTTPURLResponse *)response newRequest:(NSURLRequest *)newRequest completionHandler:(void (^)(NSURLRequest *))completionHandler
{
    // 重定向：默认是可以的。可以通过 completionHandler(nil) 来禁止重定向。同时可以通过 cancel 方法来禁止会话达到禁止重定向
    NSMutableURLRequest *redirectRequest;
    
    redirectRequest = [newRequest mutableCopy];
    [[self class] removePropertyForKey:TNWebpURLRequestHandledKey inRequest:redirectRequest]; // remove掉才会有新的 request
    
    [self.session invalidateAndCancel];
    
        //  3.回调
    [[self client] URLProtocol:self wasRedirectedToRequest:redirectRequest redirectResponse:response]; // 告诉发生了重定向
    [[self client] URLProtocol:self didFailWithError:[NSError errorWithDomain:NSCocoaErrorDomain code:NSUserCancelledError userInfo:nil]]; // cancel掉之后会传一个错误给请求   
}
```

是默认允许发生重定向的。当然也可以禁止重定向。

1. 通过completionHandler(nil) 可以禁止重定向。
2. 通过 cancle 请求禁止重定向。


```
// 收到 respond
- (void)URLSession:(NSURLSession *)session dataTask:(NSURLSessionDataTask *)dataTask didReceiveResponse:(NSURLResponse *)response completionHandler:(void (^)(NSURLSessionResponseDisposition disposition))completionHandler
{
    // 此方法可以通过自定义一个 response返回 或者 返回请求的 response
    // 3.回调
    [self.client URLProtocol:self didReceiveResponse:response cacheStoragePolicy:NSURLCacheStorageNotAllowed];
    [self performOnThread:self.clientThread block:^{
            completionHandler(NSURLSessionResponseAllow);
}

- (void)URLSession:(NSURLSession *)session dataTask:(NSURLSessionDataTask *)dataTask didReceiveData:(NSData *)data{
    [[self client] URLProtocol:self didLoadData:data];
}

- (void)URLSession:(NSURLSession *)session task:(NSURLSessionTask *)task didCompleteWithError:(nullable NSError *)error{
    if (error) {
        //  3.回调
        [self.client URLProtocol:self didFailWithError:error];
    }else{
        //  3.回调
        [[self client] URLProtocol:self didLoadData:imageData];
                }];
        //  3.回调
        [self.client URLProtocolDidFinishLoading:self];
       
    }
}
```


当然项目中每个 NSURLSession 回调方法中都做了一些自定义处理。

这就是我们项目中对 NSURLProtocol 的使用，当然它还有很多其他的应用如：
1.网络请求 mock stub， 如 [OHHTTPStubs](https://github.com/AliSoftware/OHHTTPStubs) 就是基于 NSURLProtocok。
	2.配合实现[HTTPDNS](https://allluckly.cn/%E6%8A%95%E7%A8%BF/tougao75)。  等等。