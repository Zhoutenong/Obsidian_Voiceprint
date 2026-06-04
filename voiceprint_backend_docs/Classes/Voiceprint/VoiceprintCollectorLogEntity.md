# VoiceprintCollectorLogEntity — 声纹采集通信日志实体

## 基本信息

- **实体名称**：`VoiceprintCollectorLogEntity`
- **数据库表**：`vp_collector_comm_log`
- **模块位置**：`module/ast-voiceprint/Ast.Voiceprint.Domain/Entities/`
- **继承关系**：`AggregateRoot<Guid>`

## 实体说明

声纹采集通信日志实体用于记录边缘采集器与后端系统的通信状态。每次采集任务的通信事件都会记录在此表中，用于监控采集器健康状态和排查通信问题。

## 字段说明

### 主键与关联

| 字段名 | 数据类型 | 说明 | 约束 |
|-------|---------|------|------|
| `Id` | `Guid` | 主键 | Primary Key |

### 采集器信息

| 字段名 | 数据类型 | 说明 | 备注 |
|-------|---------|------|------|
| `CollectorDeviceId` | `string(64)` | 采集器设备编号 | 边缘设备唯一标识 |
| `MonitoredObjectId` | `Guid?` | 关联监测对象ID | 可选，用于关联业务对象 |
| `GroupId` | `Guid?` | 采集任务批次 | 用于关联同一批次的多次通信 |

### 通信状态

| 字段名 | 数据类型 | 说明 | 备注 |
|-------|---------|------|------|
| `Status` | `string(32)` | 通信状态 | 如："Connected"、"Failed"、"Timeout" |
| `Message` | `string(256)?` | 状态描述 | 详细说明或错误信息 |

### 时间信息

| 字段名 | 数据类型 | 说明 | 备注 |
|-------|---------|------|------|
| `OccurredAt` | `DateTime` | 状态发生时间 | 实际事件发生时间 |
| `CreatedAt` | `DateTime` | 创建时间 | 日志记录创建时间 |

## 业务规则

### 日志记录规则
1. **完整记录**：每次与采集器通信都记录日志
2. **状态分类**：Status 字段使用标准状态码
3. **时间精确**：OccurredAt 记录实际发生时间，CreatedAt 记录入库时间
4. **批次关联**：通过 GroupId 关联同一采集任务的多条日志

### 通信状态类型

| 状态码 | 说明 | 典型场景 | 处理建议 |
|-------|------|---------|---------|
| `Connected` | 连接成功 | 采集器正常连接 | - |
| `Disconnected` | 连接断开 | 采集器离线 | 检查网络和设备 |
| `Timeout` | 连接超时 | 网络延迟或设备无响应 | 重试或检查设备 |
| `AuthFailed` | 认证失败 | Token过期或设备认证失败 | 重新认证 |
| `DataReceived` | 数据接收成功 | 音频数据接收成功 | - |
| `DataFailed` | 数据接收失败 | 音频数据损坏或格式错误 | 检查采集器配置 |
| `UploadSuccess` | 上传成功 | 音频文件上传成功 | - |
| `UploadFailed` | 上传失败 | 上传到存储失败 | 检查存储服务 |

## 使用场景

### 场景1：采集任务通信日志
```
采集任务批次: GroupId = xxx-xxx-xxx

2026-06-04 10:00:00 - Connected - 连接到采集器 Collector001
2026-06-04 10:00:01 - DataReceived - 接收音频数据 1024KB
2026-06-04 10:00:02 - UploadSuccess - 音频文件上传成功
2026-06-04 10:00:03 - Disconnected - 连接断开
```

### 场景2：通信异常排查
```
采集器 Collector002 通信异常：

2026-06-04 10:05:00 - Timeout - 连接超时
2026-06-04 10:06:00 - Timeout - 连接超时
2026-06-04 10:07:00 - AuthFailed - 认证失败

分析：采集器可能Token过期，需要重新认证
```

### 场景3：采集器健康监控
```
统计最近24小时通信状态：
- Collector001: Connected 95%, Timeout 3%, Failed 2% - 健康
- Collector002: Connected 60%, Timeout 30%, Failed 10% - 异常
- Collector003: Connected 98%, Timeout 1%, Failed 1% - 健康

行动：检查 Collector002 的网络和设备状态
```

## 相关实体

- [[VoiceprintDeviceAudioRecordEntity]] — 声纹设备音频记录
- [[MonitoredObjectAggregateRoot]] — 监测对象
- [[VoiceprintStandardAudioEntity]] — 标准音频库

## 相关服务

- [[VoiceprintCaptureRuntimeStateService]] — 声纹采集运行时状态服务
- [[VoiceprintAudioAppService]] — 声纹音频应用服务

## 数据查询示例

### 查询某采集器的最近通信日志
```sql
SELECT * FROM vp_collector_comm_log
WHERE collector_device_id = 'Collector001'
ORDER BY occurred_at DESC
LIMIT 50;
```

### 查询某批次的通信日志
```sql
SELECT * FROM vp_collector_comm_log
WHERE group_id = 'xxx-xxx-xxx'
ORDER BY occurred_at ASC;
```

### 统计采集器通信成功率（最近24小时）
```sql
SELECT
    collector_device_id,
    COUNT(*) as total_count,
    SUM(CASE WHEN status = 'Connected' THEN 1 ELSE 0 END) as connected_count,
    SUM(CASE WHEN status = 'DataReceived' THEN 1 ELSE 0 END) as data_received_count,
    SUM(CASE WHEN status LIKE '%Failed%' OR status = 'Timeout' THEN 1 ELSE 0 END) as failed_count,
    CAST(SUM(CASE WHEN status = 'Connected' OR status = 'DataReceived' THEN 1 ELSE 0 END) AS FLOAT) / COUNT(*) * 100 as success_rate
FROM vp_collector_comm_log
WHERE occurred_at >= DATEADD(HOUR, -24, GETDATE())
GROUP BY collector_device_id
ORDER BY success_rate DESC;
```

### 查询异常通信日志
```sql
SELECT
    collector_device_id,
    status,
    message,
    occurred_at
FROM vp_collector_comm_log
WHERE status IN ('Timeout', 'AuthFailed', 'DataFailed', 'UploadFailed')
AND occurred_at >= DATEADD(HOUR, -1, GETDATE())
ORDER BY occurred_at DESC;
```

## 索引建议

```sql
-- 采集器ID + 时间复合索引（用于查询某采集器的日志）
CREATE INDEX IX_collector_occurred ON vp_collector_comm_log(collector_device_id, occurred_at DESC);

-- 批次ID索引（用于查询某批次的日志）
CREATE INDEX IX_group_id ON vp_collector_comm_log(group_id);

-- 状态索引（用于查询异常日志）
CREATE INDEX IX_status ON vp_collector_comm_log(status);
```

## 业务价值

**核心作用**：
1. **通信监控**：实时监控采集器通信状态
2. **故障排查**：记录详细的通信日志，便于排查问题
3. **健康评估**：基于历史日志评估采集器健康度
4. **性能分析**：分析通信成功率和响应时间

## 设计模式

### 日志模式
- 采用事件日志模式，记录所有通信事件
- 支持通过 GroupId 关联同一批次的多个事件
- 时间戳分为 OccurredAt（发生时间）和 CreatedAt（入库时间）

### 健康度计算
```csharp
public async Task<CollectorHealthDto> CalculateHealthAsync(string collectorDeviceId)
{
    var logs = await _repository.GetListAsync(l =>
        l.CollectorDeviceId == collectorDeviceId &&
        l.OccurredAt >= DateTime.Now.AddHours(-24)
    );

    var total = logs.Count;
    var success = logs.Count(l => l.Status == "Connected" || l.Status == "DataReceived");
    var healthRate = total > 0 ? (double)success / total : 0;

    return new CollectorHealthDto
    {
        CollectorDeviceId = collectorDeviceId,
        HealthRate = healthRate,
        Status = healthRate > 0.9 ? "Healthy" : healthRate > 0.7 ? "Warning" : "Critical",
        TotalAttempts = total,
        SuccessAttempts = success,
        FailedAttempts = total - success
    };
}
```

## 告警集成

### 采集器离线告警
```csharp
// 检查采集器是否长时间无通信日志
public async Task CheckCollectorOfflineAsync()
{
    var threshold = DateTime.Now.AddMinutes(-10);

    var activeCollectors = await _collectorService.GetActiveCollectorsAsync();

    foreach (var collector in activeCollectors)
    {
        var lastLog = await _repository.GetFirstAsync(l =>
            l.CollectorDeviceId == collector.DeviceId,
            orderBy: l => l.OccurredAt,
            orderByOrderByType: OrderByType.Desc
        );

        if (lastLog == null || lastLog.OccurredAt < threshold)
        {
            // 创建采集器离线告警
            await _alarmService.CreateOfflineAlarmAsync(collector);
        }
    }
}
```

---

> **最后更新**：2026-06-04
> **源码位置**：`module/ast-voiceprint/Ast.Voiceprint.Domain/Entities/VoiceprintCollectorLogEntity.cs`