# Iec61850AlarmReportHandler — IEC61850告警上报事件处理器

## 基本信息

- **处理器名称**：`Iec61850AlarmReportHandler`
- **模块位置**：`module/ast-intellisub/Ast.IntelliSub.Application/EventHandlers/`
- **事件类型**：`AlarmRecordItemCreatedEventArgs` (ILocalEventHandler)
- **生命周期**：`ITransientDependency` - 瞬态依赖

## 处理器概述

Iec61850AlarmReportHandler 负责将告警记录项事件上报到IEC 61850服务端。当新的告警记录项创建时，该处理器会查找对应的61850映射关系，并根据映射配置将告警数据推送到61850系统。

## 核心职责

1. **告警映射查询** - 查找告警点位的61850映射关系
2. **告警值转换** - 根据数据类型转换告警值
3. **分组上报** - 按ICD文件名分组进行并行上报
4. **配置控制** - 支持通过配置开关启用/禁用告警上报

## 依赖注入

| 依赖 | 职责 |
|------|------|
| `IIec61850MappingManager` | 61850映射管理器 |
| `IIec61850ProcessorFactory` | 61850处理器工厂 |
| `IAstPointService` | 点位服务 |
| `ILogger<Iec61850AlarmReportHandler>` | 日志记录器 |
| `IOptions<Iec61850Options>` | 61850配置选项 |

## 事件数据结构

### AlarmRecordItemCreatedEventArgs

```csharp
public class AlarmRecordItemCreatedEventArgs
{
    public AlarmRecordItemEntity AlarmRecordItem { get; set; }  // 告警记录项
}
```

## 配置选项

### Iec61850Options

```csharp
public class Iec61850Options
{
    public bool EnableAlarmReport { get; set; }  // 是否启用告警上报
    // ... 其他配置项
}
```

## 处理流程

```
1. 检查配置开关
   │
   ├─ 如果未启用告警上报
   │  └─ 直接返回
   │
2. 查询点位信息
   │
   ├─ 通过 AstPointId 查询点位
   │
   ├─ 如果点位不存在
   │  ├─ 记录警告日志
   │  └─ 直接返回
   │
3. 查找映射关系
   │
   ├─ 构建PointKey: device_id|sensor_key|property|alarm_category_name
   │
   ├─ 通过映射管理器查找映射
   │
   ├─ 如果映射不存在
   │  └─ 直接返回
   │
4. 转换告警值
   │
   ├─ 根据告警级别获取告警值
   │  ├─ AlarmLevel = 1 → Value = 1
   │  ├─ AlarmLevel = 2 → Value = 2
   │  ├─ AlarmLevel = 3 → Value = 3
   │  └─ 其他 → Value = 0
   │
   ├─ 根据DataType转换数据类型
   │  ├─ Boolean → alarmValue > 0
   │  ├─ Int → alarmValue
   │  ├─ String → alarmMessage
   │  └─ Float → (float)alarmValue
   │
5. 创建上报上下文
   │
   ├─ 映射信息字段
   │  ├─ PointKey
   │  ├─ Reference
   │  ├─ Host
   │  ├─ IcdFileName
   │  ├─ DataType
   │  └─ Description
   │
   ├─ 点位值字段
   │  ├─ DeviceId
   │  ├─ SensorKey
   │  ├─ Property
   │  ├─ Value
   │  └─ Timestamp
   │
   └─ 告警相关字段
      ├─ AlarmCategoryName
      ├─ AlarmContent
      └─ AlarmLevel
   │
6. 按ICD文件名分组处理
   │
   ├─ 按IcdFileName分组
   │
   ├─ 并行处理每个分组
   │  ├─ 创建对应的Processor
   │  ├─ 调用ProcessAsync
   │  └─ 捕获处理异常
   │
7. 记录成功日志
   │
   └─ 输出：Reference, AlarmLevel, DataType, PointKey
```

## 核心方法

### HandleEventAsync - 处理告警上报事件

**签名**：
```csharp
public async Task HandleEventAsync(AlarmRecordItemCreatedEventArgs eventData)
```

**功能**：
处理告警记录项创建事件，将告警上报到61850系统。

### GetAlarmValue - 获取告警值

**签名**：
```csharp
private int GetAlarmValue(int alarmLevel)
```

**功能**：
根据告警级别获取告警值。

**转换规则**：
```csharp
return alarmLevel switch
{
    1 => 1, // 轻微告警
    2 => 2, // 重要告警  
    3 => 3, // 紧急告警
    _ => 0  // 无告警
};
```

## PointKey构建规则

### 告警上报的PointKey

```
{device_id}|{sensor_key}|{property}|{alarm_category_name}
```

**示例**：
```
device-001|temp01|temperature|温度告警
```

**说明**：
- PointKey由四个部分组成，使用"|"分隔
- 最后一个部分是告警类别名称，用于区分同一点位的不同告警类型
- 这个PointKey用于在映射管理器中查找对应的61850映射关系

## 数据类型转换

### DataType对应关系

| DataType | 转换逻辑 | 示例 |
|----------|---------|------|
| Boolean | `alarmValue > 0` | `alarmValue = 1` → `true` |
| Int | `alarmValue` | `alarmValue = 2` → `2` |
| String | `alarmMessage` | `"温度告警: 温度超过阈值"` |
| Float | `(float)alarmValue` | `alarmValue = 3` → `3.0` |

### 告警消息格式

```csharp
var alarmMessage = $"{alarmItem.AlarmCategoryName}: {alarmItem.AlarmContent}";
```

**示例**：
```
温度告警: 温度超过80℃
震动告警: 震动频率异常
```

## 上报上下文结构

### IecContext

```csharp
public class IecContext
{
    // 映射信息字段
    public string PointKey { get; set; }           // 映射键值
    public string Reference { get; set; }          // 61850引用
    public string Host { get; set; }               // 主机地址
    public string IcdFileName { get; set; }        // ICD文件名
    public Iec61850DataType DataType { get; set; } // 数据类型
    public string Description { get; set; }        // 描述信息
    
    // 点位值字段
    public string DeviceId { get; set; }          // 设备ID
    public string SensorKey { get; set; }         // 传感器键值
    public string Property { get; set; }           // 属性名称
    public object Value { get; set; }             // 转换后的值
    public DateTime Timestamp { get; set; }        // 时间戳
    
    // 告警相关字段
    public string AlarmCategoryName { get; set; }  // 告警类别名称
    public string AlarmContent { get; set; }       // 告警内容
    public int AlarmLevel { get; set; }            // 告警级别
}
```

## 分组并行处理

### 按ICD文件名分组

```csharp
// 按ICD文件名分组处理（告警通常只有一个，但保持一致的处理逻辑）
var groups = contexts.GroupBy(c => c.IcdFileName);

var tasks = groups.Select(async group =>
{
    try
    {
        var processor = _processorFactory.CreateProcessor(group.Key);
        await processor.ProcessAsync(group.ToList());
    }
    catch (Exception ex)
    {
        _logger.LogError(ex, "处理61850告警上报失败: IcdFile={IcdFile}, PointKey={PointKey}", 
            group.Key, pointKey);
    }
});

await Task.WhenAll(tasks);
```

**说明**：
- 按ICD文件名分组，确保同一ICD文件的告警由同一个Processor处理
- 使用Task.WhenAll实现并行处理
- 每个分组独立捕获异常，不影响其他分组

## 日志记录

### 信息日志 (LogInformation)

```csharp
// 上报成功
"61850告警上报成功: Reference={Reference}, AlarmLevel={AlarmLevel}, DataType={DataType}, PointKey={PointKey}"
```

### 警告日志 (LogWarning)

```csharp
// 点位不存在
"未找到AstPoint: AstPointId={AstPointId}"

// 映射不存在（已注释，通常不需要记录）
//"未找到告警的61850映射关系: PointKey={PointKey}"
```

### 错误日志 (LogError)

```csharp
// 分组处理失败
"处理61850告警上报失败: IcdFile={IcdFile}, PointKey={PointKey}"

// 整体处理失败
"61850告警上报失败: AlarmId={AlarmId}, AstPointId={AstPointId}"
```

## 错误处理

### 异常处理策略

```csharp
try
{
    // 处理逻辑
}
catch (Exception ex)
{
    _logger.LogError(ex, "61850告警上报失败: AlarmId={AlarmId}, AstPointId={AstPointId}", 
        eventData.AlarmRecordItem.Id, eventData.AlarmRecordItem.AstPointId);
    // 不抛出异常，避免影响主流程
}
```

**设计决策**：
- 告警上报失败不影响主业务流程
- 记录详细错误日志便于排查
- 分组级别的异常不影响其他分组

## 配置控制

### 启用/禁用告警上报

```csharp
if (!_options.Value.EnableAlarmReport) return;
```

**appsettings.json配置**：
```json
{
  "Iec61850": {
    "EnableAlarmReport": true,
    "EnableDataReport": true
  }
}
```

## 相关实体

- [[AlarmRecordItemEntity]] - 告警记录项实体
- [[AstPointEntity]] - 点位实体

## 相关服务

- [[IIec61850MappingManager]] - 61850映射管理器
- [[IIec61850ProcessorFactory]] - 61850处理器工厂
- [[IAstPointService]] - 点位服务

## 相关枚举

- [[Iec61850DataType]] - 61850数据类型枚举
- [[AlarmLevelEnum]] - 告警级别枚举

## 使用场景

### 1. 温度告警上报

当温度超过阈值时：
```csharp
alarmItem.AlarmCategoryName = "温度告警";
alarmItem.AlarmContent = "温度超过80℃";
alarmItem.AlarmLevel = AlarmLevelEnum.Medium; // 2

// 转换后
value = 2; // 或 true (如果DataType是Boolean)
alarmMessage = "温度告警: 温度超过80℃";
```

### 2. 震动告警上报

当震动频率异常时：
```csharp
alarmItem.AlarmCategoryName = "震动告警";
alarmItem.AlarmContent = "震动频率超过50Hz";
alarmItem.AlarmLevel = AlarmLevelEnum.High; // 3

// 转换后
value = 3;
alarmMessage = "震动告警: 震动频率超过50Hz";
```

### 3. 多个告警并行上报

同一时刻产生多个告警：
```csharp
// 按ICD文件名分组
Group1 (ICD_A): 告警1, 告警2 → Processor A处理
Group2 (ICD_B): 告警3 → Processor B处理
// 并行执行，互不影响
```

## 设计特点

1. **配置控制** - 支持开关控制告警上报功能
2. **映射查询** - 通过PointKey查找映射关系
3. **类型转换** - 根据DataType智能转换告警值
4. **分组并行** - 按ICD文件名分组并行处理
5. **容错设计** - 异常不影响主流程和其他分组
6. **详细日志** - 记录上报的详细信息

## 性能考虑

1. **早期返回** - 未启用或找不到映射时直接返回
2. **并行处理** - 多个ICD文件的告警并行上报
3. **异常隔离** - 分组级别异常不影响其他分组

## 注意事项

1. **PointKey唯一性** - 告警的PointKey包含告警类别名称，确保同一点位的不同告警类型可以映射到不同的61850节点
2. **告警级别映射** - 告警级别映射到整数值（0-3），根据DataType进行相应转换
3. **配置开关** - 生产环境可以通过配置关闭告警上报，避免影响61850系统
4. **映射关系** - 需要提前配置好告警点位的61850映射关系

---

> **最后更新**：2026-06-04  
> **源码位置**：`module/ast-intellisub/Ast.IntelliSub.Application/EventHandlers/Iec61850AlarmReportHandler.cs`
