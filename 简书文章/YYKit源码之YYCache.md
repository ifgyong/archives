#### YYCache源码解读

> 大致思路是YYMemoryCache是使用的双向链表+CFDictionaryCreateMutable,查找复杂度O(1),大量使用信号量实现读写的安全机制；个人定制化强，不受系统影响；而且不限object的类型，为泛型id。优点多多

```
@interface YYMemoryCache : NSObject

#pragma mark - Attribute
///=============================================================================
/// @name Attribute
///=============================================================================

//cache的name
@property (nullable, copy) NSString *name;

//cache的一共的个数 (read-only) 
@property (readonly) NSUInteger totalCount;

//cache的一共占用内存大小
@property (readonly) NSUInteger totalCost;


#pragma mark - Limit
///=============================================================================
/// @name Limit
///=============================================================================

//cache的个数
@property NSUInteger countLimit;

//cache的缓存大小空间
@property NSUInteger costLimit;


@property NSTimeInterval ageLimit;

//自动检查更新缓存的时间
@property NSTimeInterval autoTrimInterval;

//接收内存警告的时候 释放所有缓存 默认YES
@property BOOL shouldRemoveAllObjectsOnMemoryWarning;

//进入到后台的时候 释放所有缓存 默认YES
@property BOOL shouldRemoveAllObjectsWhenEnteringBackground;

//接收内存警告的时候 执行的block
@property (nullable, copy) void(^didReceiveMemoryWarningBlock)(YYMemoryCache *cache);

//进入后台的时候 执行的block
@property (nullable, copy) void(^didEnterBackgroundBlock)(YYMemoryCache *cache);

//是否在mainThread 释放 默认NO
@property BOOL releaseOnMainThread;

//是否异步释放缓存 默认YES
@property BOOL releaseAsynchronously;


#pragma mark - Access Methods
///=============================================================================
/// @name Access Methods
///=============================================================================

//是否包含key
- (BOOL)containsObjectForKey:(id)key;

//返回该key的 obj
- (nullable id)objectForKey:(id)key;

//设置obj 
- (void)setObject:(nullable id)object forKey:(id)key;

//设置key为obj 占用内存大小
- (void)setObject:(nullable id)object forKey:(id)key withCost:(NSUInteger)cost;

//删除key的obj
- (void)removeObjectForKey:(id)key;

//删除所有obj
- (void)removeAllObjects;


#pragma mark - Trim
///=============================================================================
/// @name Trim
///=============================================================================

//设置更新缓存后的个数
- (void)trimToCount:(NSUInteger)count;

//设置更新缓存后的大小
- (void)trimToCost:(NSUInteger)cost;

//设置更新缓存的age的临界值
- (void)trimToAge:(NSTimeInterval)age;

@end

@end
```
#### 关键函数注释获取内存大小和个数 没有循环遍历大小，直接返回值，因为都是动态计算的；
```
//信号量读数据加锁 使用CFDictionaryContainsKey c函数返回其值。
- (BOOL)containsObjectForKey:(id)key {
    if (!key) return NO;
    pthread_mutex_lock(&_lock);
    BOOL contains = CFDictionaryContainsKey(_lru->_dic, (__bridge const void *)(key));
    pthread_mutex_unlock(&_lock);
    return contains;
}
//先在CFDictionaryGetValue 获取node的地址
- (id)objectForKey:(id)key {
    if (!key) return nil;
    pthread_mutex_lock(&_lock);
    _YYLinkedMapNode *node = CFDictionaryGetValue(_lru->_dic, (__bridge const void *)(key));
    if (node) {
//更新访问时间
        node->_time = CACurrentMediaTime();
//更新node的节点位置到head，删除的时候是在最后删除
        [_lru bringNodeToHead:node];
    }
    pthread_mutex_unlock(&_lock);
    return node ? node->_value : nil;
}

- (void)setObject:(id)object forKey:(id)key {
    [self setObject:object forKey:key withCost:0];
}

- (void)setObject:(id)object forKey:(id)key withCost:(NSUInteger)cost {
    if (!key) return;
    if (!object) {
        [self removeObjectForKey:key];
        return;
    }
//信号量加锁
    pthread_mutex_lock(&_lock);
//获取node
    _YYLinkedMapNode *node = CFDictionaryGetValue(_lru->_dic, (__bridge const void *)(key));
    NSTimeInterval now = CACurrentMediaTime();
    if (node) {
        _lru->_totalCost -= node->_cost;
        _lru->_totalCost += cost;
        node->_cost = cost;
        node->_time = now;
        node->_value = object;
//更新node到head
        [_lru bringNodeToHead:node];
    } else {
        node = [_YYLinkedMapNode new];
        node->_cost = cost;
        node->_time = now;
        node->_key = key;
        node->_value = object;
//更新node到head

        [_lru insertNodeAtHead:node];
    }
    if (_lru->_totalCost > _costLimit) {
        dispatch_async(_queue, ^{
            [self trimToCost:_costLimit];
        });
    }
    if (_lru->_totalCount > _countLimit) {
        _YYLinkedMapNode *node = [_lru removeTailNode];
        if (_lru->_releaseAsynchronously) {
            dispatch_queue_t queue = _lru->_releaseOnMainThread ? dispatch_get_main_queue() : YYMemoryCacheGetReleaseQueue();
            dispatch_async(queue, ^{
                [node class]; //hold and release in queue
            });
        } else if (_lru->_releaseOnMainThread && !pthread_main_np()) {
            dispatch_async(dispatch_get_main_queue(), ^{
                [node class]; //hold and release in queue
            });
        }
    }//解锁
    pthread_mutex_unlock(&_lock);
}
//删除key value
- (void)removeObjectForKey:(id)key {
    if (!key) return;
    pthread_mutex_lock(&_lock);
    _YYLinkedMapNode *node = CFDictionaryGetValue(_lru->_dic, (__bridge const void *)(key));
    if (node) {
//node 删除
        [_lru removeNode:node];
        if (_lru->_releaseAsynchronously) {
            dispatch_queue_t queue = _lru->_releaseOnMainThread ? dispatch_get_main_queue() : YYMemoryCacheGetReleaseQueue();
            dispatch_async(queue, ^{
                [node class]; //hold and release in queue
            });
        } else if (_lru->_releaseOnMainThread && !pthread_main_np()) {
            dispatch_async(dispatch_get_main_queue(), ^{
                [node class]; //hold and release in queue
            });
        }
    }
    pthread_mutex_unlock(&_lock);
}

- (void)removeAllObjects {
//删除all obj 加锁删除
    pthread_mutex_lock(&_lock);
    [_lru removeAll];
    pthread_mutex_unlock(&_lock);
}
```


#### YYDiskCache源码解读

```
@interface YYDiskCache : NSObject

#pragma mark - Attribute
///=============================================================================
/// @name Attribute
///=============================================================================

/** cache的名字 */
@property (nullable, copy) NSString *name;

/** cache的路径 (read-only). */
@property (readonly) NSString *path;

/**
 SQLite的文件大小 默认20kb.
 */
@property (readonly) NSUInteger inlineThreshold;

/**
 没有实现`NSCoding` protocol的可以使用此block代替
 The default value is nil.
 */
@property (nullable, copy) NSData *(^customArchiveBlock)(id object);

/**
 没有实现`NSCoding` protocol的可以使用此block代替
 
 The default value is nil.
 */
@property (nullable, copy) id (^customUnarchiveBlock)(NSData *data);

/**
cache的加密方式 默认是md5
 
 The default value is nil.
 */
@property (nullable, copy) NSString *(^customFileNameBlock)(NSString *key);



#pragma mark - Limit
///=============================================================================
/// @name Limit
///=============================================================================

/**
 The maximum number of objects the cache should hold.
 
 @discussion The default value is NSUIntegerMax, which means no limit.
 This is not a strict limit — if the cache goes over the limit, some objects in the
 cache could be evicted later in background queue.
 */
@property NSUInteger countLimit;

/**
 The maximum total cost that the cache can hold before it starts evicting objects.
 
 @discussion The default value is NSUIntegerMax, which means no limit.
 This is not a strict limit — if the cache goes over the limit, some objects in the
 cache could be evicted later in background queue.
 */
@property NSUInteger costLimit;

/**
 The maximum expiry time of objects in cache.
 
 @discussion The default value is DBL_MAX, which means no limit.
 This is not a strict limit — if an object goes over the limit, the objects could
 be evicted later in background queue.
 */
@property NSTimeInterval ageLimit;

//缓存应保留的最小空间
@property NSUInteger freeDiskSpaceLimit;

//更新缓存的频率 默认60s
@property NSTimeInterval autoTrimInterval;

//是否开启日期功能 在DEBUG环境下
@property BOOL errorLogsEnabled;

#pragma mark - Initializer
///=============================================================================
/// @name Initializer
///=============================================================================
//禁用init 和new函数
- (instancetype)init UNAVAILABLE_ATTRIBUTE;
+ (instancetype)new UNAVAILABLE_ATTRIBUTE;

//创建一个基础的路径保存cache
- (nullable instancetype)initWithPath:(NSString *)path;
//保存完整的路径到文件，如果文件大于threshold
- (nullable instancetype)initWithPath:(NSString *)path
                      inlineThreshold:(NSUInteger)threshold NS_DESIGNATED_INITIALIZER;


#pragma mark - Access Methods
///=============================================================================
/// @name Access Methods
///=============================================================================

//是否包含key
- (BOOL)containsObjectForKey:(NSString *)key;
//是否包含该key
- (void)containsObjectForKey:(NSString *)key withBlock:(void(^)(NSString *key, BOOL contains))block;

- (nullable id<NSCoding>)objectForKey:(NSString *)key;

- (void)objectForKey:(NSString *)key withBlock:(void(^)(NSString *key, id<NSCoding> _Nullable object))block;

//设置key 保存obj
- (void)setObject:(nullable id<NSCoding>)object forKey:(NSString *)key;

//如果 obj 是nil 直接删除key，
- (void)setObject:(nullable id<NSCoding>)object forKey:(NSString *)key withBlock:(void(^)(void))block;

//删除该key
- (void)removeObjectForKey:(NSString *)key;

//删除该 key 
- (void)removeObjectForKey:(NSString *)key withBlock:(void(^)(NSString *key))block;

//删除allObjects
- (void)removeAllObjects;

//删除allObjects
- (void)removeAllObjectsWithBlock:(void(^)(void))block;

//删除obj 和回调block
- (void)removeAllObjectsWithProgressBlock:(nullable void(^)(int removedCount, int totalCount))progress
                                 endBlock:(nullable void(^)(BOOL error))end;


//所有大小
- (NSInteger)totalCount;
- (void)totalCountWithBlock:(void(^)(NSInteger totalCount))block;

//占用空间大小
- (NSInteger)totalCost;
- (void)totalCostWithBlock:(void(^)(NSInteger totalCost))block;


#pragma mark - Trim
///=============================================================================
/// @name Trim
///=============================================================================


- (void)trimToCount:(NSUInteger)count;


- (void)trimToCount:(NSUInteger)count withBlock:(void(^)(void))block;


- (void)trimToCost:(NSUInteger)cost;


- (void)trimToCost:(NSUInteger)cost withBlock:(void(^)(void))block;


- (void)trimToAge:(NSTimeInterval)age;


- (void)trimToAge:(NSTimeInterval)age withBlock:(void(^)(void))block;


#pragma mark - Extended Data
///=============================================================================
/// @name Extended Data
///=============================================================================


+ (nullable NSData *)getExtendedDataFromObject:(id)object;


+ (void)setExtendedData:(nullable NSData *)extendedData toObject:(id)object;

@end

```
#### 重要的是简历SQLite的初始化
> create index if not exists last_access_time_idx on manifest(last_access_time) 使用last_access_time创建索引和`primary key(key)`，缩短select的时间。
每次YYDisk读取的data都会加入到YYMemoryCache中，这个和runtime的method的cache原理一样的，下次访问该data的时候首先进行memory查找，没有的话再到disk中查找，增加memory的命中率，提高效率。
```
- (BOOL)_dbInitialize {
    NSString *sql = @"pragma journal_mode = wal; pragma synchronous = normal; create table if not exists manifest (key text, filename text, size integer, inline_data blob, modification_time integer, last_access_time integer, extended_data blob, primary key(key)); create index if not exists last_access_time_idx on manifest(last_access_time);";
    return [self _dbExecute:sql];
}
```
