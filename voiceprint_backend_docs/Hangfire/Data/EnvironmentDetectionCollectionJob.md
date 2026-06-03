# 环境检测采集任务 (EnvironmentDetectionCollectionJob)

## 概述

定期获取环境检测和行為检测分析点位，调用AI识别服务进行图像分析，通过事件总线上报识别结果。支持重复值过滤、批量识别和多区域检测。

**职责：**
- 获取环境检测和行為检测点位配置
- 调用摄像头进行截图
- 批量调用AI识别服务
- 过滤重复值上报
- 通过事件总线上报识别结果

## 任务配置

### appsettings.json 配置

```json
{
  "EnvironmentDetectionCollection": {
    "Enabled": true,
    "CronExpression": "*/30 * * * * ?",
    "RecognitionTimeoutMs": 60000,
    "EnableDuplicateValueFilter": true,
    "DuplicateFilterTimeWindowMinutes": 10,
    "MaxDuplicateReportsInWindow": 1
  }
}
```

### 配置选项说明

| 选项 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `Enabled` | bool | true | 是否启用任务 |
| `CronExpression` | string | "*/30 * * * * ?" | Cron表达式（每30秒） |
| `RecognitionTimeoutMs` | int | 60000 | 识别超时时间（毫秒） |
| `EnableDuplicateValueFilter` | bool | true | 是否启用重复值过滤 |
| `DuplicateFilterTimeWindowMinutes` | int | 10 | 重复值过滤时间窗口（分钟） |
| `MaxDuplicateReportsInWindow` | int | 1 | 时间窗口内允许最大重复上报次数 |

## Cron 表达式说明

| 表达式 | 说明 |
|--------|------|
| `*/30 * * * * ?` | 每30秒执行（默认） |
| `*/15 * * * * ?` | 每15秒执行 |
| `*/60 * * * * ?` | 每60秒执行 |
| `0 * * * * ?` | 每分钟执行 |
| `0 */5 * * * ?` | 每5分钟执行 |

## 执行流程

```
获取环境检测点位 → 按设备+预置位分组 → 截图 → 批量AI识别 → 重复值过滤 → 上报数据
```

### 详细步骤

1. **检查任务启用状态**
   - 读取 `Enabled` 配置
   - 禁用时跳过执行

2. **获取环境检测点位**
   ```csharp
   var input = new RtMonitoredPointBindingGetListInputDto
   {
       AlgorithmType = AlgorithmTypeEnum.EnvironmentDetectionAnalysis,
       IncludeUnbound = false,
       DataSrcType = DataSrcTypeEnum.RealTimeMonitoring
   };
   ```
   **注意：** 同时获取 `EnvironmentDetectionAnalysis` 和 `BehaviorDetectionAnalysis` 两种类型的点位

3. **分组处理点位**
   - 按 `设备ID + 预置位号` 组合分组
   - 同一分组的点位使用同一张截图

4. **截图获取**
   - 调用 `IMediaService.CaptureAndSaveSnapshotAsync`
   - 获取当前预置位的图片

5. **构造AI识别任务**
   - 将点位配置转换为AI识别任务
   - 包含算法ID、识别区域、参数

6. **批量AI识别**
   - 调用 `IAIRecognitionService.BatchRecognizeAsync`
   - 一次调用处理所有识别区域

7. **转换识别结果**
   - 将AI结果转换为点位值数据
   - 应用重复值过滤

8. **上报数据**
   - 通过事件总线发布
   - 包含识别结果和图片URL

## 数据采集逻辑

### 点位类型

任务同时处理两种类型的点位：

| 类型 | 枚举值 | 说明 |
|------|--------|------|
| 环境检测 | `EnvironmentDetectionAnalysis` | 检测环境状态（如烟雾、积水） |
| 行为检测 | `BehaviorDetectionAnalysis` | 检测人员行为（如入侵、徘徊） |

### 截图逻辑

```csharp
private async Task<string?> CaptureSnapshotAsync(string nvrId, string deviceId)
{
    var input = new CaptureAndSaveSnapshotInputDto
    {
        NvrId = nvrId,
        CameraId = deviceId
    };

    var snapshotPath = await _mediaService.CaptureAndSaveSnapshotAsync(input);
    return snapshotPath;
}
```

### AI识别任务构造

```csharp
var task = new AIBatchRecognitionTaskDto
{
    TaskId = point.AstPointId.ToString()!,
    AlgorithmId = point.AlgorithmId!.Value,
    AlgorithmKey = point.AlgorithmKey,
    TaskName = point.AstPointName ?? $"环境检测-{point.MonitoredPointName}",
    RecognitionArea = point.RecArea,  // JSON格式的识别区域
    Params = null
};
```

### 批量识别调用

```csharp
var aiResults = await _aiRecognitionService.BatchRecognizeAsync(snapshotPath, aiTasks);
```

**优点：**
- 一次调用处理多个识别区域
- 减少图片加载次数
- 提高识别效率

## 重复值过滤

### 过滤机制

防止在短时间内重复上报相同的值，减少无效告警和数据量。

### 过滤参数

```csharp
DuplicateFilterTimeWindowMinutes = 10  // 10分钟时间窗口
MaxDuplicateReportsInWindow = 1        // 允许上报1次
```

### 过滤逻辑

```csharp
if (_options.EnableDuplicateValueFilter)
{
    var shouldReport = await _pointValueCacheService.ShouldReportValueAsync(
        pointId,
        value,
        _options.DuplicateFilterTimeWindowMinutes,
        _options.MaxDuplicateReportsInWindow
    );

    if (!shouldReport)
    {
        Logger.LogDebug("重复值过滤: 跳过上报点位 {PointId} 的重复值 {Value}", pointId, value);
        continue;
    }
}
```

### 缓存清理

```csharp
// 清理过期的缓存记录
var expiredBefore = reportTime.AddMinutes(-_options.DuplicateFilterTimeWindowMinutes * 2);
await _pointValueCacheService.CleanExpiredRecordsAsync(expiredBefore);
```

## 数据上报

### 事件总线格式

```csharp
var eventArgs = new PointValueEventArgs
{
    Type = "environment_detection_analysis",
    Data = new MqPointValueDto
    {
        PointVals = environmentDetectionData
    }
};

await _localEventBus.PublishAsync(eventArgs);
```

### 点位值数据结构

```csharp
public class MqPointValueItemDto
{
    public string PointId { get; set; }        // 点位ID
    public DateTime Ts { get; set; }           // 采集时间
    public string DeviceId { get; set; }        // 设备ID
    public string SensorKey { get; set; }      // 传感器Key
    public string Property { get; set; }       // 属性
    public string GroupId { get; set; }        // 分组ID
    public object Value { get; set; }          // 识别结果值
    public int ValueType { get; set; }         // 值类型
    public string ExtInfo { get; set; }        // 扩展信息（JSON）
    public AlarmLevelEnum AlarmLevel { get; set; } // 告警级别
}
```

### ExtInfo 内容

```json
{
  "url": "标注结果图片URL或原始截图"
}
```

## 设备连接

### 媒体服务接口

任务通过 `IMediaService` 获取摄像头截图：

| 方法 | 说明 |
|------|------|
| `CaptureAndSaveSnapshotAsync` | 截图并保存到本地 |

### AI识别服务接口

任务通过 `IAIRecognitionService` 进行图像识别：

| 方法 | 说明 |
|------|------|
| `BatchRecognizeAsync` | 批量识别图片 |

### 设备要求

- 支持RTSP/ONVIF协议的摄像头
- 配置预置位
- 配置识别区域
- 关联AI算法

## 依赖服务

- [[IRealtimeMonitoringPointService]] - 实时监测点位服务
- [[IAIRecognitionService]] - AI识别服务
- [[IMediaService]] - 媒体服务
- [[IPointValueCacheService]] - 点位值缓存服务
- [[ILocalEventBus]] - 本地事件总线

## 相关任务

- [[InfraredTemperatureCollectionJob]] - 红外测温采集任务
- [[PatrolJobManager]] - 巡检任务管理
- [[PointValueCacheCleanupJob]] - 点位值缓存清理任务

## 监控和日志

### 日志级别

```csharp
Logger.LogInformation("开始执行 [EnvironmentDetectionCollectionJob] 环境检测分析数据采集任务...");
Logger.LogInformation("找到 {Count} 个环境检测分析点位，开始采集数据", count);
Logger.LogInformation("监测点位组 {Name} 识别并上报了 {Count} 个环境检测结果", name, count);
Logger.LogWarning("无法获取监测点位组的截图，跳过AI识别");
Logger.LogWarning("AI识别服务未返回有效结果");
Logger.LogDebug("重复值过滤: 跳过上报点位 {PointId} 的重复值 {Value}");
Logger.LogError(ex, "环境检测分析数据采集任务执行失败");
```

### Hangfire Dashboard

访问路径：`/hangfire`

查看任务执行历史、状态和性能指标。

### 监控指标

- 处理点位数量
- 成功识别数量
- 重复值过滤数量
- AI识别成功率
- 平均执行时间
- 截图失败率

## 异常处理

### 截图失败

```csharp
var snapshotPath = await _mediaService.CaptureAndSaveSnapshotAsync(input);
if (string.IsNullOrEmpty(snapshotPath))
{
    Logger.LogWarning("截图失败: NvrId={NvrId}, DeviceId={DeviceId}", nvrId, deviceId);
    return null;
}
```

### AI识别失败

```csharp
if (aiResult == null || aiResult.IsUnrecognizable)
{
    Logger.LogWarning("环境检测识别失败: PointId={PointId}, Error={Error}",
        point.AstPointId, aiResult?.ErrorMessage);
    continue;
}
```

### 参数缺失

```csharp
if (string.IsNullOrEmpty(nvrId) || string.IsNullOrEmpty(deviceId))
{
    Logger.LogWarning("监测点位组缺少NVR或设备ID，跳过处理");
    return;
}
```

### 识别超时

```csharp
// 配置超时时间
RecognitionTimeoutMs = 60000  // 60秒

// AI识别服务内部处理超时
```

## 注意事项

### 并发控制

```csharp
[DisableConcurrentExecution(timeoutInSeconds: 3 * 60)]
```
- 防止任务重叠执行
- 超时时间：3分钟
- 前一个任务运行时，后续调度将被跳过

### 分组策略

- 按 `设备ID + 预置位号` 分组
- 同一分组的点位共享同一张截图
- 减少 AI 识别调用次数

### GroupId 生成

```csharp
// 按 MonitoredPointId 分组，相同的 MonitoredPointId 使用相同的 GroupId
var pointsByMonitoredPointId = pointGroup
    .Where(x => x.AstPointId.HasValue)
    .GroupBy(x => x.MonitoredPointId)
    .ToList();

foreach (var monitoredPointGroup in pointsByMonitoredPointId)
{
    var groupId = Guid.NewGuid().ToString();
    // 同一分组内的点位使用相同的 GroupId
}
```

### 重复值过滤配置

**建议配置：**

| 场景 | 时间窗口 | 最大次数 | 说明 |
|------|---------|---------|------|
| 高频监控 | 5分钟 | 1次 | 快速响应，避免重复 |
| 常规监控 | 10分钟 | 1次 | 平衡响应和重复 |
| 低频监控 | 30分钟 | 1次 | 减少无效数据 |
| 不启用 | - | - | 保留所有数据 |

### 性能优化

- 批量识别：一次调用处理多个区域
- 重复值过滤：减少无效数据上报
- 缓存清理：定期清理过期缓存
- 分组处理：减少截图和识别次数

## 识别结果处理

### 值类型

```csharp
public object Value { get; set; }  // 支持多种类型
```

常见值类型：
- `string` - 文本描述（如"正常"、"异常"）
- `int` - 数量统计（如人数）
- `bool` - 状态判断（如true/false）
- `double` - 数值测量（如温度、湿度）

### 告警级别

```csharp
AlarmLevelEnum AlarmLevel = AlarmLevelEnum.Normal; // 默认正常级别
```

**注意：** 具体告警级别由告警模块根据阈值判断，采集任务统一设置为 `Normal`。

### 识别图片

```csharp
extInfoUrl = aiResult.RecognitionImageUrl ?? imagePath;
```

优先使用标注结果图片，如果不存在则使用原始截图。

## 相关文档

- [[IAIRecognitionService]] - AI识别服务文档
- [[IMediaService]] - 媒体服务文档
- [[IPointValueCacheService]] - 点位值缓存服务文档
- [[InfraredTemperatureCollectionJob]] - 红外测温采集任务文档
- [[RtMonitoredPointBinding]] - 实时监测点位绑定文档

## 示例配置

### 完整配置示例

```json
{
  "EnvironmentDetectionCollection": {
    "Enabled": true,
    "CronExpression": "*/30 * * * * ?",
    "RecognitionTimeoutMs": 60000,
    "EnableDuplicateValueFilter": true,
    "DuplicateFilterTimeWindowMinutes": 10,
    "MaxDuplicateReportsInWindow": 1
  }
}
```

### 高频采集配置

```json
{
  "EnvironmentDetectionCollection": {
    "Enabled": true,
    "CronExpression": "*/15 * * * * ?",
    "DuplicateFilterTimeWindowMinutes": 5
  }
}
```

### 低频采集配置

```json
{
  "EnvironmentDetectionCollection": {
    "Enabled": true,
    "CronExpression": "0 */5 * * * ?",
    "DuplicateFilterTimeWindowMinutes": 30
  }
}
```

### 不启用重复值过滤

```json
{
  "EnvironmentDetectionCollection": {
    "Enabled": true,
    "CronExpression": "*/30 * * * * ?",
    "EnableDuplicateValueFilter": false
  }
}
```

## 与巡检任务的区别

| 特性 | 环境检测采集任务 | 巡检任务 |
|------|----------------|---------|
| 触发方式 | 定时触发 | 手动/定时触发 |
| 执行路线 | 所有配置点位 | 按巡检路线执行 |
| 截图策略 | 每组一张 | 每个点位一张 |
| 识别调用 | 按组批量 | 逐个调用 |
| 结果上报 | 立即上报 | 汇总后上报 |
| 重复过滤 | 支持 | 不支持 |

## 最佳实践

1. **合理配置执行频率**
   - 根据监控需求调整
   - 避免过于频繁导致性能问题
   - 考虑AI服务处理能力

2. **启用重复值过滤**
   - 减少无效数据上报
   - 降低告警频率
   - 提高系统效率

3. **分组策略优化**
   - 合理配置预置位
   - 同一场景的点位使用相同预置位
   - 减少截图次数

4. **监控识别成功率**
   - 定期检查AI识别结果
   - 分析识别失败原因
   - 优化识别区域配置

5. **日志分析**
   - 关注截图失败率
   - 关注识别超时情况
   - 关注重复值过滤效果
