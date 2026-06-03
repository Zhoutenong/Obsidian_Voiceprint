# Iec61850MappingManager

**路径**: `module/ast-intellisub/Ast.IntelliSub.Application/Managers/Iec61850MappingManager.cs`

**依赖**: `ISingletonDependency`

## 概述

IEC 61850 映射管理器，负责从 ICD 模型文件中解析点位映射关系并提供运行时查询。支持动态加载和热重载映射配置，是 IEC61850 数据上报的核心组件。

## 核心功能

### 1. 映射初始化

```csharp
public async Task InitializeAsync()
```

**职责**：
- 清空现有映射缓存
- 遍历启用的模型文件配置
- 调用 `IcdFileParser` 解析 ICD 文件
- 构建点位键 → 映射信息的字典
- 记录初始化统计信息

**配置来源**：
```json
{
  "Iec61850": {
    "ModelFileDirectory": "Iec61850",
    "ModelFiles": [
      {
        "ModelFile": "device1.icd",
        "Host": "192.168.1.100:8080",
        "Processor": "DefaultProcessor",
        "Enabled": true
      }
    ]
  }
}
```

### 2. 映射查询

```csharp
public Iec61850Map? GetMapping(string pointKey)
```

**职责**：
- 根据点位键（PointKey）查询映射信息
- 返回 IEC 61850 对象引用、主机地址等元数据

**点位键格式**：
```
device_id|sensor_key|property
device_id|sensor_key|property|alarm_category_name
```

### 3. 批量查询

```csharp
public IReadOnlyDictionary<string, Iec61850Map> GetAllMappings()
```

**职责**：
- 返回所有映射的只读字典副本
- 用于批量操作和调试

### 4. 热重载

```csharp
public async Task ReloadAsync()
```

**职责**：
- 重新加载映射配置
- 无需重启应用

## 数据结构

### Iec61850Map

```csharp
public class Iec61850Map
{
    public string PointKey { get; set; }          // 点位键
    public string Reference { get; set; }         // 61850对象引用
    public string Host { get; set; }              // 服务器地址
    public string IcdFileName { get; set; }       // ICD文件名
    public Iec61850DataType DataType { get; set; } // 数据类型
    public string Description { get; set; }       // 中文描述
}
```

### 数据类型枚举

- `Boolean` - 布尔值
- `Int8` / `Int16` / `Int32` / `Int64` - 整数
- `Float32` / `Float64` - 浮点数
- `String` - 字符串

## 协作对象

### IcdFileParser

ICD 文件解析器，负责从 XML 文件中提取映射关系。

**调用关系**：
```
Iec61850MappingManager.InitializeAsync()
    ↓
IcdFileParser.ParseAsync(icdFilePath, host)
    ↓
返回 List<Iec61850Map>
```

### 应用模块启动

```csharp
public override async Task OnApplicationInitializationAsync()
{
    await _iec61850MappingManager.InitializeAsync();
}
```

## 使用示例

### 事件处理器中使用

```csharp
public class Iec61850DataReportHandler : ILocalEventHandler
{
    private readonly IIec61850MappingManager _mappingManager;
    
    public async Task HandleEventAsync(EntityCreatedEventData<PointData> eventData)
    {
        var pointKey = $"{data.DeviceId}|{data.SensorKey}|{data.Property}";
        var map = _mappingManager.GetMapping(pointKey);
        
        if (map != null)
        {
            // 使用 map.Reference 和 map.Host 上报数据
            await _iec61850Api.ReportDataAsync(map.Host, map.Reference, data.Value);
        }
    }
}
```

### 批量查询

```csharp
var allMappings = _mappingManager.GetAllMappings();
foreach (var kvp in allMappings)
{
    _logger.LogInformation("PointKey: {Key}, Reference: {Ref}", 
        kvp.Key, kvp.Value.Reference);
}
```

## 性能特性

### 并发安全

使用 `ConcurrentDictionary` 存储映射，支持多线程并发访问。

### 内存效率

- 单例模式，全局共享
- 映射缓存常驻内存
- 典型配置：数百条映射，内存占用 < 1MB

### 查询性能

- `O(1)` 哈希查找
- 典型延迟：< 1μs

## 错误处理

### 文件不存在

```csharp
if (!File.Exists(icdFilePath))
{
    _logger.LogWarning("ICD文件不存在: {FilePath}", icdFilePath);
    continue;  // 跳过该文件
}
```

### 重复点位键

```csharp
if (!_mappings.TryAdd(map.PointKey, map))
{
    _logger.LogWarning("重复的点位键: {PointKey}", map.PointKey);
}
```

### 解析失败

```csharp
catch (Exception ex)
{
    _logger.LogError(ex, "解析ICD文件失败: {FilePath}", icdFilePath);
    // 继续处理其他文件
}
```

## 配置示例

### appsettings.json

```json
{
  "Iec61850": {
    "ModelFileDirectory": "Iec61850",
    "ModelFiles": [
      {
        "ModelFile": "gateway_device1.icd",
        "Host": "192.168.130.2:8080",
        "Processor": "DefaultProcessor",
        "Enabled": true
      },
      {
        "ModelFile": "gateway_device2.icd",
        "Host": "192.168.130.3:8080",
        "Processor": "CustomProcessor",
        "Enabled": false
      }
    ]
  }
}
```

## 监控指标

### 初始化日志

```
开始初始化IEC 61850映射关系...
解析ICD文件完成: gateway_device1.icd, 点位数量: 150
重复的点位键: device1|sensor1|temperature
IEC 61850映射关系初始化完成，总映射数: 298，重复数: 1
```

## 相关文档

- [IcdFileParser](./IcdFileParser.md) - ICD 文件解析器
- [IEC61850 数据上报](../../Features/Iec61850DataReport.md)
- [IEC61850 告警上报](../../Features/Iec61850AlarmReport.md)
