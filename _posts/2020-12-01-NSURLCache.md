# NSURLCache缓存的使用

### 缓存策略
App 中有3种网络缓存存策略（只对get请求做缓存）
1.不返回缓存数据，实时返回接口数据
2.首先返回缓存数据，接口数据覆盖缓存，并返回
3.默认不返回缓存数据，接口失败时返回

同时对 WebView 也使用的缓存策略（UIWebView的NSURLRequest设置了缓存策略，NSURLCache 会自动根据缓存策略来使用）。

都是使用 NSURLCache 来实现的。不需要单独写工具类来实现缓存。有缓存策略同时也有清除机制分自动清除和手动清除。


### NSURLRequest 的缓存策略
1.NSURLRequest需要一个缓存策略参数来说明它请求的url何如缓存数据的：CachePolicy类型。
(1)NSURLRequestUseProtocolCachePolicy：NSURLRequest默认的cachepolicy，使用Protocol协议定义。
(2)NSURLRequestReloadIgnoringCacheData：忽略缓存直接从原始地址下载。
(3)NSURLRequestReturnCacheDataElseLoad：只有在cache中不存在data时才从原始地址下载。
(4)NSURLRequestReturnCacheDataDontLoad：只使用cache数据，如果不存在cache，请求失败;用于没有建立网络连接离线模式;
(5)NSURLRequestReloadIgnoringLocalAndRemoteCacheData：忽略本地和远程的缓存数据，直接从原始地址下载，与NSURLRequestReloadIgnoringCacheData类似。 
(6)NSURLRequestReloadRevalidatingCacheData:验证本地数据与远程数据是否相同，如果不同则下载远程数据，否则使用本地数据。
(7)说明：5和6苹果暂未实现。

### NSURLCache 使用

在 Appdelegate 中的 didFinishLaunchingWithOptions 方法中实现

```
NSURLCache *URLCache = [[NSURLCache alloc] initWithMemoryCapacity:4 * 1024 * 1024 diskCapacity:20 * 1024 * 1024 diskPath:nil];
        [NSURLCache setSharedURLCache:URLCache];
```
在自定义网络底层请求方法中根据不同的缓存策略进行不同的业务缓存操作核心方法

```
写缓存
 NSCachedURLResponse *cachedResp = [[NSCachedURLResponse alloc] initWithResponse:response
                                                                              data:responseObject];
                                                                               [[NSURLCache sharedURLCache] storeCachedResponse:cachedResp forRequest:URLRequest];         

读缓存
NSCachedURLResponse *cachedResp = [[NSURLCache sharedURLCache] cachedResponseForRequest:URLRequest];
```

### NSCachedURLResponse（包装了一下系统缓存机制的对象）

NSURLCacheStoragePolicy 缓存策略有三种
enum
{
    NSURLCacheStorageAllowed,
    NSURLCacheStorageAllowedInMemoryOnly,
    NSURLCacheStorageNotAllowed,
};

主要是对其属性 data 来获取请求数据。


### 清除缓存机制

在app 启动是清除上一次 app 所产生的缓存。只缓存本次使用中 app 的请求。
