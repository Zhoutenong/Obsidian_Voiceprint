# 点位数据清理任务 (PointDataCleanupJob)

## 概述
定期清理过期的点位数据，控制数据库大小，是数据维护的重要任务。

## 职责
- 根据保留期删除过期数据
- 清理相关缓存
- 记录清理统计
- 优化数据库性能

## 配置

### appsettings.json

```json
{
  "PointDataCleanupJob": {
    "Enabled": true,
    "CronExpression": "0 0 4 * * *",
    "RetentionDays": 90,
    "BatchSize": 10000
  }
}
```

## 执行流程

```
计算截止时间 → 批量删除数据 → 清理缓存 → 优化数据库 → 记录统计
```

### 详细步骤

1. **计算截止时间**
   - 根据保留期计算截止日期
   - 生成删除条件

2. **批量删除**
   - 按批次删除数据
   - 记录删除数量
   - 处理删除异常

3. **清理缓存**
   - 清理相关 Redis 缓存
   - 清理内存缓存

4. **优化数据库**
   - 重建索引
   - 更新统计信息

5. **记录统计**
   - 记录删除数量
   - 记录执行时间
   - 记录释放空间

## 数据保留策略

### 保留期配置

| 数据类型 | 保留期 | 说明 |
|---------|--------|------|
| 实时数据 | 90 天 | 默认保留期 |
| 告警数据 | 365 天 | 告警数据长期保留 |
| 历史归档 | 永久 | 重要数据归档 |
| 临时数据 | 7 天 | 临时数据短保留 |

### 分级保留

```csharp
public async Task ExecuteAsync()
{
    var cutoffDate = DateTime.Now.AddDays(-_retentionDays);

    // 清理实时数据
    await CleanupRealtimeDataAsync(cutoffDate);

    // 清理临时数据
    await CleanupTempDataAsync(DateTime.Now.AddDays(-7));

    // 告警数据不清理
    // 归档数据不清理
}
```

## 执行逻辑

### 批量删除

```csharp
public async Task ExecuteAsync()
{
    _logger.LogInformation("开始执行点位数据清理任务");

    var cutoffDate = DateTime.Now.AddDays(-_retentionDays);
    var totalDeleted = 0;
    var batch = _batchSize;

    while (true)
    {
        // 批量删除
        var deleted = await _pointDataRepository.DeleteAsync(
            predicate: d => d.Timestamp < cutoffDate,
            take: batch
        );

        totalDeleted += deleted;

        // 没有更多数据
        if (deleted < batch)
            break;

        // 避免长时间锁定
        await Task.Delay(100);
    }

    // 清理缓存
    await CleanupCacheAsync(cutoffDate);

    // 优化数据库
    await OptimizeDatabaseAsync();

    _logger.LogInformation("点位数据清理完成，删除: {Count} 条", totalDeleted);
}
```

### 缓存清理

```csharp
private async Task CleanupCacheAsync(DateTime cutoffDate)
{
    // 清理 Redis 缓存
    var keys = await _redis.KeysAsync("point:data:*");
    foreach (var key in keys)
    {
        var ttl = await _redis.KeyExpireTimeAsync(key);
        if (ttl < cutoffDate)
        {
            await _redis.KeyDeleteAsync(key);
        }
    }
}
```

## 数据优化

### 索引优化

```sql
-- 重建索引
ALTER INDEX ALL ON PointData REORGANIZE;

-- 更新统计信息
UPDATE STATISTICS PointData;

-- 收缩数据库
DBCC SHRINKDATABASE (IntelliSub);
```

### 表分区

```sql
-- 按日期分区
CREATE PARTITION FUNCTION PointDataPartitionFunc (date)
AS RANGE RIGHT FOR VALUES ('2026-01-01', '2026-02-01', ...);

-- 分区方案
CREATE PARTITION SCHEME PointDataPartitionScheme
AS PARTITION PointDataPartitionFunc
ALL TO ([PRIMARY]);
```

## 依赖服务

- [[AstPointDataService]] - 点位数据服务
- [[PointValueCacheCleanupJob]] - 点位值缓存清理任务

## 相关任务

- [[PointValueCacheCleanupJob]] - 点位值缓存清理任务

## 异常处理

### 删除异常
- 锁定超时 → 重试删除
- 死锁 → 回滚重试
- 约束冲突 → 记录日志

### 性能问题
- 删除缓慢 → 减小批次大小
- 内存占用 → 增加延迟
- 数据库锁 → 分时段删除

## 日志记录

```
[2026-06-03 04:00:00] INFO  开始执行点位数据清理任务
[2026-06-03 04:00:00] INFO  截止日期: 2026-03-05
[2026-06-03 04:00:05] INFO  批量删除: 第1批，删除 10000 条
[2026-06-03 04:00:10] INFO  批量删除: 第2批，删除 10000 条
[2026-06-03 04:01:00] INFO  批量删除: 第3批，删除 5000 条
[2026-06-03 04:01:00] INFO  清理缓存
[2026-06-03 04:01:05] INFO  优化数据库
[2026-06-03 04:01:10] INFO  点位数据清理完成，删除: 25000 条
```

## 监控指标

### 清理统计
- 删除记录数量
- 执行时间
- 释放空间大小
- 缓存清理数量

### 数据库性能
- 删除前数据库大小
- 删除后数据库大小
- 索引碎片率
- 查询性能提升

## 配置项

```json
{
  "PointDataCleanupJob": {
    "Enabled": true,
    "CronExpression": "0 0 4 * * *",
    "RetentionDays": 90,
    "BatchSize": 10000,
    "DeleteTempData": true,
    "TempRetentionDays": 7,
    "OptimizeDatabase": true
  }
}
```

## 最佳实践

1. **执行策略**
   - 选择低峰时段执行
   - 分批删除避免长时间锁表
   - 定期检查数据库大小

2. **性能优化**
   - 使用批量删除
   - 删除前备份重要数据
   - 删除后优化数据库

3. **监控告警**
   - 执行时间过长告警
   - 删除失败告警
   - 数据库空间告警

4. **数据安全**
   - 删除前确认备份
   - 重要数据长期保留
   - 支持数据恢复

## 相关文档

- [[PointValueCacheCleanupJob]] - 点位值缓存清理任务文档
- [[AstPointDataService]] - 点位数据服务文档
- [[PointData]] - 点位数据实体

## 注意事项

1. **数据备份**：删除前建议备份数据
2. **业务确认**：确认数据保留策略
3. **分批处理**：大批量删除要分批进行
4. **索引维护**：删除后重建索引
5. **空间监控**：监控数据库空间使用

## TDengine 清理

如果使用 TDengine 存储时序数据：

```sql
-- 删除过期数据
DELETE FROM point_data WHERE ts < NOW - 90d;

-- 删除子表
DROP TABLE IF EXISTS point_data_202601;

-- 修改数据保留期
ALTER DATABASE intellisub KEEP 90;
```

## 数据归档

对于需要长期保留的数据：

```csharp
// 归档到历史表
await _pointDataRepository.ArchiveAsync(cutoffDate);

// 导出到文件
await _pointDataRepository.ExportAsync(cutoffDate, exportPath);

// 导入到数据仓库
await _pointDataRepository.WarehouseAsync(cutoffDate);
```
