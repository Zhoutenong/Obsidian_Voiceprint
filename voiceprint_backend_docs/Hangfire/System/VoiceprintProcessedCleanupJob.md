# 声纹处理后清理任务 (VoiceprintProcessedCleanupJob)

## 概述
清理已处理的音频文件，释放存储空间，是声纹模块的定期维护任务。

## 职责
- 查询处理完成的音频记录
- 检查文件保留期限
- 删除过期音频文件
- 更新数据库记录
- 记录清理统计

## 配置

### appsettings.json

```json
{
  "VoiceprintProcessedCleanupJob": {
    "Enabled": true,
    "CronExpression": "0 0 3 * * *",
    "RetentionDays": 30,
    "BatchSize": 100
  }
}
```

### Cron 表达式说明

```
0 0 3 * * *       // 每天凌晨3点执行
0 0 3 ? * MON     // 每周一凌晨3点执行
0 0 3 1 * *       // 每月1号凌晨3点执行
```

## 执行流程

```
查询待处理记录 → 检查文件创建时间 → 删除过期文件 → 更新数据库 → 记录统计
```

### 详细步骤

1. **查询待处理记录**
   - 查询状态为 `Processed` 的音频记录
   - 过滤出有文件路径的记录
   - 按批次处理

2. **检查文件期限**
   - 获取文件创建时间
   - 计算文件保存天数
   - 判断是否超过保留期

3. **删除文件**
   - 删除原始音频文件
   - 删除处理后音频文件
   - 删除临时文件

4. **更新数据库**
   - 更新记录状态为 `Cleaned`
   - 清空文件路径
   - 记录清理时间

5. **记录统计**
   - 记录清理文件数量
   - 记录释放空间大小
   - 记录失败数量

## 清理规则

### 文件类型

| 文件类型 | 路径模式 | 是否清理 |
|---------|----------|---------|
| 原始音频 | `/voiceprint/original/{id}.wav` | 是 |
| 处理音频 | `/voiceprint/processed/{id}.wav` | 是 |
| 临时文件 | `/voiceprint/temp/{id}.tmp` | 是 |
| 标准音频 | `/voiceprint/standard/{id}.wav` | 否 |

### 保留规则

```csharp
public bool ShouldCleanup(AudioRecord record, int retentionDays)
{
    // 标准音频不清理
    if (record.IsStandard)
        return false;

    // 未处理的音频不清理
    if (record.Status != AudioStatus.Processed)
        return false;

    // 未超过保留期的不清理
    if (record.ProcessedAt.HasValue &&
        (DateTime.Now - record.ProcessedAt.Value).Days < retentionDays)
        return false;

    return true;
}
```

## 执行逻辑

### 批量处理

```csharp
public async Task ExecuteAsync()
{
    _logger.LogInformation("开始执行声纹音频清理任务");

    var totalCleaned = 0;
    var totalFailed = 0;
    var totalSize = 0L;

    // 分批处理
    var batchSize = _batchSize;
    var skip = 0;

    while (true)
    {
        // 查询一批记录
        var records = await _audioRepository.GetListAsync(
            skip: skip,
            take: batchSize,
            status: AudioStatus.Processed
        );

        if (!records.Any())
            break;

        // 处理每条记录
        foreach (var record in records)
        {
            var result = await CleanupRecordAsync(record);
            if (result.Success)
            {
                totalCleaned++;
                totalSize += result.FileSize;
            }
            else
            {
                totalFailed++;
            }
        }

        skip += batchSize;
    }

    _logger.LogInformation(
        "声纹音频清理完成，清理: {Cleaned}, 失败: {Failed}, 释放空间: {Size}MB",
        totalCleaned, totalFailed, totalSize / 1024 / 1024
    );
}
```

### 文件删除

```csharp
private async Task<CleanupResult> CleanupRecordAsync(AudioRecord record)
{
    try
    {
        var totalSize = 0L;

        // 删除原始音频
        if (!string.IsNullOrEmpty(record.OriginalPath))
        {
            var size = await _fileService.DeleteAsync(record.OriginalPath);
            totalSize += size;
        }

        // 删除处理音频
        if (!string.IsNullOrEmpty record.ProcessedPath))
        {
            var size = await _fileService.DeleteAsync(record.ProcessedPath);
            totalSize += size;
        }

        // 更新数据库
        await _audioRepository.UpdateAsync(record.Id, new
        {
            Status = AudioStatus.Cleaned,
            OriginalPath = null,
            ProcessedPath = null,
            CleanedAt = DateTime.Now
        });

        return new CleanupResult
        {
            Success = true,
            FileSize = totalSize
        };
    }
    catch (Exception ex)
    {
        _logger.LogError(ex, "清理音频记录 {RecordId} 失败", record.Id);
        return new CleanupResult { Success = false };
    }
}
```

## 依赖服务

- [[VoiceprintAudioAppService]] - 声纹音频应用服务
- [[FileService]] - 文件服务
- [[AudioRecord]] - 音频记录实体

## 相关任务

- [[VoiceprintCaptureJob]] - 声纹采集任务
- [[VoiceprintProcessedCleanupJob]] - 当前任务

## 任务状态

### 音频记录状态

| 状态 | 说明 | 是否清理 |
|-----|------|---------|
| Pending | 待处理 | 否 |
| Processing | 处理中 | 否 |
| Processed | 已处理 | 是（超过保留期） |
| Cleaned | 已清理 | 否 |

## 异常处理

### 文件删除异常
- 文件不存在 → 记录日志，继续处理
- 文件占用 → 记录日志，下次重试
- 权限不足 → 记录日志，发送告警

### 数据库异常
- 更新失败 → 回滚操作，记录日志
- 连接超时 → 重试连接
- 记录不存在 → 跳过处理

## 日志记录

```
[2026-06-03 03:00:00] INFO  开始执行声纹音频清理任务
[2026-06-03 03:00:01] INFO  查询到 500 条待处理记录
[2026-06-03 03:00:05] INFO  批量清理 0-100 条记录
[2026-06-03 03:00:15] INFO  批量清理 100-200 条记录
[2026-06-03 03:01:00] INFO  批量清理 400-500 条记录
[2026-06-03 03:01:00] INFO  声纹音频清理完成
[2026-06-03 03:01:00] INFO  清理: 450, 失败: 50, 释放空间: 5120MB
```

## 监控指标

### 清理统计
- 清理文件数量
- 释放存储空间
- 清理失败数量
- 平均执行时间

### 存储统计
- 当前存储使用量
- 清理后存储使用量
- 存储节省比例

## 配置项

```json
{
  "VoiceprintProcessedCleanupJob": {
    "Enabled": true,
    "CronExpression": "0 0 3 * * *",
    "RetentionDays": 30,
    "BatchSize": 100,
    "DeleteStandardAudio": false,
    "MaxRetryCount": 3
  }
}
```

## 最佳实践

1. **执行时间**
   - 选择低峰时段执行（凌晨3点）
   - 避免与其他清理任务冲突

2. **保留策略**
   - 根据存储空间调整保留期
   - 重要音频延长保留期
   - 标准音频不清理

3. **性能优化**
   - 批量处理提高效率
   - 控制批次大小避免内存溢出
   - 使用后台任务避免阻塞

4. **监控告警**
   - 清理失败告警
   - 存储空间告警
   - 执行超时告警

## 相关文档

- [[VoiceprintCaptureJob]] - 声纹采集任务文档
- [[VoiceprintAudioAppService]] - 声纹音频应用服务文档
- [[FileService]] - 文件服务文档

## 注意事项

1. **数据安全**：删除前确认不需要保留
2. **标准音频**：标准音频库文件不清理
3. **法律合规**：考虑数据保留的法律要求
4. **存储监控**：监控存储空间使用情况
5. **失败重试**：清理失败的文件下次重试

## 存储优化建议

1. **分级存储**
   - 热数据（30天内）：本地存储
   - 温数据（30-90天）：归档存储
   - 冷数据（90天以上）：删除或云存储

2. **压缩存储**
   - 压缩历史音频文件
   - 使用有损压缩降低存储

3. **定期审计**
   - 审计清理策略
   - 审计清理执行情况
   - 审计存储使用情况
