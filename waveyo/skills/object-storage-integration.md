# Object Storage Integration — WaveYo 风格

> 触发条件：需要接入 S3 / OSS / COS 对象存储时加载。
> 来源证据：YoOSF-API, Koishi-Mirror 的对象存储集成模式。

---

## 支持的存储后端

- AWS S3
- 阿里云 OSS
- 腾讯云 COS
- MinIO（自建兼容 S3）

统一通过 S3 兼容接口接入，使用相同的客户端模式。

---

## 客户端初始化

```go
type StorageClient struct {
    client     *s3.Client
    bucket     string
    endpoint   string
    useSSL     bool
    logger     *zap.Logger
}

func NewStorageClient(cfg StorageConfig, logger *zap.Logger) (*StorageClient, error) {
    resolver := aws.EndpointResolverWithOptionsFunc(
        func(service, region string, options ...interface{}) (aws.Endpoint, error) {
            return aws.Endpoint{
                URL:               cfg.Endpoint,
                HostnameImmutable: true,
                SigningRegion:     cfg.Region,
            }, nil
        },
    )

    creds := credentials.NewStaticCredentialsProvider(
        cfg.AccessKey, cfg.SecretKey, "",
    )

    client := s3.New(s3.Options{
        Region:           cfg.Region,
        EndpointResolver: resolver,
        Credentials:      creds,
        UsePathStyle:     !cfg.UseVirtualHost,
    })

    return &StorageClient{
        client:   client,
        bucket:   cfg.Bucket,
        endpoint: cfg.Endpoint,
        useSSL:   cfg.UseSSL,
        logger:   logger,
    }, nil
}
```

---

## 核心操作

### 预签名 URL

```go
func (s *StorageClient) PresignGet(ctx context.Context, key string, ttl time.Duration) (string, error) {
    presignClient := s3.NewPresignClient(s.client)
    req, err := presignClient.PresignGetObject(ctx, &s3.GetObjectInput{
        Bucket: aws.String(s.bucket),
        Key:    aws.String(key),
    }, func(opts *s3.PresignOptions) {
        opts.Expires = ttl
    })
    if err != nil {
        return "", fmt.Errorf("presign get: %w", err)
    }
    return req.URL, nil
}
```

### CDN 优先下载链路

```
用户请求文件
  │
  ├─ 1. 构造 CDN URL（如果配置了 CDN）
  │     → 返回 CDN URL，浏览器重定向
  │
  └─ 2. CDN 未配置或 miss
        → 生成 S3 预签名 URL
        → 返回重定向到源站
```

```go
func (s *StorageClient) GetDownloadURL(ctx context.Context, key string) (string, error) {
    if s.cdnBase != "" {
        return s.cdnBase + "/" + key, nil
    }
    return s.PresignGet(ctx, key, 1*time.Hour)
}
```

### 文件列表（游标分页）

```go
func (s *StorageClient) ListObjects(ctx context.Context, prefix string, limit int32, cursor string) ([]ObjectInfo, string, error) {
    input := &s3.ListObjectsV2Input{
        Bucket:     aws.String(s.bucket),
        Prefix:     aws.String(prefix),
        MaxKeys:    aws.Int32(limit),
        StartAfter: aws.String(cursor),  // 游标分页，非 offset
    }

    var objects []ObjectInfo
    var nextCursor string

    paginator := s3.NewListObjectsV2Paginator(s.client, input)
    pageNum := 0
    for paginator.HasMorePages() && pageNum < 1 {
        page, err := paginator.NextPage(ctx)
        if err != nil {
            return nil, "", fmt.Errorf("list objects: %w", err)
        }
        for _, obj := range page.Contents {
            objects = append(objects, ObjectInfo{
                Key:          aws.ToString(obj.Key),
                Size:         aws.ToInt64(obj.Size),
                LastModified: aws.ToTime(obj.LastModified),
            })
        }
        if page.NextContinuationToken != nil {
            nextCursor = *page.NextContinuationToken
        }
        pageNum++
    }
    return objects, nextCursor, nil
}
```

---

## 镜像同步编排

```go
type MirrorSync struct {
    source *StorageClient
    target *StorageClient
    cache  *MultiLevelCache
    logger *zap.Logger
}

func (m *MirrorSync) SyncObject(ctx context.Context, key string) error {
    // 1. 查本地缓存（避免重复同步）
    if _, ok := m.cache.Get("mirror:" + key); ok {
        return nil
    }

    // 2. 从源站下载
    obj, err := m.source.GetObject(ctx, key)
    if err != nil {
        return fmt.Errorf("get source: %w", err)
    }
    defer obj.Body.Close()

    // 3. 上传到目标
    _, err = m.target.client.PutObject(ctx, &s3.PutObjectInput{
        Bucket:      aws.String(m.target.bucket),
        Key:         aws.String(key),
        Body:        obj.Body,
        ContentType: obj.ContentType,
    })
    if err != nil {
        return fmt.Errorf("put target: %w", err)
    }

    // 4. 标记已同步
    m.cache.Set("mirror:"+key, true, 24*time.Hour)
    m.logger.Info("mirror synced", zap.String("key", key))
    return nil
}
```

---

## 注意事项

- 预签名 URL TTL 根据文件大小设置：小文件（<10MB）1h，大文件（>100MB）24h
- 对象列表使用游标分页（ContinuationToken），不使用 offset 分页
- 文件删除操作必须检查是否有依赖（如缓存中的引用）
- 镜像同步失败需要重试逻辑（exponential backoff）
- CDN 回源策略优先于直连 S3，减少源站压力
- 所有 S3 操作设置 context timeout

> 来源：WaveYo / YoOSF-API 和 Koishi-Mirror 的 S3/COS 集成模式。
