---
layout: post
title:  "SDWebImage学习笔记"
date:   2016-06-07 14:55:37
categories: animate
---

SDWebImage是iOS开发中最常用的第三方图片下载库,本文是对SDWebImage源码的学习笔记.

源码地址：https://github.com/rs/SDWebImage

主要的功能有:

- An UIImageView category adding web image and cache management to the Cocoa Touch framework (添加UIImageView的分类实现网络图片加载和缓存的功能
- An asynchronous image downloader (异步图片下载器)
- An asynchronous memory + disk image caching with automatic cache expiration handling （异步的内存和磁盘缓存）
- Animated GIF support （支持GIF图片）
- WebP format support （支持WebP图片）
- A background image decompression  （后台图片解压缩)
- A guarantee that the same URL won't be downloaded several times （确保相同的URL不会被多次下载）
- A guarantee that bogus URLs won't be retried again and again （确保假的URL不会被重复加载）
- A guarantee that main thread will never be blocked （确保主线程不会被堵塞）
- Performances! (性能!)
- Use GCD and ARC （使用GCD和ARC）
- Arm64 support  （支持64位）

## 关于下载

### 1.SDWebImageDownloader

SDWebImage的下载操作是由SDWebImageDownloader来完成的

在头文件中定义了2个枚举,分别是关于下载选项的SDWebImageDownloaderOptions和下载顺序的SDWebImageDownloaderExecutionOrder 以及3个下载回调block

SDWebImageDownloaderOptions:

```
typedef NS_OPTIONS(NSUInteger, SDWebImageDownloaderOptions) {
    
    /**  将图片下载放到低优先级 */
    SDWebImageDownloaderLowPriority = 1 << 0,
    
    /** 下载优先 */
    SDWebImageDownloaderProgressiveDownload = 1 << 1,

    /**
     * 默认情况下是不使用NSURLCache缓存 当开启后使用NSURLCache
     */
    SDWebImageDownloaderUseNSURLCache = 1 << 2,

    /**
     *如果从NSURLCache读取image/imageData,则设置完成block的参数设为nil
     */

    SDWebImageDownloaderIgnoreCachedResponse = 1 << 3,
    /**
     * 在iOS4之后的系统 允许在后台下载图片
     * 后台会分配额外的时间让请求完成，如果后台任务推出了下载操作也会被取消
     */

    SDWebImageDownloaderContinueInBackground = 1 << 4,

    /**
     * 通过设置NSMutableURLRequest.HTTPShouldHandleCookies = YES来处理存储在NSHTTPCookieStore中的cookie
     */
    SDWebImageDownloaderHandleCookies = 1 << 5,

    /**
     * 允许不受信任的SSL证书。主要用于测试目的。
     */
    SDWebImageDownloaderAllowInvalidSSLCertificates = 1 << 6,

    /**
     * 将图片下载放到高优先级
     */
    SDWebImageDownloaderHighPriority = 1 << 7,
};

```
SDWebImageDownloaderExecutionOrder:

```
typedef NS_ENUM(NSInteger, SDWebImageDownloaderExecutionOrder) {
    /**
     * 默认值，所有的操作将会遵循先进先出原则(FIFO)
     */
    SDWebImageDownloaderFIFOExecutionOrder,

    /**
     * 所有下载操作遵循先进后出原则 (LIFO)
     */
    SDWebImageDownloaderLIFOExecutionOrder
};
```

下载回调block:

```
/** 下载进度 */
typedef void(^SDWebImageDownloaderProgressBlock)(NSInteger receivedSize, NSInteger expectedSize);

/** 下载完成 */
typedef void(^SDWebImageDownloaderCompletedBlock)(UIImage *image, NSData *data, NSError *error, BOOL finished);

/** header过滤 */
typedef NSDictionary *(^SDWebImageDownloaderHeadersFilterBlock)(NSURL *url, NSDictionary *headers);

```

SDWebImageDownloader是以单例的形式来管理下载操作的

```
+ (SDWebImageDownloader *)sharedDownloader {
    static dispatch_once_t once;
    static id instance;
    dispatch_once(&once, ^{
        instance = [self new];
    });
    return instance;
}
```
在.m 的实现中定义了几个属性

有 downloadQueue: 负责下载操作的队列,它能设置最大并发数,添加操作,暂停操作和取消操作

 lastAddedOperation:最后一个添加的Operation

```
设置最大并发数默认是6个,可以通过maxConcurrentDownloads这个属性来修改
_downloadQueue.maxConcurrentOperationCount = 6;

//暂停操作
- (void)setSuspended:(BOOL)suspended {
    [self.downloadQueue setSuspended:suspended];
}

//取消所有的操作
- (void)cancelAllDownloads {
    [self.downloadQueue cancelAllOperations];
}

```

每个图片的下载信息以字典的形式存储起来:URLCallbacks 它以URL为key,value是一个可变数组
数组里面装着字典,字典里面就是下载进度和下载完成的block回调

```
//类似这种形式
NSMutableDictionary *URLCallbacks = @{@"URL":@[@"kProgressCallbackKey":progressBlock,@"kCompletedCallbackKey":completedBlock]....};


```

关于下载的2个核心方法 

//下载图片
- (id <SDWebImageOperation>)downloadImageWithURL: options: progress: completed:

//添加操作
- (void)addProgressCallback: completedBlock: forURL: createCallback:

```
- (void)addProgressCallback:(SDWebImageDownloaderProgressBlock)progressBlock completedBlock:(SDWebImageDownloaderCompletedBlock)completedBlock forURL:(NSURL *)url createCallback:(SDWebImageNoParamsBlock)createCallback {

    // 因为URL将作为字典的key所以不能为空,当url为空时如果完成的block回调存在则设置image和data为空
    if (url == nil) {
        if (completedBlock != nil) {
            completedBlock(nil, nil, nil, NO);
        }
        return;
    }

    //为了线程间的安全使用dispatch_barrier_sync保证在同一时间只有一条线程对URLCallbacks进行修改
    dispatch_barrier_sync(self.barrierQueue, ^{
        BOOL first = NO;
        if (!self.URLCallbacks[url]) {
            self.URLCallbacks[url] = [NSMutableArray new];
            first = YES;
        }


        NSMutableArray *callbacksForURL = self.URLCallbacks[url];
        NSMutableDictionary *callbacks = [NSMutableDictionary new];
        if (progressBlock) callbacks[kProgressCallbackKey] = [progressBlock copy];
        if (completedBlock) callbacks[kCompletedCallbackKey] = [completedBlock copy];
        [callbacksForURL addObject:callbacks];
        
        //添加新的回调信息
        self.URLCallbacks[url] = callbacksForURL;

        if (first) {
            createCallback();
        }
    });
}


```


