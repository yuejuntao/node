[TOC]

#### 一  前言

Google 出的 [Guava](https://github.com/google/guava) 是 Java 核心增强的库，应用非常广泛。

缓存在日常开发中举足轻重，如果你的应用对某类数据有着较高的读取频次，并且改动较小时那就非常适合利用缓存来提高性能。缓存之所以可以提高性能是因为它的读取效率很高，就像是 CPU 的 `L1、L2、L3` 缓存一样，级别越高相应的读取速度也会越快。

但也不是什么好处都占，读取速度快了但是它的内存更小资源更宝贵，所以我们应当缓存真正需要的数据。

> 其实也就是典型的空间换时间。

下面谈谈 Java 中所用到的缓存。

##### 1  JVM 缓存

其实就是创建一些全局变量，如 `Map、List` 之类的容器用于存放数据。

这样的优势是使用简单但是也有以下问题：

- 只能显式的写入，清除数据。
- 不能按照一定的规则淘汰数据，如 `LRU，LFU，FIFO` 等。
- 清除数据时的回调通知。
- 其他一些定制功能等。

##### 2  Ehcache、Guava Cache

所以出现了一些专门用作 JVM 缓存的开源工具出现了，如本文提到的 Guava Cache。它具有上文 JVM 缓存不具有的功能，如自动清除数据、多种清除算法、清除回调等。

但也正因为有了这些功能，这样的缓存必然会多出许多东西需要额外维护，自然也就增加了系统的消耗。

##### 3  分布式缓存

刚才提到的两种缓存其实都是堆内缓存，只能在单个节点中使用，这样在分布式场景下就招架不住了。于是也有了一些缓存中间件，如 Redis、Memcached，在分布式环境下可以共享内存。

#### 二  Guava Cache使用示例

##### 1  缓存失效策略

CacheBuilder有3种失效重载模式：

（1）expireAfterWrite

当**创建**或**写**之后的固定有效期到达时，数据会被自动从缓存中移除，源码注释如下：

````java
/**
   * 指明每个数据实体：当 创建 或 最新一次更新 之后的 固定值的 有效期到达时，数据会被自动从缓存中移除
   * Specifies that each entry should be automatically removed from the cache once a fixed duration
   * has elapsed after the entry's creation, or the most recent replacement of its value.
   * 当间隔被设置为0时，maximumSize设置为0，忽略其它容量和权重的设置。这使得测试时 临时性地 禁用缓存且不用改代码。
   * <p>When {@code duration} is zero, this method hands off to
   * {@link #maximumSize(long) maximumSize}{@code (0)}, ignoring any otherwise-specificed maximum
   * size or weight. This can be useful in testing, or to disable caching temporarily without a code
   * change.
   * 过期的数据实体可能会被Cache.size统计到，但不能进行读写，数据过期后会被清除。
   * <p>Expired entries may be counted in {@link Cache#size}, but will never be visible to read or
   * write operations. Expired entries are cleaned up as part of the routine maintenance described
   * in the class javadoc.
   *
   * @param duration the length of time after an entry is created that it should be automatically
   *     removed
   * @param unit the unit that {@code duration} is expressed in
   * @throws IllegalArgumentException if {@code duration} is negative
   * @throws IllegalStateException if the time to live or time to idle was already set
   */
  public CacheBuilder<K, V> expireAfterWrite(long duration, TimeUnit unit) {
    checkState(expireAfterWriteNanos == UNSET_INT, "expireAfterWrite was already set to %s ns",
        expireAfterWriteNanos);
    checkArgument(duration >= 0, "duration cannot be negative: %s %s", duration, unit);
    this.expireAfterWriteNanos = unit.toNanos(duration);
    return this;
  }
````

（2）expireAfterAccess

指明每个数据实体：当**创建**或**写**或**读之后**的固定值的有效期到达时，数据会被自动从缓存中移除。读写操作都会重置访问时间，但asMap方法不会。源码注释如下：

````java
 /**
    * 指明每个数据实体：当 创建 或 更新 或 访问 之后的 固定值的有效期到达时，数据会被自动从缓存中移除。读写操作都会重置访问时间，但asMap方法不会。
   * Specifies that each entry should be automatically removed from the cache once a fixed duration
   * has elapsed after the entry's creation, the most recent replacement of its value, or its last
   * access. Access time is reset by all cache read and write operations (including
   * {@code Cache.asMap().get(Object)} and {@code Cache.asMap().put(K, V)}), but not by operations
   * on the collection-views of {@link Cache#asMap}.
   *
   * <p>When {@code duration} is zero, this method hands off to
   * {@link #maximumSize(long) maximumSize}{@code (0)}, ignoring any otherwise-specificed maximum
   * size or weight. This can be useful in testing, or to disable caching temporarily without a code
   * change.
   *
   * <p>Expired entries may be counted in {@link Cache#size}, but will never be visible to read or
   * write operations. Expired entries are cleaned up as part of the routine maintenance described
   * in the class javadoc.
   *
   * @param duration the length of time after an entry is last accessed that it should be
   *     automatically removed
   * @param unit the unit that {@code duration} is expressed in
   * @throws IllegalArgumentException if {@code duration} is negative
   * @throws IllegalStateException if the time to idle or time to live was already set
   */
  public CacheBuilder<K, V> expireAfterAccess(long duration, TimeUnit unit) {
    checkState(expireAfterAccessNanos == UNSET_INT, "expireAfterAccess was already set to %s ns",
        expireAfterAccessNanos);
    checkArgument(duration >= 0, "duration cannot be negative: %s %s", duration, unit);
    this.expireAfterAccessNanos = unit.toNanos(duration);
    return this;
  }
````

（3）refreshAfterWrite

指明每个数据实体：当**创建**或**写**之后的 固定值的有效期到达时，**且新请求过来时**，数据会被自动刷新（**注意不是删除是异步刷新，不会阻塞读取，先返回旧值，异步重载到数据返回后复写新值**）。源码注释如下：

````java
/**
   * 指明每个数据实体：当 创建 或 更新 之后的 固定值的有效期到达时，数据会被自动刷新。刷新方法在LoadingCache接口的refresh()申明，实际最终调用的是CacheLoader的reload()
   * Specifies that active entries are eligible for automatic refresh once a fixed duration has
   * elapsed after the entry's creation, or the most recent replacement of its value. The semantics
   * of refreshes are specified in {@link LoadingCache#refresh}, and are performed by calling
   * {@link CacheLoader#reload}.
   * 默认reload是同步方法，所以建议用户覆盖reload方法，否则刷新将在无关的读写操作间操作。
   * <p>As the default implementation of {@link CacheLoader#reload} is synchronous, it is
   * recommended that users of this method override {@link CacheLoader#reload} with an asynchronous
   * implementation; otherwise refreshes will be performed during unrelated cache read and write
   * operations.
   *
   * <p>Currently automatic refreshes are performed when the first stale request for an entry
   * occurs. The request triggering refresh will make a blocking call to {@link CacheLoader#reload}
   * and immediately return the new value if the returned future is complete, and the old value
   * otherwise.
   * 触发刷新操作的请求会阻塞调用reload方法并且当返回的Future完成时立即返回新值，否则返回旧值。
   * <p><b>Note:</b> <i>all exceptions thrown during refresh will be logged and then swallowed</i>.
   *
   * @param duration the length of time after an entry is created that it should be considered
   *     stale, and thus eligible for refresh
   * @param unit the unit that {@code duration} is expressed in
   * @throws IllegalArgumentException if {@code duration} is negative
   * @throws IllegalStateException if the refresh interval was already set
   * @since 11.0
   */
  @Beta
  @GwtIncompatible("To be supported (synchronously).")
  public CacheBuilder<K, V> refreshAfterWrite(long duration, TimeUnit unit) {
    checkNotNull(unit);
    checkState(refreshNanos == UNSET_INT, "refresh was already set to %s ns", refreshNanos);
    checkArgument(duration > 0, "duration must be positive: %s %s", duration, unit);
    this.refreshNanos = unit.toNano(duration);
    return this;
  }
````

##### 2  使用示例

````java
public class CommonGuavaCache {
    public static final Logger logger = LoggerFactory.getLogger(CommonGuavaCache.class);

    // 保存key和接口函数
    private static Map<String, Supplier<Object>> supplierMap = Maps.newConcurrentMap();
    /**
     * 创建cache类
	 * 1.expireAfterWrite:指定时间内没有创建/覆盖时，会移除该key，下次取的时候触发"同步load"(一个线程执行load)
	 * 2.refreshAfterWrite:指定时间内没有被创建/覆盖，则指定时间过后，再次访问时，会去刷新该缓存，在新值没有到来之前，始终返回旧值
	 * "异步reload"（也是一个线程执行reload）
	 * 3.expireAfterAccess:指定时间内没有读写，会移除该key，下次取的时候从loading中取
	 * 区别：指定时间过后，expire是remove该key，下次访问是同步去获取返回新值；
	 * 而refresh则是指定时间后，不会remove该key，下次访问会触发刷新，新值没有回来时返回旧值
	 * 同时使用：可避免定时刷新+定时删除下次访问载入
	 */
    private static LoadingCache<String, Object> cache = CacheBuilder.newBuilder()
            //.refreshAfterWrite(1, TimeUnit.SECONDS)
 		   .expireAfterWrite(1, TimeUnit.SECONDS)
 		   //.expireAfterAccess(1,TimeUnit.SECONDS)
            .initialCapacity(10)
        	.maximumSize(1000)
        	.build(new CacheLoader<String, Object>() {
                @Override
                public Object load(String key) throws Exception {
                    // 获取对应的接口函数
                    Supplier<Object> supplier = supplierMap.get(key);
                    if (supplier == null) {
                        return null;
                    }
                    // 执行接口函数
                    Object obj = supplier.get();
                    // 返回json格式的结果
                    return JSON.toJSONString(obj);
                }
            });


    // 注册接口函数
    public static <T> void registerSupplier(String key, Supplier<T> supplier) {
        supplierMap.put(key, (Supplier<Object>) supplier);
    }

    // 对外提供的获取缓存数据接口
    public static <T> T getValue(String key, TypeReference<T> typeReference) {
        try {
            String value = (String) cache.get(key);
            return JSON.parseObject(value, typeReference);
        } catch (Exception e) {
            logger.error("", e);
        }
        return null;
    }
}
````

调用方式:

````java
	// 初始化时将接口函数注册到GuavaCache中
	@PostConstruct
    private void init() {
        CommonGuavaCache.registerSupplier(ADS_EXT_PARAMS_CONFIG, this::getAdsExtParamConfig);
    }
    public AdsExtParamConfig getAdsExtParamConfig() {
        AdsExtParamConfig adsExtParamConfig = cmsContentProxy.getValueByContentType(ADS_EXT_PARAMS_CONTENT_TYPE, null, new TypeReference<AdsExtParamConfig>() {
        });
        if(adsExtParamConfig==null){
            adsExtParamConfig = new AdsExtParamConfig();
        }
        return adsExtParamConfig;
    }
````

#### 三  源码解析

##### 1  数据结构

先看一下google cache 核心类如下：

- CacheBuilder：类，缓存构建器。构建缓存的入口，指定缓存配置参数并初始化本地缓存。

  CacheBuilder在build方法中，会把前面设置的参数，全部传递给LocalCache，它自己实际不参与任何计算。采用构造器模式（Builder）使得初始化参数的方法值得借鉴，代码简洁易读。

- CacheLoader：抽象类。用于从数据源加载数据，定义load、reload、loadAll等操作。

- Cache：接口，定义get、put、invalidate等操作，这里只有缓存增删改的操作，没有数据加载的操作。

- LoadingCache：接口，继承自Cache。定义get、getUnchecked、getAll等操作，这些操作都会从数据源load数据。

- LocalCache：类。整个guava cache的核心类，包含了guava cache的数据结构以及基本的缓存的操作方法。

- LocalManualCache：LocalCache内部静态类，实现Cache接口。其内部的增删改缓存操作全部调用成员变量localCache（LocalCache类型）的相应方法。

- LocalLoadingCache：LocalCache内部静态类，继承自LocalManualCache类，实现LoadingCache接口。其所有操作也是调用成员变量localCache（LocalCache类型）的相应方法。

##### 2  LocalCache的数据结构图

<img src="https://raw.githubusercontent.com/yuejuntao/typoraImg/master/img/image-20201230182312941.png" alt="image-20201230182312941" style="zoom: 33%;" />

LocalCache类似ConcurrentHashMap采用了分段策略，通过减小锁的粒度来提高并发，LocalCache中数据存储在Segment[]中，每个segment又包含5个队列和一个table,这个table是自定义的一种类数组的结构，每个元素都包含一个ReferenceEntry<k,v>链表，指向next entry。

这些队列，前2个是key、value引用队列用以加速GC回收，后3个队列记录用户的写记录、访问记录、高频访问顺序队列用以实现LRU算法。

AtomicReferenceArray是JUC包下的*Doug Lea*老李头设计的类：一组对象引用，其中元素支持原子性更新。

最后是ReferenceEntry：引用数据存储接口，默认强引用

##### 3  源码解析

CacheBuilder参数设置完毕后最后调用build（CacheLoader ）构造，参数是用户自定义的CacheLoader缓存加载器，复写一些方法（load,reload），返回LoadingCache接口（一种面向接口编程的思想，实际返回具体实现类）如下图：

````java
public <K1 extends K, V1 extends V> LoadingCache<K1, V1> build(CacheLoader<? super K1, V1> loader) {
    checkWeightWithWeigher();
    return new LocalCache.LocalLoadingCache<K1, V1>(this, loader);
  }
````

实际是构造了一个LoadingCache接口的实现类：LocalCache的静态类LocalLoadingCache，本地加载缓存类。

````java
	LocalLoadingCache(CacheBuilder<? super K, ? super V> builder, CacheLoader<? super K, V> loader) {
      super(new LocalCache<K, V>(builder, checkNotNull(loader)));
    }
````

LocalLoadingCache的构造函数中构造函数需要一个LocalCache作为参数：

````java
LocalCache(CacheBuilder<? super K, ? super V> builder, @Nullable CacheLoader<? super K, V> loader) {
    //默认并发水平是4
    concurrencyLevel = Math.min(builder.getConcurrencyLevel(), MAX_SEGMENTS);
    //key\value的强引用
    keyStrength = builder.getKeyStrength();
    valueStrength = builder.getValueStrength();
	//key\value比较器
    keyEquivalence = builder.getKeyEquivalence();
    valueEquivalence = builder.getValueEquivalence();

    maxWeight = builder.getMaximumWeight();
    weigher = builder.getWeigher();
    //读写后有效期，超时重载
    expireAfterAccessNanos = builder.getExpireAfterAccessNanos();
    //写后有效期，超时重载
    expireAfterWriteNanos = builder.getExpireAfterWriteNanos();
    refreshNanos = builder.getRefreshNanos();

    //缓存触发失效 或者 GC回收软/弱引用，触发监听器
    removalListener = builder.getRemovalListener();
    //移除通知队列
    removalNotificationQueue = (removalListener == NullListener.INSTANCE)
        ? LocalCache.<RemovalNotification<K, V>>discardingQueue()
        : new ConcurrentLinkedQueue<RemovalNotification<K, V>>();

    ticker = builder.getTicker(recordsTime());
    entryFactory = EntryFactory.getFactory(keyStrength, usesAccessEntries(), usesWriteEntries());
    globalStatsCounter = builder.getStatsCounterSupplier().get();
    //缓存加载器
    defaultLoader = loader;

    int initialCapacity = Math.min(builder.getInitialCapacity(), MAXIMUM_CAPACITY);
    if (evictsBySize() && !customWeigher()) {
      initialCapacity = Math.min(initialCapacity, (int) maxWeight);
    }

    int segmentShift = 0;
    int segmentCount = 1;
    while (segmentCount < concurrencyLevel
           && (!evictsBySize() || segmentCount * 20 <= maxWeight)) {
      ++segmentShift;
      segmentCount <<= 1;
    }
    this.segmentShift = 32 - segmentShift;
    segmentMask = segmentCount - 1;

    this.segments = newSegmentArray(segmentCount);

    int segmentCapacity = initialCapacity / segmentCount;
    if (segmentCapacity * segmentCount < initialCapacity) {
      ++segmentCapacity;
    }

    int segmentSize = 1;
    while (segmentSize < segmentCapacity) {
      segmentSize <<= 1;
    }

    if (evictsBySize()) {
      // Ensure sum of segment max weights = overall max weights
      long maxSegmentWeight = maxWeight / segmentCount + 1;
      long remainder = maxWeight % segmentCount;
      for (int i = 0; i < this.segments.length; ++i) {
        if (i == remainder) {
          maxSegmentWeight--;
        }
        this.segments[i] =
            createSegment(segmentSize, maxSegmentWeight, builder.getStatsCounterSupplier().get());
      }
    } else {
      for (int i = 0; i < this.segments.length; ++i) {
        this.segments[i] =
            createSegment(segmentSize, UNSET_INT, builder.getStatsCounterSupplier().get());
      }
    }
  }
````

数据过期不会自动重载，而是通过get操作时执行过期重载。具体就是上面追踪到了CacheBuilder构造的LocalLoadingCache，返回LocalCache.LocalLoadingCache后就可以调用如下方法：

````java
static class LocalLoadingCache<K, V> extends LocalManualCache<K, V> implements LoadingCache<K, V> {
    LocalLoadingCache(CacheBuilder<? super K, ? super V> builder, CacheLoader<? super K, V> loader) {
      super(new LocalCache<K, V>(builder, checkNotNull(loader)));
    }

    @Override
    public V get(K key) throws ExecutionException {
      return localCache.getOrLoad(key);
    }

    @Override
    public V getUnchecked(K key) {
      try {
        return get(key);
      } catch (ExecutionException e) {
        throw new UncheckedExecutionException(e.getCause());
      }
    }

    @Override
    public ImmutableMap<K, V> getAll(Iterable<? extends K> keys) throws ExecutionException {
      return localCache.getAll(keys);
    }

    @Override
    public void refresh(K key) {
      localCache.refresh(key);
    }

    @Override
    public final V apply(K key) {
      return getUnchecked(key);
    }

    private static final long serialVersionUID = 1;

    @Override
    Object writeReplace() {
      return new LoadingSerializationProxy<K, V>(localCache);
    }
  }
````

最终get方法调用如下：

````java
	@Override
    public V get(K key) throws ExecutionException {
      return localCache.getOrLoad(key);
    }
````

````java
V getOrLoad(K key) throws ExecutionException {
    return get(key, defaultLoader);
  }
````

````java
V get(K key, CacheLoader<? super K, V> loader) throws ExecutionException {
    int hash = hash(checkNotNull(key));
    return segmentFor(hash).get(key, hash, loader);
  }
````

````java
V get(K key, int hash, CacheLoader<? super K, V> loader) throws ExecutionException {
      checkNotNull(key);
      checkNotNull(loader);
      try {
         // 读volatile 当前段的元素个数，如果存在元素
		if (count != 0) { // read-volatile
			// don't call getLiveEntry, which would ignore loading values
			ReferenceEntry<K, V> e = getEntry(key, hash);
			if (e != null) {
				long now = map.ticker.read();
				V value = getLiveValue(e, now);
				if (value != null) {
                      //记录访问时间，并添加进最近使用（LRU）队列
					recordRead(e, now);
                      //命中缓存，基数+1
					statsCounter.recordHits(1);
                      //刷新值并返回
					return scheduleRefresh(e, key, hash, value, now, loader);
				}
				ValueReference<K, V> valueReference = e.getValueReference();
                 //如果正在重载数据，等待重载完毕后返回值
				if (valueReference.isLoading()) {
					return waitForLoadingValue(e, key, valueReference);
				}
			}
		}
         // 当前segment中找不到实体
		return lockedGetOrLoad(key, hash, loader);
	} catch (ExecutionException ee) {
		Throwable cause = ee.getCause();
		if (cause instanceof Error) {
			throw new ExecutionError((Error) cause);
		} else if (cause instanceof RuntimeException) {
			throw new UncheckedExecutionException(cause);
		}
		throw ee;
	} finally {
		postReadCleanup();
	}
}
````

````java
V scheduleRefresh(ReferenceEntry<K, V> entry, K key, int hash, V oldValue, long now, CacheLoader<? super K, V> loader) {
	if (map.refreshes() && (now - entry.getWriteTime() > map.refreshNanos) && !entry.getValueReference().isLoading()) {
         //重载数据
		V newValue = refresh(key, hash, loader, true);
         //重载数据成功，直接返回
		if (newValue != null) {
			return newValue;
		}
	}
    //否则返回旧值
	return oldValue;
}
````

最终刷新调用的是CacheBuilder中预先设置好的CacheLoader接口实现类的reload方法实现的异步刷新。

