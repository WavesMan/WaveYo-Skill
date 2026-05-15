# Multi-Level Cache Pattern — WaveYo 风格

> 触发条件：需要为后端服务引入缓存优化时加载。
> 来源证据：YoOSF-API（AcquireLock 模式）、Koishi-Mirror（Redis semaphore）。

---

## 缓存分层架构

```
请求
  │
  ├─ Layer 1: 本地 LRU/LFU (内存，纳秒级)
  │   命中 → 返回
  │   未命中 ↓
  │
  ├─ Layer 2: Redis (网络，微秒级)
  │   命中 → 异步回填 L1 → 返回
  │   未命中 ↓
  │
  └─ Layer 3: 源站 / DB / API (毫秒级)
      获取 → 异步回填 Redis + L1 → 返回
```

---

## 本地缓存（Layer 1）

```go
type LocalCache struct {
    shards []*cacheShard  // 分片减少锁竞争
    shardMask uint32
}

type cacheShard struct {
    mu   sync.RWMutex
    data map[string]*cacheEntry
    lru  *lruList
}

type cacheEntry struct {
    value      interface{}
    expiresAt  time.Time
    lruNode    *list.Element
}

func NewLocalCache(shardCount int, maxPerShard int, ttl time.Duration) *LocalCache {
    shards := make([]*cacheShard, shardCount)
    for i := range shards {
        shards[i] = &cacheShard{
            data: make(map[string]*cacheEntry),
        }
    }
    return &LocalCache{
        shards:    shards,
        shardMask: uint32(shardCount - 1),
    }
}

func (c *LocalCache) Get(key string) (interface{}, bool) {
    shard := c.shards[hash(key)&c.shardMask]
    shard.mu.RLock()
    defer shard.mu.RUnlock()
    entry, ok := shard.data[key]
    if !ok || time.Now().After(entry.expiresAt) {
        return nil, false
    }
    return entry.value, true
}
```

---

## Redis 缓存（Layer 2）+ 异步刷新

```go
type RedisCache struct {
    client    *redis.Client
    local     *LocalCache
    ttl       time.Duration
    semaphore chan struct{}  // 控制并发刷新数
}

func (c *RedisCache) Get(ctx context.Context, key string) (interface{}, error) {
    // 1. 先查本地
    if val, ok := c.local.Get(key); ok {
        metrics.CacheHit("l1", key)
        return val, nil
    }

    // 2. 查 Redis
    val, err := c.client.Get(ctx, key).Result()
    if err == nil {
        metrics.CacheHit("redis", key)
        // 异步回填本地
        go c.backfillLocal(key, val)
        return val, nil
    }
    metrics.CacheMiss(key)
    return nil, ErrCacheMiss
}

func (c *RedisCache) Set(ctx context.Context, key string, value interface{}) error {
    // 先写 Redis
    if err := c.client.Set(ctx, key, value, c.ttl).Err(); err != nil {
        return err
    }
    // 再写本地（同步，本地写很快）
    c.local.Set(key, value, c.ttl/2)  // 本地 TTL 更短
    return nil
}

// backfillLocal 异步回填本地缓存，用 semaphore 控制并发
func (c *RedisCache) backfillLocal(key string, value interface{}) {
    c.semaphore <- struct{}{}
    defer func() { <-c.semaphore }()
    c.local.Set(key, value, c.ttl/2)
}
```

---

## 缓存预热

```go
func (c *RedisCache) Warmup(ctx context.Context, keys []string) error {
    var wg sync.WaitGroup
    errCh := make(chan error, len(keys))

    for _, key := range keys {
        wg.Add(1)
        go func(k string) {
            defer wg.Done()
            val, err := c.client.Get(ctx, k).Result()
            if err == nil {
                c.local.Set(k, val, c.ttl/2)
            }
            if err != nil && err != redis.Nil {
                errCh <- err
            }
        }(key)
    }
    wg.Wait()
    close(errCh)

    var errs []error
    for e := range errCh {
        errs = append(errs, e)
    }
    if len(errs) > 0 {
        return fmt.Errorf("warmup errors: %v", errs)
    }
    return nil
}
```

---

## 指标暴露

```go
var (
    cacheHits = prometheus.NewCounterVec(
        prometheus.CounterOpts{Name: "cache_hits_total"},
        []string{"layer"},
    )
    cacheMisses = prometheus.NewCounter(
        prometheus.CounterOpts{Name: "cache_misses_total"},
    )
)

func RecordHit(layer string) {
    cacheHits.WithLabelValues(layer).Inc()
}

func RecordMiss() {
    cacheMisses.Inc()
}
```

---

## TTL 策略

- 本地缓存 TTL < Redis TTL（通常本地 = Redis/2）
- Redis TTL < 源站数据更新周期
- 预签名 URL：小文件 1h，大文件 24h
- 文件列表：根据更新频率，通常 5-30 分钟

---

## 注意事项

- 分片数必须是 2 的幂（hash & mask 性能最佳）
- 本地缓存在高并发下仍需要写锁保护（sync.RWMutex）
- Redis 操作必须设 context timeout
- 异步回填必须用 semaphore 限流，防止缓存穿透
- 预热在启动时异步执行，不阻塞主流程
- 压测数据必须标注环境（CPU、内存、并发数、数据量），标注为"压测记录"

> 来源：WaveYo / YoOSF-API AcquireLock + Koishi-Mirror semaphore 缓存模式。
