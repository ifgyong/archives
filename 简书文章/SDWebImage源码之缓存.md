#### SDWebImage缓存分为两类
- SDMemoryCache
- SDDiskCache

SDMemoryCache

源码：
```
@class SDImageCacheConfig;
// A protocol to allow custom memory cache used in SDImageCache.
//协议继承NSObject
@protocol SDMemoryCache <NSObject>

@required
/**
 Create a new memory cache instance with the specify cache config. You can check `maxMemoryCost` and `maxMemoryCount` used for memory cache.

 @param config The cache config to be used to create the cache.
 @return The new memory cache instance.
 */
//创建MemoryCache  config：配置处理Cache的配置文件
- (nonnull instancetype)initWithConfig:(nonnull SDImageCacheConfig *)config;

/**
 Returns the value associated with a given key.
 
 @param key An object identifying the value. If nil, just return nil.
 @return The value associated with key, or nil if no value is associated with key.
 */
//根据key取出Obj
- (nullable id)objectForKey:(nonnull id)key;

/**
 Sets the value of the specified key in the cache (0 cost).
 
 @param object The object to be stored in the cache. If nil, it calls `removeObjectForKey:`.
 @param key    The key with which to associate the value. If nil, this method has no effect.
 @discussion Unlike an NSMutableDictionary object, a cache does not copy the key
 objects that are put into it.
 */
//缓存 key value
- (void)setObject:(nullable id)object forKey:(nonnull id)key;

/**
 Sets the value of the specified key in the cache, and associates the key-value
 pair with the specified cost.
 
 @param object The object to store in the cache. If nil, it calls `removeObjectForKey`.
 @param key    The key with which to associate the value. If nil, this method has no effect.
 @param cost   The cost with which to associate the key-value pair.
 @discussion Unlike an NSMutableDictionary object, a cache does not copy the key
 objects that are put into it.
 */
//设置每个key value 花费的大小 配置有最大文件设置当超过最大值的时候自动删除超出部分
- (void)setObject:(nullable id)object forKey:(nonnull id)key cost:(NSUInteger)cost;

/**
 Removes the value of the specified key in the cache.
 
 @param key The key identifying the value to be removed. If nil, this method has no effect.
 */
//根据key删除value
- (void)removeObjectForKey:(nonnull id)key;

/**
 Empties the cache immediately.
 */
//删除所有的obj
- (void)removeAllObjects;

@end

// A memory cache which auto purge the cache on memory warning and support weak cache.
//继承NSCache 好处是可以自己管理释放条件 
@interface SDMemoryCache <KeyType, ObjectType> : NSCache <KeyType, ObjectType> <SDMemoryCache>
//只读的 配置文件
@property (nonatomic, strong, nonnull, readonly) SDImageCacheConfig *config;

@end
```
#### SDMemoryCache解析：
```
//没有使用config初始化 直接自己初始化SDImageCacheConfig
- (instancetype)init {
    self = [super init];
    if (self) {
        _config = [[SDImageCacheConfig alloc] init];
        [self commonInit];
    }
    return self;
}

- (instancetype)initWithConfig:(SDImageCacheConfig *)config {
    self = [super init];
    if (self) {
        _config = config;
        [self commonInit];
    }
    return self;
}

- (void)commonInit {
//NSMapTable 可以设置弱引用 不会copy出来一份新的地址
    self.weakCache = [[NSMapTable alloc] initWithKeyOptions:NSPointerFunctionsStrongMemory valueOptions:NSPointerFunctionsWeakMemory capacity:0];
//信号量 读写锁
    self.weakCacheLock = dispatch_semaphore_create(1);
    
    SDImageCacheConfig *config = self.config;
//最大内存
    self.totalCostLimit = config.maxMemoryCost;
//最大数量
    self.countLimit = config.maxMemoryCount;
    //动态接受设置参数
    [config addObserver:self forKeyPath:NSStringFromSelector(@selector(maxMemoryCost)) options:0 context:SDMemoryCacheContext];
    [config addObserver:self forKeyPath:NSStringFromSelector(@selector(maxMemoryCount)) options:0 context:SDMemoryCacheContext];
    
#if SD_UIKIT
    [[NSNotificationCenter defaultCenter] addObserver:self
                                             selector:@selector(didReceiveMemoryWarning:)
                                                 name:UIApplicationDidReceiveMemoryWarningNotification
                                               object:nil];
#endif
}



- (void)didReceiveMemoryWarning:(NSNotification *)notification {
    // Only remove cache, but keep weak cache
//低内存警告时候直接调用super删除allObjects
    [super removeAllObjects];
}


// `setObject:forKey:` just call this with 0 cost. Override this is enough
//存储key value 花费大小
- (void)setObject:(id)obj forKey:(id)key cost:(NSUInteger)g {
    [super setObject:obj forKey:key cost:g];
//如果不使用shouldUseWeakMemoryCache 默认 YES，直接return
    if (!self.config.shouldUseWeakMemoryCache) {
        return;
    }
    if (key && obj) {
        // 写入锁 每次只能一个执行
        SD_LOCK(self.weakCacheLock);
        [self.weakCache setObject:obj forKey:key];
        SD_UNLOCK(self.weakCacheLock);
//释放锁
    }
}

- (id)objectForKey:(id)key {
//首先 根据key取出来value
    id obj = [super objectForKey:key];
//如果没使用shouldUseWeakMemoryCache 直接返回
    if (!self.config.shouldUseWeakMemoryCache) {
        return obj;
    }
    if (key && !obj) {
        // 读数据锁 
        SD_LOCK(self.weakCacheLock);
        obj = [self.weakCache objectForKey:key];
        SD_UNLOCK(self.weakCacheLock);
        if (obj) {
            // Sync cache
            NSUInteger cost = 0;
            if ([obj isKindOfClass:[UIImage class]]) {
                cost = [(UIImage *)obj sd_memoryCost];
            }
//同步 image 到Super 因为上边从Super没有取出来，所以再次加入缓存中
            [super setObject:obj forKey:key cost:cost];
        }
    }
    return obj;
}
//删除key value
- (void)removeObjectForKey:(id)key {
    [super removeObjectForKey:key];
    if (!self.config.shouldUseWeakMemoryCache) {
        return;
    }
    if (key) {
        // Remove weak cache
        SD_LOCK(self.weakCacheLock);
        [self.weakCache removeObjectForKey:key];
        SD_UNLOCK(self.weakCacheLock);
    }
}

- (void)removeAllObjects {
    [super removeAllObjects];
    if (!self.config.shouldUseWeakMemoryCache) {
        return;
    }
    // Manually remove should also remove weak cache
    SD_LOCK(self.weakCacheLock);
    [self.weakCache removeAllObjects];
    SD_UNLOCK(self.weakCacheLock);
}
#endif

#pragma mark - KVO

- (void)observeValueForKeyPath:(NSString *)keyPath ofObject:(id)object change:(NSDictionary<NSKeyValueChangeKey,id> *)change context:(void *)context {
    if (context == SDMemoryCacheContext) {
        if ([keyPath isEqualToString:NSStringFromSelector(@selector(maxMemoryCost))]) {
//观察者 监听 设置的maxMemoryCost变动
            self.totalCostLimit = self.config.maxMemoryCost;
        } else if ([keyPath isEqualToString:NSStringFromSelector(@selector(maxMemoryCount))]) {
//观察者 监听 设置的maxMemoryCount变动
            self.countLimit = self.config.maxMemoryCount;
        }
    } else {
        [super observeValueForKeyPath:keyPath ofObject:object change:change context:context];
    }
}
```
#### SDWebImageCompat解析：
```
#import "SDWebImageCompat.h"

@class SDImageCacheConfig;
// A protocol to allow custom disk cache used in SDImageCache.
@protocol SDDiskCache <NSObject>

// All of these method are called from the same global queue to avoid blocking on main queue and thread-safe problem. But it's also recommend to ensure thread-safe yourself using lock or other ways.
@required
//使用config初始化
- (nullable instancetype)initWithCachePath:(nonnull NSString *)cachePath config:(nonnull SDImageCacheConfig *)config;

//是否包含key
- (BOOL)containsDataForKey:(nonnull NSString *)key;

//根据key取出NSData数据
- (nullable NSData *)dataForKey:(nonnull NSString *)key;

//设置data根据key
- (void)setData:(nullable NSData *)data forKey:(nonnull NSString *)key;

//删除key
- (void)removeDataForKey:(nonnull NSString *)key;

//删除all data
- (void)removeAllData;

//删除过期data
- (void)removeExpiredData;

//获取key的路径
- (nullable NSString *)cachePathForKey:(nonnull NSString *)key;

//cache的数量
- (NSUInteger)totalCount;

//cache的大小
- (NSUInteger)totalSize;

@end

// The built-in disk cache
@interface SDDiskCache : NSObject <SDDiskCache>
//只读的config 只能通过函数配置
@property (nonatomic, strong, readonly, nonnull) SDImageCacheConfig *config;
//进制使用init
- (nonnull instancetype)init NS_UNAVAILABLE;

//移动data 从srcPath到dstPath
- (void)moveCacheDirectoryFromPath:(nonnull NSString *)srcPath toPath:(nonnull NSString *)dstPath;



#import "SDDiskCache.h"
#import "SDImageCacheConfig.h"
#import <CommonCrypto/CommonDigest.h>

@interface SDDiskCache ()
//cache的路径
@property (nonatomic, copy) NSString *diskCachePath;
//fileManger
@property (nonatomic, strong, nonnull) NSFileManager *fileManager;

@end

@implementation SDDiskCache
//禁用init函数
- (instancetype)init {
    NSAssert(NO, @"Use `initWithCachePath:` with the disk cache path");
    return nil;
}

#pragma mark - SDcachePathForKeyDiskCache Protocol
- (instancetype)initWithCachePath:(NSString *)cachePath config:(nonnull SDImageCacheConfig *)config {
    if (self = [super init]) {
        _diskCachePath = cachePath;
        _config = config;
        [self commonInit];
    }
    return self;
}

- (void)commonInit {
    if (self.config.fileManager) {
        self.fileManager = self.config.fileManager;
    } else {
        self.fileManager = [NSFileManager new];
    }
}

- (BOOL)containsDataForKey:(NSString *)key {
    NSParameterAssert(key);
    NSString *filePath = [self cachePathForKey:key];
    BOOL exists = [self.fileManager fileExistsAtPath:filePath];
    //缓存没有的话 再用fileManger查询去掉格式一下
       if (!exists) {
        exists = [self.fileManager fileExistsAtPath:filePath.stringByDeletingPathExtension];
    }
    
    return exists;
}

- (NSData *)dataForKey:(NSString *)key {
//判断是否有key
    NSParameterAssert(key);
    NSString *filePath = [self cachePathForKey:key];
//读取data
    NSData *data = [NSData dataWithContentsOfFile:filePath options:self.config.diskCacheReadingOptions error:nil];
    if (data) {
        return data;
    }
   
    data = [NSData dataWithContentsOfFile:filePath.stringByDeletingPathExtension options:self.config.diskCacheReadingOptions error:nil];
    if (data) {
        return data;
    }
    
    return nil;
}

- (void)setData:(NSData *)data forKey:(NSString *)key {
//判断是否有key 和data
    NSParameterAssert(data);
    NSParameterAssert(key);
    if (![self.fileManager fileExistsAtPath:self.diskCachePath]) {
        [self.fileManager createDirectoryAtPath:self.diskCachePath withIntermediateDirectories:YES attributes:nil error:NULL];
    }
    //新建文件失败 然后再调用data writetoURL 写入Disk
    // get cache Path for image key
    NSString *cachePathForKey = [self cachePathForKey:key];
    // transform to NSUrl
    NSURL *fileURL = [NSURL fileURLWithPath:cachePathForKey];
    
    [data writeToURL:fileURL options:self.config.diskCacheWritingOptions error:nil];
    
    // disable iCloud backup
    if (self.config.shouldDisableiCloud) {
//忽略cloud同步功能
        [fileURL setResourceValue:@YES forKey:NSURLIsExcludedFromBackupKey error:nil];
    }
}

- (void)removeDataForKey:(NSString *)key {
    NSParameterAssert(key);//删除key
    NSString *filePath = [self cachePathForKey:key];
    [self.fileManager removeItemAtPath:filePath error:nil];
}

- (void)removeAllData {//删除所有diskCache
    [self.fileManager removeItemAtPath:self.diskCachePath error:nil];
//创建目录
    [self.fileManager createDirectoryAtPath:self.diskCachePath
            withIntermediateDirectories:YES
                             attributes:nil
                                  error:NULL];
}
//删除过期Data
- (void)removeExpiredData {
    NSURL *diskCacheURL = [NSURL fileURLWithPath:self.diskCachePath isDirectory:YES];
    
    NSURLResourceKey cacheContentDateKey = NSURLContentModificationDateKey;
    switch (self.config.diskCacheExpireType) {
        case SDImageCacheConfigExpireTypeAccessDate:
            cacheContentDateKey = NSURLContentAccessDateKey;
            break;
            
        case SDImageCacheConfigExpireTypeModificationDate:
            cacheContentDateKey = NSURLContentModificationDateKey;
            break;
            
        default:
            break;
    }
    
    NSArray<NSString *> *resourceKeys = @[NSURLIsDirectoryKey, cacheContentDateKey, NSURLTotalFileAllocatedSizeKey];
    
    // This enumerator prefetches useful properties for our cache files.
    NSDirectoryEnumerator *fileEnumerator = [self.fileManager enumeratorAtURL:diskCacheURL
                                               includingPropertiesForKeys:resourceKeys
                                                                  options:NSDirectoryEnumerationSkipsHiddenFiles
                                                             errorHandler:NULL];
    
    NSDate *expirationDate = (self.config.maxDiskAge < 0) ? nil: [NSDate dateWithTimeIntervalSinceNow:-self.config.maxDiskAge];
    NSMutableDictionary<NSURL *, NSDictionary<NSString *, id> *> *cacheFiles = [NSMutableDictionary dictionary];
    NSUInteger currentCacheSize = 0;
    
    // Enumerate all of the files in the cache directory.  This loop has two purposes:
    //
    //  1. Removing files that are older than the expiration date.
    //  2. Storing file attributes for the size-based cleanup pass.
    NSMutableArray<NSURL *> *urlsToDelete = [[NSMutableArray alloc] init];
    for (NSURL *fileURL in fileEnumerator) {
        NSError *error;
        NSDictionary<NSString *, id> *resourceValues = [fileURL resourceValuesForKeys:resourceKeys error:&error];
        
        // Skip directories and errors.
        if (error || !resourceValues || [resourceValues[NSURLIsDirectoryKey] boolValue]) {
            continue;
        }
        
        // Remove files that are older than the expiration date;
        NSDate *modifiedDate = resourceValues[cacheContentDateKey];
        if (expirationDate && [[modifiedDate laterDate:expirationDate] isEqualToDate:expirationDate]) {
            [urlsToDelete addObject:fileURL];
            continue;
        }
        
        // Store a reference to this file and account for its total size.
        NSNumber *totalAllocatedSize = resourceValues[NSURLTotalFileAllocatedSizeKey];
        currentCacheSize += totalAllocatedSize.unsignedIntegerValue;
        cacheFiles[fileURL] = resourceValues;
    }
    
    for (NSURL *fileURL in urlsToDelete) {
        [self.fileManager removeItemAtURL:fileURL error:nil];
    }
    
    // If our remaining disk cache exceeds a configured maximum size, perform a second
    // size-based cleanup pass.  We delete the oldest files first.
    NSUInteger maxDiskSize = self.config.maxDiskSize;
    if (maxDiskSize > 0 && currentCacheSize > maxDiskSize) {
        // Target half of our maximum cache size for this cleanup pass.
        const NSUInteger desiredCacheSize = maxDiskSize / 2;
        
        // Sort the remaining cache files by their last modification time or last access time (oldest first).
        NSArray<NSURL *> *sortedFiles = [cacheFiles keysSortedByValueWithOptions:NSSortConcurrent
                                                                 usingComparator:^NSComparisonResult(id obj1, id obj2) {
                                                                     return [obj1[cacheContentDateKey] compare:obj2[cacheContentDateKey]];
                                                                 }];
        
        // Delete files until we fall below our desired cache size.
        for (NSURL *fileURL in sortedFiles) {
            if ([self.fileManager removeItemAtURL:fileURL error:nil]) {
                NSDictionary<NSString *, id> *resourceValues = cacheFiles[fileURL];
                NSNumber *totalAllocatedSize = resourceValues[NSURLTotalFileAllocatedSizeKey];
                currentCacheSize -= totalAllocatedSize.unsignedIntegerValue;
                
                if (currentCacheSize < desiredCacheSize) {
                    break;
                }
            }
        }
    }
}

- (nullable NSString *)cachePathForKey:(NSString *)key {
    NSParameterAssert(key);
    return [self cachePathForKey:key inPath:self.diskCachePath];
}
//遍历文件 统计文件一共大小
- (NSUInteger)totalSize {
    NSUInteger size = 0;
    NSDirectoryEnumerator *fileEnumerator = [self.fileManager enumeratorAtPath:self.diskCachePath];
    for (NSString *fileName in fileEnumerator) {
        NSString *filePath = [self.diskCachePath stringByAppendingPathComponent:fileName];
        NSDictionary<NSString *, id> *attrs = [self.fileManager attributesOfItemAtPath:filePath error:nil];
        size += [attrs fileSize];
    }
    return size;
}
//遍历统计该目录下文件多少
- (NSUInteger)totalCount {
    NSUInteger count = 0;
    NSDirectoryEnumerator *fileEnumerator = [self.fileManager enumeratorAtPath:self.diskCachePath];
    count = fileEnumerator.allObjects.count;
    return count;
}

#pragma mark - Cache paths

- (nullable NSString *)cachePathForKey:(nullable NSString *)key inPath:(nonnull NSString *)path {
    NSString *filename = SDDiskCacheFileNameForKey(key);
    return [path stringByAppendingPathComponent:filename];
}

- (void)moveCacheDirectoryFromPath:(nonnull NSString *)srcPath toPath:(nonnull NSString *)dstPath {
    NSParameterAssert(srcPath);
    NSParameterAssert(dstPath);
    // Check if old path is equal to new path
    if ([srcPath isEqualToString:dstPath]) {
        return;
    }
    BOOL isDirectory;
    // Check if old path is directory
    if (![self.fileManager fileExistsAtPath:srcPath isDirectory:&isDirectory] || !isDirectory) {
        return;
    }
    // Check if new path is directory
    if (![self.fileManager fileExistsAtPath:dstPath isDirectory:&isDirectory] || !isDirectory) {
        if (!isDirectory) {
            // New path is not directory, remove file
            [self.fileManager removeItemAtPath:dstPath error:nil];
        }
        NSString *dstParentPath = [dstPath stringByDeletingLastPathComponent];
        // Creates any non-existent parent directories as part of creating the directory in path
        if (![self.fileManager fileExistsAtPath:dstParentPath]) {
            [self.fileManager createDirectoryAtPath:dstParentPath withIntermediateDirectories:YES attributes:nil error:NULL];
        }
        // New directory does not exist, rename directory
        [self.fileManager moveItemAtPath:srcPath toPath:dstPath error:nil];
    } else {
        // New directory exist, merge the files
        NSDirectoryEnumerator *dirEnumerator = [self.fileManager enumeratorAtPath:srcPath];
        NSString *file;
        while ((file = [dirEnumerator nextObject])) {
            [self.fileManager moveItemAtPath:[srcPath stringByAppendingPathComponent:file] toPath:[dstPath stringByAppendingPathComponent:file] error:nil];
        }
        // Remove the old path
        [self.fileManager removeItemAtPath:srcPath error:nil];
    }
}

#pragma mark - Hash
#md5 唯一性建立key
#define SD_MAX_FILE_EXTENSION_LENGTH (NAME_MAX - CC_MD5_DIGEST_LENGTH * 2 - 1)

static inline NSString * _Nonnull SDDiskCacheFileNameForKey(NSString * _Nullable key) {
    const char *str = key.UTF8String;
    if (str == NULL) {
        str = "";
    }
    unsigned char r[CC_MD5_DIGEST_LENGTH];
    CC_MD5(str, (CC_LONG)strlen(str), r);
    NSURL *keyURL = [NSURL URLWithString:key];
    NSString *ext = keyURL ? keyURL.pathExtension : key.pathExtension;
    // File system has file name length limit, we need to check if ext is too long, we don't add it to the filename
    if (ext.length > SD_MAX_FILE_EXTENSION_LENGTH) {
        ext = nil;
    }
    NSString *filename = [NSString stringWithFormat:@"%02x%02x%02x%02x%02x%02x%02x%02x%02x%02x%02x%02x%02x%02x%02x%02x%@",
                          r[0], r[1], r[2], r[3], r[4], r[5], r[6], r[7], r[8], r[9], r[10],
                          r[11], r[12], r[13], r[14], r[15], ext.length == 0 ? @"" : [NSString stringWithFormat:@".%@", ext]];
    return filename;
}

@end
```
#### 缓存的基本关键使用
```
- (void)storeImage:(nullable UIImage *)image
         imageData:(nullable NSData *)imageData
            forKey:(nullable NSString *)key
          toMemory:(BOOL)toMemory
            toDisk:(BOOL)toDisk
        completion:(nullable SDWebImageNoParamsBlock)completionBlock {
//没有data or key 直接执行完成函数 return
    if (!image || !key) {
        if (completionBlock) {
            completionBlock();
        }
        return;
    }
    // if memory cache is enabled
    if (toMemory && self.config.shouldCacheImagesInMemory) {
//保存memory cache
        NSUInteger cost = image.sd_memoryCost;
        [self.memCache setObject:image forKey:key cost:cost];
    }
    
    if (toDisk) {
//异步保存到disk
        dispatch_async(self.ioQueue, ^{
//使用Autorepleasepool释放临时变量
            @autoreleasepool {
                NSData *data = imageData;
                if (!data && image) {
                    //检查是否是png 透明格式
                    SDImageFormat format;
                    if ([SDImageCoderHelper CGImageContainsAlpha:image.CGImage]) {
                        format = SDImageFormatPNG;
                    } else {
                        format = SDImageFormatJPEG;
                    }
//格式化image成二进制
                    data = [[SDImageCodersManager sharedManager] encodedDataWithImage:image format:format options:nil];
                }
                [self _storeImageDataToDisk:data forKey:key];
            }
            
            if (completionBlock) {//执行完成回调
                dispatch_async(dispatch_get_main_queue(), ^{
                    completionBlock();
                });
            }
        });
    } else {//如果不写入 disk 直接回调block
        if (completionBlock) {
            completionBlock();
        }
    }
}
```
总结：继承NSCache可以拥有更多的话语权，NSCache释放规则不受自己控制，这样子可以自己使用MapTable另外缓存一份数据，使用空间换时间的思想提高cache命中率，异步处理disk写入，提升体验；读data的时候加入到memory中，提高再次读取的命中率。


