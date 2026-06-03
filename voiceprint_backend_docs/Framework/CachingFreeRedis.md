# Yi.Framework.Caching.FreeRedis

## 概述

`Yi.Framework.Caching.FreeRedis` 是基于 FreeRedis 的分布式缓存实现，为系统提供高性能的 Redis 缓存支持。该模块得益于 FreeRedis 作者对 `IDistributedCache` 接口的实现，能够无缝集成到 ABP 框架的缓存体系中。

**核心特性：**
- ✅ 完整的 IDistributedCache 接口实现
- ✅ 可配置的 Key 前缀规范
- ✅ 多租户支持（可选，当前已禁用）
- ✅ 灵活的启用/禁用配置

## 模块配置

### 依赖模块

```csharp
[DependsOn(typeof(AbpCachingModule))]
public class YiFrameworkCachingFreeRedisModule : AbpModule
```

### 配置项

在 `appsettings.json` 中配置 Redis 连接：

```json
{
  "Redis": {
    "IsEnabled": true,
    "Configuration": "localhost:6379,defaultDatabase=0"
  }
}
```

**配置说明：**
- `IsEnabled`: Redis 缓存启用开关，默认 `true`
- `Configuration`: FreeRedis 连接字符串，支持多种格式

**FreeRedis 连接字符串格式：**
```
localhost:6379,defaultDatabase=0
password=123456@localhost:6379,defaultDatabase=0
192.168.1.100:6379,password=123456,defaultDatabase=1
```

### 服务注册

模块在 `ConfigureServices` 中自动注册 Redis 服务：

```csharp
public override void ConfigureServices(ServiceConfigurationContext context)
{
    var configuration = context.Services.GetConfiguration();
    var redisEnabled = configuration["Redis:IsEnabled"];
    
    if (redisEnabled.IsNullOrEmpty() || bool.Parse(redisEnabled))
    {
        var redisConfiguration = configuration["Redis:Configuration"];
        RedisClient redisClient = new RedisClient(redisConfiguration);

        context.Services.AddSingleton<IRedisClient>(redisClient);
        context.Services.Replace(ServiceDescriptor.Singleton<IDistributedCache>(
            new DistributedCache(redisClient)));
    }
}
```

**关键服务：**
- `IRedisClient`: FreeRedis 客户端单例
- `IDistributedCache`: 替换 ABP 默认的内存缓存实现

## Key 规范化

### YiDistributedCacheKeyNormalizer

自定义的缓存 Key 规范化器，继承自 `IDistributedCacheKeyNormalizer`。

```csharp
[Dependency(ReplaceServices = true)]
public class YiDistributedCacheKeyNormalizer : IDistributedCacheKeyNormalizer, ITransientDependency
{
    protected ICurrentTenant CurrentTenant { get; }
    protected AbpDistributedCacheOptions DistributedCacheOptions { get; }

    public YiDistributedCacheKeyNormalizer(
        ICurrentTenant currentTenant,
        IOptions<AbpDistributedCacheOptions> distributedCacheOptions)
    {
        CurrentTenant = currentTenant;
        DistributedCacheOptions = distributedCacheOptions.Value;
    }
}
```

### Key 规范化规则

```csharp
public virtual string NormalizeKey(DistributedCacheKeyNormalizeArgs args)
{
    var normalizedKey = $"{DistributedCacheOptions.KeyPrefix}{args.Key}";
    
    // 多租户支持（当前已禁用）
    //if (!args.IgnoreMultiTenancy && CurrentTenant.Id.HasValue)
    //{
    //    normalizedKey = $"t:{CurrentTenant.Id.Value},{normalizedKey}";
    //}

    return normalizedKey;
}
```

**Key 格式：**
```
{KeyPrefix}{OriginalKey}
```

**示例：**
- 原始 Key: `User:123`
- KeyPrefix (默认): `abp-`
- 规范化后: `abp-User:123`

### 多租户支持（已禁用）

代码中包含多租户 Key 前缀的实现，但已被注释：

```csharp
//if (!args.IgnoreMultiTenancy && CurrentTenant.Id.HasValue)
//{
//    normalizedKey = $"t:{CurrentTenant.Id.Value},{normalizedKey}";
//}
```

如需启用多租户支持，取消注释即可。格式为：
```
t:{TenantId},{KeyPrefix}{OriginalKey}
```

## 使用方式

### 依赖注入

```csharp
public class MyService : ITransientDependency
{
    private readonly IDistributedCache _distributedCache;

    public MyService(IDistributedCache distributedCache)
    {
        _distributedCache = distributedCache;
    }
}
```

### 基本操作

**设置缓存：**
```csharp
await _distributedCache.SetStringAsync("myKey", "myValue");
await _distributedCache.SetAsync("myKey", byteArray, new DistributedCacheEntryOptions
{
    AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(30)
});
```

**获取缓存：**
```csharp
string value = await _distributedCache.GetStringAsync("myKey");
byte[] data = await _distributedCache.GetAsync("myKey");
```

**删除缓存：**
```csharp
await _distributedCache.RemoveAsync("myKey");
```

### ABp 缓存集成

该模块无缝集成 ABP 缓存系统，可配合 `ICache` 接口使用：

```csharp
public class MyService : ITransientDependency
{
    private readonly ICache<int, UserDto> _userCache;

    public MyService(ICache<int, UserDto> userCache)
    {
        _userCache = userCache;
    }

    public async Task<UserDto> GetUserAsync(int userId)
    {
        return await _userCache.GetOrAddAsync(userId, async () => 
        {
            return await _userRepository.GetAsync(userId);
        });
    }
}
```

## 禁用 Redis 缓存

如需禁用 Redis 缓存，修改配置：

```json
{
  "Redis": {
    "IsEnabled": false
  }
}
```

系统将自动回退到 ABP 默认的内存缓存实现。

## 性能优化建议

1. **连接池管理**: FreeRedis 内置连接池管理，无需额外配置
2. **序列化选择**: 复杂对象建议使用 JSON 序列化
3. **过期时间策略**: 根据业务场景设置合理的过期时间
4. **Key 命名**: 使用有意义的 Key 命名规范，便于监控和排查

## 故障排查

**连接失败：**
- 检查 Redis 服务是否启动
- 验证 `Redis:Configuration` 连接字符串格式
- 确认网络连接和防火墙设置

**缓存不生效：**
- 确认 `Redis:IsEnabled` 设置为 `true`
- 检查 Key 前缀配置是否正确
- 验证数据是否被其他缓存覆盖

## 参考资源

- [FreeRedis GitHub](https://github.com/2881099/csredis)
- [ABP Distributed Cache Documentation](https://docs.abp.io/en/abp/latest/Distributed-Caching)
