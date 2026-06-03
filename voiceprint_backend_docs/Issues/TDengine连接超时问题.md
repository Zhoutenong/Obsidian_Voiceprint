# TDengine连接超时问题

## 问题描述

在高并发场景下，TDengine 数据库频繁出现连接超时错误，导致时序数据写入失败。

### 症状

- 大量数据写入时连接超时
- 错误日志: "TDengine connection timeout" 或 "Unable to connect to TDengine"
- 查询响应缓慢
- 偶发性连接断开

### 影响范围

- **模块**: ast-intellisubdata (时序数据采集)
- **数据库**: TDengine 3.x
- **场景**: 高频传感器数据写入、聚合查询

## 根本原因

1. **连接池配置不当**
   - 默认连接数不足
   - 连接超时时间过短
   - 无连接复用机制

2. **网络延迟**
   - TDengine 服务器负载高
   - 网络不稳定
   - 防火墙限制

3. **资源竞争**
   - 并发写入超过数据库承载能力
   - 查询阻塞写入操作
   - 连接池耗尽

## 解决方案

### 方案 1: 优化连接池配置

```json
{
  "DbConnOptions": {
    "DeploymentMode": "HighPerformance",
    "TDengineUrl": "Host=192.168.3.111;Port=6030;Database=ast;Username=root;Password=taosdata;",
    "TDengineConnectionPoolSize": 50,
    "TDengineConnectionTimeout": 30,
    "TDengineReadTimeout": 60,
    "TDengineWriteTimeout": 30
  }
}
```

### 方案 2: 实现重试机制

```csharp
// AstPointDataService.cs
public async Task<bool> WritePointDataWithRetryAsync(PointDataEntity data, int maxRetries = 3)
{
    for (int attempt = 1; attempt <= maxRetries; attempt++)
    {
        try
        {
            await _pointDataRepository.InsertAsync(data);
            return true;
        }
        catch (Exception ex) when (attempt < maxRetries)
        {
            _logger.LogWarning(ex, 
                "写入TDengine失败 (尝试 {Attempt}/{MaxRetries}): {Message}", 
                attempt, maxRetries, ex.Message);
            
            // 指数退避: 1s, 2s, 4s
            var delay = TimeSpan.FromSeconds(Math.Pow(2, attempt - 1));
            await Task.Delay(delay);
        }
    }
    
    _logger.LogError("写入TDengine最终失败，已重试 {MaxRetries} 次", maxRetries);
    return false;
}
```

### 方案 3: 批量写入优化

```csharp
// 使用批量插入减少连接开销
public async Task BulkInsertAsync(List<PointDataEntity> dataList)
{
    if (dataList.Count == 0) return;

    try
    {
        // TDengine 支持批量插入
        await _tdDb.GetDbClient().Insertable(dataList)
            .ExecuteCommandAsync();
            
        _logger.LogInformation("批量写入 {Count} 条TDengine数据成功", dataList.Count);
    }
    catch (Exception ex)
    {
        _logger.LogError(ex, "批量写入TDengine失败: {Count} 条", dataList.Count);
        throw;
    }
}
```

### 方案 4: 子表名优化

TDengine 子表名过长导致连接开销增大：

```csharp
// ChildTableNameManager.cs
public string GenerateChildTableName(string superTableName, string deviceId, 
    string sensorKey, string property)
{
    const int MAX_TABLE_NAME_LENGTH = 192;
    
    // 清理非法字符
    var cleanDeviceId = CleanIdentifier(deviceId);
    var cleanSensorKey = CleanIdentifier(sensorKey);
    var cleanProperty = CleanIdentifier(property);
    
    // 构造表名
    var baseName = $"{superTableName}_{cleanDeviceId}_{cleanSensorKey}_{cleanProperty}"
        .ToLower();
    
    // 如果超长，采用"截断 + 短哈希"策略
    if (baseName.Length > MAX_TABLE_NAME_LENGTH)
    {
        var shortHash = GenerateShortHash(deviceId, sensorKey, property);
        var prefix = $"{superTableName}_";
        var availableLength = MAX_TABLE_NAME_LENGTH - prefix.Length - shortHash.Length - 1;
        
        var truncatedTags = $"{cleanDeviceId}_{cleanSensorKey}_{cleanProperty}"
            .Substring(0, Math.Min(availableLength, baseName.Length));
        
        return $"{prefix}{truncatedTags}_{shortHash}".ToLower();
    }
    
    return baseName;
}
```

## 配置最佳实践

### TDengine 服务器配置

```ini
# taos.cfg
# 最大连接数
maxConnections = 1000

# 查询超时时间 (秒)
queryTimes = 600

# 缓冲区大小 (MB)
buffer = 1024

# 并行查询线程数
numOfQueryThreads = 4

# 批量写入支持
supportBatchInsert = 1
```

### 应用层配置

```csharp
// TDengineDbContext.cs
public TDengineDbContext(IOptions<DbConnOptions> dbConfigOptions, ILogger<TDengineDbContext> logger)
{
    var dbConfig = dbConfigOptions.Value;
    
    if (dbConfig.DeploymentMode == DeploymentMode.HighPerformance)
    {
        // 配置连接参数
        ConnectionString = dbConfig.TDengineUrl;
        
        // 初始化 SqlSugar 客户端
        InstanceFactory.CustomAssemblies = new[] { typeof(TDengineProvider).Assembly };
        
        var configId = "TDengine";
        InstanceFactory.Add(configId, new ConnectionConfig
        {
            ConfigId = configId,
            ConnectionString = ConnectionString,
            DbType = DbType.TDengine,
            IsAutoCloseConnection = true,  // 自动释放连接
            InitKeyType = InitKeyType.Attribute,
            
            // 连接池配置
            ConfigureExternalServices = new ConfigureExternalServices
            {
                // 自定义连接处理
            }
        });
    }
}
```

## 监控指标

| 指标 | 监控方式 | 正常范围 |
|------|---------|----------|
| 连接池使用率 | 应用日志 | < 80% |
| 写入延迟 | TDengine `show dnodes` | < 100ms |
| 查询响应时间 | 应用计时 | < 1s |
| 失败率 | 错误计数 | < 0.1% |

## 故障排查步骤

### 1. 检查网络连接

```bash
# 测试 TDengine 端口
telnet 192.168.3.111 6030

# 或使用 nc
nc -zv 192.168.3.111 6030
```

### 2. 检查 TDengine 服务状态

```bash
# 查看 TDengine 日志
tail -f /var/log/taos/taosd.log

# 检查数据节点状态
taos -s "show dnodes;"
```

### 3. 查看应用日志

```bash
# 查看连接超时错误
grep "TDengine" logs/all/*.log | grep -i timeout
```

## 预防措施

1. **连接预热**
   ```csharp
   // 应用启动时建立连接池
   public async Task InitializeAsync()
   {
       var testConn = _tdDb.GetDbClient();
       await testConn.Ado.ExecuteCommandAsync("SELECT 1");
       _logger.LogInformation("TDengine连接池初始化完成");
   }
   ```

2. **限流保护**
   ```csharp
   // 使用信号量限制并发写入
   private readonly SemaphoreSlim _writeSemaphore = new(10, 10);
   
   public async Task WriteDataAsync(PointDataEntity data)
   {
       await _writeSemaphore.WaitAsync();
       try
       {
           await _repository.InsertAsync(data);
       }
       finally
       {
           _writeSemaphore.Release();
       }
   }
   ```

3. **降级策略**
   ```csharp
   // TDengine 不可用时降级到 SQLite
   if (!IsTDengineAvailable())
   {
       _logger.LogWarning("TDengine不可用，降级到SQLite存储");
       await _sqliteRepository.InsertAsync(data);
   }
   ```

## 相关文件

- `/module/ast-intellisubdata/Ast.IntelliSubData.SqlSugarCore/ITDengineDbContext.cs` - TDengine 上下文
- `/module/ast-intellisubdata/Ast.IntelliSubData.Application/Services/AstPointDataService.cs` - 数据服务
- `/module/ast-intellisubdata/Ast.IntelliSubData.Application/Managers/ChildTableNameManager.cs` - 子表管理

## 参考资料

- [TDengine 官方文档](https://docs.tdengine.com/)
- [SqlSugar TDengine 集成](https://www.donet5.com/Home/Doc/type/6/2210)
