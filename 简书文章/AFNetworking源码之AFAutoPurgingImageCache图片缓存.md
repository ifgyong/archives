#### AFAutoPurgingImageCache图片缓存
通过2个protocol解耦，通过协议继承来解耦。
协议相当于 Java定义的接口，只负责声明，谁继承谁实现。

```
@protocol AFImageCache <NSObject>

/**
添加image
 */
- (void)addImage:(UIImage *)image withIdentifier:(NSString *)identifier;

/**
 删除 标示的image
 */
- (BOOL)removeImageWithIdentifier:(NSString *)identifier;

/**
删除所有缓存
 */
- (BOOL)removeAllImages;

/**
 获取imag 使用标示
 */
- (nullable UIImage *)imageWithIdentifier:(NSString *)identifier;
@end


/**
 The `ImageRequestCache` protocol extends the `ImageCache` protocol by adding methods for adding, removing and fetching images from a cache given an `NSURLRequest` and additional identifier.
RequestCache 包含了imageCache协议
 */
@protocol AFImageRequestCache <AFImageCache>

/**
是否缓存该image 
 */
- (BOOL)shouldCacheImage:(UIImage *)image forRequest:(NSURLRequest *)request withAdditionalIdentifier:(nullable NSString *)identifier;

/**
添加该image 使用request
 */
- (void)addImage:(UIImage *)image forRequest:(NSURLRequest *)request withAdditionalIdentifier:(nullable NSString *)identifier;

/**
删除该image 使用request
*/
- (BOOL)removeImageforRequest:(NSURLRequest *)request withAdditionalIdentifier:(nullable NSString *)identifier;

/**
 
获取该image 使用request

 */
- (nullable UIImage *)imageforRequest:(NSURLRequest *)request withAdditionalIdentifier:(nullable NSString *)identifier;

@end

/**
 实现协议的Class
 */
@interface AFAutoPurgingImageCache : NSObject <AFImageRequestCache>

/**
内存的占用大大小
 */
@property (nonatomic, assign) UInt64 memoryCapacity;

/**
当占用超过memoryUsage之后，del 一些内存剩下的最大内存默认是100->60MB
 */
@property (nonatomic, assign) UInt64 preferredMemoryUsageAfterPurge;

/**
 当前最大使用的内存
 */
@property (nonatomic, assign, readonly) UInt64 memoryUsage;


- (instancetype)init;

/**
 Initialies the `AutoPurgingImageCache` instance with the given memory capacity and preferred memory usage
 after purge limit.

 @param memoryCapacity The total memory capacity of the cache in bytes.
 @param preferredMemoryCapacity The preferred memory usage after purge in bytes.

 @return The new `AutoPurgingImageCache` instance.
 */
- (instancetype)initWithMemoryCapacity:(UInt64)memoryCapacity preferredMemoryCapacity:(UInt64)preferredMemoryCapacity;

@end
```

```
生成一个image 计算占用空间的大小和最后的访问日期
- (instancetype)initWithImage:(UIImage *)image identifier:(NSString *)identifier {
    if (self = [self init]) {
        self.image = image;
        self.identifier = identifier;

        CGSize imageSize = CGSizeMake(image.size.width * image.scale, image.size.height * image.scale);
//每一个像素是rgba 所以空间大小 = 像素个数*4
//为什么一个像素占4bytes呢？因为rgba(255,255,2552,1);
//一个255需要8位，3个需要3个8位，最后一个最大值是1，根据内存对齐，占用为4*8/8=4byte
        CGFloat bytesPerPixel = 4.0;
        CGFloat bytesPerSize = imageSize.width * imageSize.height;
        self.totalBytes = (UInt64)bytesPerPixel * (UInt64)bytesPerSize;
        self.lastAccessDate = [NSDate date];
    }
    return self;
}
//默认是最大100MB，减小之后是60MB
- (instancetype)init {
    return [self initWithMemoryCapacity:100 * 1024 * 1024 preferredMemoryCapacity:60 * 1024 * 1024];
}
//初始化 cache
- (instancetype)initWithMemoryCapacity:(UInt64)memoryCapacity preferredMemoryCapacity:(UInt64)preferredMemoryCapacity {
    if (self = [super init]) {
        self.memoryCapacity = memoryCapacity;
        self.preferredMemoryUsageAfterPurge = preferredMemoryCapacity;
//cache是用NSMutableDictionary实现的
        self.cachedImages = [[NSMutableDictionary alloc] init];

        NSString *queueName = [NSString stringWithFormat:@"com.alamofire.autopurgingimagecache-%@", [[NSUUID UUID] UUIDString]];
//新建队列
        self.synchronizationQueue = dispatch_queue_create([queueName cStringUsingEncoding:NSASCIIStringEncoding], DISPATCH_QUEUE_CONCURRENT);

        [[NSNotificationCenter defaultCenter]
         addObserver:self
         selector:@selector(removeAllImages)
         name:UIApplicationDidReceiveMemoryWarningNotification
         object:nil];

    }
    return self;
}
//队列中同步执行 返回当前内存使用大小 为什么同步呢?
//因为没有计算，只是读取值大小，只需在队列中同步执行即可。
//同步不会锁死？不是mainqueue，不会锁死。
- (UInt64)memoryUsage {
    __block UInt64 result = 0;
    dispatch_sync(self.synchronizationQueue, ^{
        result = self.currentMemoryUsage;
    });
    return result;
}
//添加image
- (void)addImage:(UIImage *)image withIdentifier:(NSString *)identifier {
    dispatch_barrier_async(self.synchronizationQueue, ^{
        AFCachedImage *cacheImage = [[AFCachedImage alloc] initWithImage:image identifier:identifier];

        AFCachedImage *previousCachedImage = self.cachedImages[identifier];
        if (previousCachedImage != nil) {
            self.currentMemoryUsage -= previousCachedImage.totalBytes;
        }

        self.cachedImages[identifier] = cacheImage;
        self.currentMemoryUsage += cacheImage.totalBytes;
    });
//通过栅栏函数异步执行 刷新内存image
    dispatch_barrier_async(self.synchronizationQueue, ^{
        if (self.currentMemoryUsage > self.memoryCapacity) {
            UInt64 bytesToPurge = self.currentMemoryUsage - self.preferredMemoryUsageAfterPurge;
            NSMutableArray <AFCachedImage*> *sortedImages = [NSMutableArray arrayWithArray:self.cachedImages.allValues];
            NSSortDescriptor *sortDescriptor = [[NSSortDescriptor alloc] initWithKey:@"lastAccessDate"
                                                                           ascending:YES];
//使用最后访问时间排序
            [sortedImages sortUsingDescriptors:@[sortDescriptor]];

            UInt64 bytesPurged = 0;
//删除AFCachedImage 直到小余设置的值
            for (AFCachedImage *cachedImage in sortedImages) {
                [self.cachedImages removeObjectForKey:cachedImage.identifier];
                bytesPurged += cachedImage.totalBytes;
                if (bytesPurged >= bytesToPurge) {
                    break;
                }
            }
            self.currentMemoryUsage -= bytesPurged;
        }
    });
}

- (BOOL)removeImageWithIdentifier:(NSString *)identifier {
    __block BOOL removed = NO;
//栅栏函数 同步执行删除操作 占用内存大小更新
    dispatch_barrier_sync(self.synchronizationQueue, ^{
        AFCachedImage *cachedImage = self.cachedImages[identifier];
        if (cachedImage != nil) {
            [self.cachedImages removeObjectForKey:identifier];
            self.currentMemoryUsage -= cachedImage.totalBytes;
            removed = YES;
        }
    });
    return removed;
}

- (BOOL)removeAllImages {
    __block BOOL removed = NO;
//栅栏函数同步删除all image 当前使用内存大小设置为0
    dispatch_barrier_sync(self.synchronizationQueue, ^{
        if (self.cachedImages.count > 0) {
            [self.cachedImages removeAllObjects];
            self.currentMemoryUsage = 0;
            removed = YES;
        }
    });
    return removed;
}
```
![图片缓存概览](https://upload-images.jianshu.io/upload_images/783986-49aaddd278c37713.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

