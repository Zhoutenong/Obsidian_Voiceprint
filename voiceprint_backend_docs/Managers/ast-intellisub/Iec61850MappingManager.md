# Iec61850MappingManager — IEC61850映射管理器

## 基本信息

- **Manager名称**：`Iec61850MappingManager`
- **模块位置**：`module/ast-intellisub/Ast.IntelliSub.Application/Managers/`
- **接口实现**：`IIec61850MappingManager`
- **生命周期**：`ISingletonDependency` - 单例依赖
- **命名空间**：`Ast.IntelliSub.Application.Managers`

## Manager概述

Iec61850MappingManager 负责管理IEC 61850协议的点位映射关系。它从ICD（IED Capability Description）文件中解析映射配置，并提供映射关系的查询和管理功能。

## 核心职责

1. **映射关系初始化** - 从ICD文件加载映射配置
2. **映射查询** - 根据PointKey查询对应的映射关系
3. **映射重载** - 支持运行时重新加载映射配置
4. **统计日志** - 记录映射统计信息和服务器分布

## 依赖注入

| 依赖 | 职责 |
|------|------|
| `IIcdFileParser` | ICD文件解析器 |
| `ILogger<Iec61850MappingManager>` | 日志记录器 |
| `IOptions<Iec61850Options>` | IEC61850配置选项 |

## 核心数据结构

### ConcurrentDictionary映射存储

```csharp
private readonly ConcurrentDictionary<string, Iec61850Map> _mappings = new();
```

**键值对**：
- **Key**: `PointKey` - 点位键值（格式：`device_id|sensor_key|property|alarm_category_name`）
- **Value**: `Iec61850Map` - IEC61850映射对象

### Iec61850Map 映射对象

```csharp
public class Iec61850Map
{
    public string PointKey { get; set; }        // 映射键值
    public string Reference { get; set; }       // 61850引用路径
    public string Host { get; set; }            // 61850服务器地址
    public string IcdFileName { get; set; }    // ICD文件名
    public Iec61850DataType DataType { get; set; } // 数据类型
    public string Description { get; set; }    // 描述信息
}
```

## 核心方法

### 1. InitializeAsync - 初始化映射关系

**签名**：
```csharp
public async Task InitializeAsync()
```

**功能**：
从配置的ICD文件中加载映射关系到内存。

**执行流程**：

```
1. 清空现有映射
   │
   ├─ _mappings.Clear()
   │
2. 遍历启用的ModelFiles配置
   │
   ├─ foreach (var config in _options.Value.ModelFiles.Where(c => c.Enabled))
   │
   ├─ 确定ICD文件路径
   │  ├─ 如果 ModelFileDirectory 为空，使用默认路径 "Iec61850"
   │  └─ 组合完整路径：Path.Combine(modelFileDirectory, config.ModelFile)
   │
3. 检查文件存在性
   │
   ├─ 如果文件不存在
   │  ├─ 记录警告日志
   │  └─ continue 跳过
   │
4. 解析ICD文件
   │
   ├─ var maps = await _icdFileParser.ParseAsync(icdFilePath, config.Host)
   │
5. 添加映射关系到字典
   │
   ├─ foreach (var map in maps)
   │  ├─ 如果 TryAdd 成功 → totalMappings++
   │  └─ 如果失败（重复键） → 记录警告日志
   │
6. 记录加载日志
   │
   └─ _logger.LogInformation("加载ICD文件: {FileName}, 点位数量: {Count}, Host: {Host}")
   │
7. 输出统计信息
   │
   └─ LogMappingStatistics()
```

**配置示例**：
```json
{
  "Iec61850": {
    "ModelFileDirectory": "Iec61850",
    "ModelFiles": [
      {
        "ModelFile": "ied1_icd.xml",
        "Host": "192.168.1.100",
        "Enabled": true
      },
      {
        "ModelFile": "ied2_icd.xml",
        "Host": "192.168.1.101",
        "Enabled": true
      }
    ]
  }
}
```

### 2. GetMapping - 查询映射关系

**签名**：
```csharp
public Iec61850Map? GetMapping(string pointKey)
```

**功能**：
根据PointKey查询对应的映射关系。

**使用示例**：
```csharp
var pointKey = "device-001|temp01|temperature|温度告警";
var mapping = _mappingManager.GetMapping(pointKey);

if (mapping != null)
{
    // 使用映射信息
    var reference = mapping.Reference;      // 61850引用路径
    var host = mapping.Host;                  // 61850服务器地址
    var dataType = mapping.DataType;         // 数据类型
}
```

### 3. GetAllMappings - 获取所有映射

**签名**：
```csharp
public IReadOnlyDictionary<string, Iec61850Map> GetAllMappings()
```

**功能**：
返回所有映射关系的只读字典。

**使用场景**：
- 调试和诊断
- 映射关系导出
- 系统监控

### 4. ReloadAsync - 重新加载映射

**签名**：
```csharp
public async Task ReloadAsync()
```

**功能**：
重新加载IEC 61850映射关系，用于运行时更新配置。

**使用场景**：
- 配置文件更新后
- 新增ICD文件后
- 映射关系变更时

### 5. LogMappingStatistics - 记录映射统计

**签名**：
```csharp
private void LogMappingStatistics()
```

**功能**：
记录映射统计信息，包括数据类型分布和服务器分布。

**统计内容**：

```
IEC 61850映射统计信息:
  Boolean: 120 个点位
  Int: 85 个点位
  Float: 60 个点位
  String: 15 个点位

IEC 61850服务器分布:
  192.168.1.100: 150 个点位
  192.168.1.101: 130 个点位
```

## PointKey格式

### 告警上报的PointKey

```
{device_id}|{sensor_key}|{property}|{alarm_category_name}
```

**示例**：
```
device-001|temp01|temperature|温度告警
device-001|vib01|vibration|震动告警
```

**说明**：
- 由四部分组成，使用 `|` 分隔
- 最后一个部分是告警类别名称，用于区分同一点位的不同告警类型
- 用于在映射字典中查找对应的61850映射关系

## 数据类型

### Iec61850DataType 枚举

```csharp
public enum Iec61850DataType
{
    Boolean,  // 布尔类型
    Int,      // 整数类型
    Float,    // 浮点类型
    String    // 字符串类型
}
```

## 日志记录

### 信息日志 (LogInformation)

```csharp
// 初始化开始
"开始初始化IEC 61850映射关系..."

// 初始化完成
"IEC 61850映射关系初始化完成，总点位数量: {Count}"

// 加载ICD文件
"加载ICD文件: {FileName}, 点位数量: {Count}, Host: {Host}"

// 重新加载
"重新加载IEC 61850映射关系..."

// 统计信息
"IEC 61850映射统计信息:"
"  {DataType}: {Count} 个点位"
"IEC 61850服务器分布:"
"  {Host}: {Count} 个点位"
```

### 警告日志 (LogWarning)

```csharp
// 文件不存在
"ICD文件不存在: {FilePath}"

// 重复的点位键
"重复的点位键: {PointKey}, 文件: {FileName}"

// 加载失败
"加载ICD文件失败: {FileName}"
```

## 错误处理

### 异常捕获

```csharp
try
{
    var maps = await _icdFileParser.ParseAsync(icdFilePath, config.Host);
    // 处理映射...
}
catch (Exception ex)
{
    _logger.LogError(ex, "加载ICD文件失败: {FileName}", config.ModelFile);
    // 继续处理下一个文件
}
```

**设计决策**：
- 单个ICD文件加载失败不影响其他文件
- 记录错误日志，不抛出异常
- 保证系统的鲁棒性

## 性能优化

### 1. 并发字典

```csharp
private readonly ConcurrentDictionary<string, Iec61850Map> _mappings = new();
```

**好处**：
- 线程安全的字典操作
- 支持高并发查询
- 无需显式锁

### 2. 单例模式

```csharp
public class Iec61850MappingManager : IIec61850MappingManager, ISingletonDependency
```

**好处**：
- 整个应用共享一个映射字典
- 减少内存占用
- 提高查询性能

### 3. 初始化时机

映射关系在应用启动时初始化一次，避免运行时频繁解析ICD文件。

## 相关服务

- `IIcdFileParser` - ICD文件解析器
- `Iec61850ProcessorFactory` - 61850处理器工厂
- `Iec61850DataReportHandler` - 61850数据上报处理器

## 相关配置

### Iec61850Options

```csharp
public class Iec61850Options
{
    public string ModelFileDirectory { get; set; }  // ICD文件目录
    public List<ModelFileConfig> ModelFiles { get; set; }  // 模型文件列表
    public bool EnableDataReport { get; set; }          // 是否启用数据上报
    public bool EnableAlarmReport { get; set; }         // 是否启用告警上报
}

public class ModelFileConfig
{
    public string ModelFile { get; set; }     // ICD文件名
    public string Host { get; set; }          // 61850服务器地址
    public bool Enabled { get; set; }        // 是否启用
}
```

## 使用场景

### 1. 告警上报

```csharp
// 构建PointKey
var pointKey = $"{deviceId}|{sensorKey}|{property}|{alarmCategoryName}";

// 查询映射关系
var mapping = _mappingManager.GetMapping(pointKey);

if (mapping != null)
{
    // 使用映射信息上报告警
    await processor.ProcessAsync(mapping, alarmValue);
}
```

### 2. 数据上报

```csharp
// 构建PointKey（数据上报可能不包含告警类别）
var pointKey = $"{deviceId}|{sensorKey}|{property}";

// 查询映射关系
var mapping = _mappingManager.GetMapping(pointKey);

if (mapping != null)
{
    // 使用映射信息上报数据
    await processor.ProcessAsync(mapping, dataValue);
}
```

### 3. 运行时重载

```csharp
// 修改ICD文件或配置后
await _mappingManager.ReloadAsync();

// 映射关系自动更新，无需重启应用
```

## 设计特点

1. **单例模式** - 整个应用共享一个映射字典
2. **并发安全** - 使用ConcurrentDictionary支持高并发
3. **容错设计** - 单个文件加载失败不影响整体
4. **运行时重载** - 支持热更新映射配置
5. **详细日志** - 记录加载过程和统计信息
6. **性能优化** - 内存映射，查询速度极快

## 注意事项

1. **ICD文件路径** - 确保文件路径配置正确
2. **PointKey格式** - 必须严格按照格式构建
3. **告警类别名称** - 告警上报时PointKey包含告警类别名称
4. **重复键处理** - 后加载的映射会覆盖先加载的映射（使用TryAdd）
5. **重载时机** - 重载会影响所有正在进行的上报操作，建议在低峰期执行

---

> **最后更新**：2026-06-04  
> **源码位置**：`module/ast-intellisub/Ast.IntelliSub.Application/Managers/Iec61850MappingManager.cs`
> **接口定义**：`module/ast-intellisub/Ast.IntelliSub.Application.Contracts/IServices/IIec61850MappingManager.cs`
