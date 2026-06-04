# Iec61850DataReportHandler — IEC61850数据上报处理器

## 概述

**Iec61850DataReportHandler** 负责将处理后的传感器数据上报到IEC61850服务端。它通过映射关系将点位数据转换为IEC61850协议格式，并按ICD文件分组并发上报。

### 触发事件
- **事件类型**: `ProcessedPointValueEventArgs` (Local Event)
- **触发场景**: 传感器数据处理完成，准备上报
- **事件源**: `PointValueEventHandler` 发布

### 核心职责
1. **映射关系查找** — 根据点位信息查找IEC61850映射配置
2. **数据转换** — 将点位数据转换为IEC61850上下文格式
3. **分组处理** — 按ICD文件分组，支持多设备并发上报
4. **协议适配** — 根据数据类型进行值格式转换

### 处理特点
- **条件触发**: 仅在启用数据上报时处理
- **批量并发**: 多个ICD文件分组并发处理
- **类型适配**: 自动适配Boolean、Int、Float、String等类型
- **容错机制**: 单个分组失败不影响其他分组

## 事件结构

### ProcessedPointValueEventArgs（输入）

```csharp
public class ProcessedPointValueEventArgs
{
    public MqPointValueDto Data { get; set; }
}

public class MqPointValueItemDto
{
    public DateTime Ts { get; set; }                   // 采样时间戳
    public string DeviceId { get; set; }              // 设备ID
    public string SensorKey { get; set; }             // 传感器标识
    public string Property { get; set; }              // 属性名
    public object Value { get; set; }                 // 值
}
```

### IecContext（内部转换）

```csharp
public class IecContext
{
    // 映射信息字段
    public string PointKey { get; set; }              // 点位键 device_id|sensor_key|property
    public string Reference { get; set; }            // IEC61850引用路径
    public string Host { get; set; }                  // 主机地址
    public string IcdFileName { get; set; }           // ICD文件名
    public Iec61850DataType DataType { get; set; }   // 数据类型
    public string Description { get; set; }          // 描述
    
    // 点位值字段
    public string DeviceId { get; set; }              // 设备ID
    public string SensorKey { get; set; }            // 传感器标识
    public string Property { get; set; }             // 属性名
    public object Value { get; set; }                // 值
    public DateTime Timestamp { get; set; }         // 时间戳
}
```

## 处理流程

```mermaid
flowchart TD
    A[ProcessedPointValueEventArgs] --> B{数据上报是否启用?}
    B -->|否| C[直接返回]
    B -->|是| D[遍历点位值列表]
    
    D --> E[构建 PointKey]
    E --> F[device_id|sensor_key|property]
    
    F --> G[查找映射关系]
    G --> H{映射是否存在?}
    H -->|否| I[跳过该点位]
    H -->|是| J[创建 IecContext]
    
    J --> K[设置映射信息字段]
    J --> L[设置点位值字段]
    
    K --> M[按 ICD 文件名分组]
    L --> M
    
    M --> N[并发处理各分组]
    N --> O[创建 IEC61850 处理器]
    O --> P[调用 Processor.ProcessAsync]
    
    P --> Q{处理是否成功?}
    Q -->|是| R[记录成功日志]
    Q -->|否| S[记录错误日志]
    
    R --> T{是否还有分组?}
    S --> T
    T -->|是| O
    T -->|否| U[处理完成]
    
    style B fill:#fff3e0
    style G fill:#e1f5fe
    style M fill:#fff9c4
    style P fill:#c8e6c9
    style U fill:#f3e5f5
```

### 关键处理步骤

#### 1. 配置检查
```csharp
if (!_options.Value.EnableDataReport || eventData.Data?.PointVals?.Any() != true)
    return;
```

#### 2. 映射关系查找
```csharp
foreach (var pointValue in eventData.Data.PointVals)
{
    // pointKey 格式: device_id|sensor_key|property
    var pointKey = $"{pointValue.DeviceId}|{pointValue.SensorKey}|{pointValue.Property}";
    var mapping = _mappingManager.GetMapping(pointKey);

    if (mapping != null)
    {
        contexts.Add(new IecContext { ... });
    }
}
```

#### 3. 按ICD文件分组
```csharp
var groups = contexts.GroupBy(c => c.IcdFileName);

var tasks = groups.Select(async group =>
{
    var processor = _processorFactory.CreateProcessor(group.Key);
    await processor.ProcessAsync(group.ToList());
});

await Task.WhenAll(tasks);
```

## 映射关系配置

### PointKey 格式

**传感器数据上报**:
```
格式: {device_id}|{sensor_key}|{property}
示例: device_001|temp_sensor|temperature
```

**告警数据上报**:
```
格式: {device_id}|{sensor_key}|{property}|{alarm_category_name}
示例: device_001|temp_sensor|temperature|温度过高
```

### IEC61850数据类型

```csharp
public enum Iec61850DataType
{
    Boolean,    // 布尔值
    Int,        // 整数
    Float,      // 浮点数
    String      // 字符串
}
```

### 映射配置来源

映射关系由 `IIec61850MappingManager` 管理，通常配置在数据库或配置文件中：

```csharp
public class Iec61850Mapping
{
    public string PointKey { get; set; }          // 点位键
    public string Reference { get; set; }          // IEC61850引用路径
    public string Host { get; set; }              // 主机地址
    public string IcdFileName { get; set; }       // ICD文件名
    public Iec61850DataType DataType { get; set; } // 数据类型
    public string Description { get; set; }        // 描述
}
```

## 依赖服务

| 服务接口 | 用途 | 核心方法 |
|---------|------|---------|
| `IIec61850MappingManager` | 映射关系管理 | `GetMapping` |
| `IIec61850ProcessorFactory` | 处理器工厂 | `CreateProcessor` |
| `IOptions<Iec61850Options>` | 配置选项 | `EnableDataReport` |

## 配置项

### IEC61850配置

```json
{
  "Iec61850": {
    "EnableDataReport": true,           // 是否启用数据上报
    "EnableAlarmReport": true,           // 是否启用告警上报
    "ModelFiles": [                     // ICD模型文件列表
      {
        "FileName": "device_1.icd",
        "FilePath": "/models/device_1.icd"
      }
    ]
  }
}
```

### Iec61850Options

```csharp
public class Iec61850Options
{
    public bool EnableDataReport { get; set; }    // 启用数据上报
    public bool EnableAlarmReport { get; set; }    // 启用告警上报
    public List<IecModelFile> ModelFiles { get; set; }  // ICD模型文件
}
```

## 错误处理

### 映射关系不存在
```csharp
if (mapping == null)
{
    _logger.LogDebug("未找到匹配的61850映射关系: PointKey={PointKey}", pointKey);
    continue;  // 跳过该点位
}
```

### 单个分组处理失败
```csharp
try
{
    var processor = _processorFactory.CreateProcessor(group.Key);
    await processor.ProcessAsync(group.ToList());
}
catch (Exception ex)
{
    _logger.LogError(ex, "处理61850数据上报失败: IcdFile={IcdFile}, ContextCount={Count}", 
        group.Key, group.Count());
    // 继续处理其他分组
}
```

### 整体事件处理失败
```csharp
catch (Exception ex)
{
    _logger.LogError(ex, "61850数据上报事件处理异常: PointCount={PointCount}", 
        eventData.Data?.PointVals?.Count ?? 0);
    // 不重新抛出异常，避免影响其他处理器
}
```

## 日志示例

### 正常流程
```
61850数据上报处理完成: ContextCount=12, GroupCount=2
```

### 映射未找到
```
未找到匹配的61850映射关系: PointKey=device_001|sensor_001|temperature
未找到匹配的61850映射关系: PointCount=15
```

### 处理失败
```
处理61850数据上报失败: IcdFile=device_1.icd, ContextCount=5
61850数据上报事件处理异常: PointCount=20
```

## 订阅关系

### 上游发布者
- `PointValueEventHandler` — 点位编排处理器

### 同级订阅者（同一事件）
- `ProcessedPointValueEventHandler` — 时序数据入库处理器
- `PatrolResultUpdateHandler` — 巡检结果更新处理器

## 性能特点

1. **条件触发**: 配置关闭时零开销
2. **批量并发**: 多ICD文件分组并发处理
3. **映射缓存**: 映射关系通常缓存在内存中
4. **容错设计**: 单点故障不影响整体上报

## 相关文档

- [Iec61850AlarmReportHandler.md](./Iec61850AlarmReportHandler.md) — IEC61850告警上报
- [Iec61850MappingManager.md](../../../ISAPI/Services/Iec61850MappingManager.md) — 映射管理器
- [EventDrivenPipeline.md](../EventDrivenPipeline.md) — 事件驱动链路总览

## 源码位置

```
module/ast-intellisub/Ast.IntelliSub.Application/EventHandlers/Iec61850DataReportHandler.cs
```
