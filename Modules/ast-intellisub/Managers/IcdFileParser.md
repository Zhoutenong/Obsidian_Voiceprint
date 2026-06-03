# IcdFileParser

**路径**: `module/ast-intellisub/Ast.IntelliSub.Application/Managers/IcdFileParser.cs`

**依赖**: `ITransientDependency`

## 概述

ICD (IED Capability Description) 文件解析器，负责从 IEC 61850 模型文件中提取点位映射关系。通过解析 XML 结构中的 DOI 元素，建立设备点位与 61850 对象引用的映射。

## 核心功能

### 1. ICD 文件解析

```csharp
public async Task<List<Iec61850Map>> ParseAsync(string icdFilePath, string host)
```

**职责**：
- 加载 ICD XML 文件
- 遍历所有 DOI (Data Object) 元素
- 提取符合格式的点位键
- 构建对象引用和数据类型
- 生成映射列表

**处理流程**：
```
XDocument.Load → Descendants(DOI) → 验证 PointKey → 
构建 Reference → 提取 DataType → 生成 Iec61850Map
```

### 2. 点位键验证

```csharp
private bool IsValidPointKey(string pointKey)
```

**格式验证**：
```
device_id|sensor_key|property
device_id|sensor_key|property|alarm_category_name
```

**正则表达式**：
```regex
^[a-zA-Z0-9_]+\|[a-zA-Z0-9_]+\|[a-zA-Z0-9_]+(\|[a-zA-Z0-9_]+)?$
```

### 3. 对象引用构建

```csharp
private (string reference, Iec61850DataType dataType) BuildReferenceAndDataType(XElement doi)
```

**职责**：
- 从 DOI 的父级元素构建对象引用路径
- 检测 DAI (Data Attribute) 确定数据类型
- 返回完整的引用路径和数据类型

**引用格式**：
```
LDInst/LLN0.DOName.DAName
GISSF60MONT1/SSWG1.UpTmpA.stVal
```

### 4. 数据类型检测

```csharp
private Iec61850DataType GetDefaultDataType(string daName)
```

**类型映射**：
| DA 名称 | 数据类型 | 说明 |
|---------|----------|------|
| `stVal`, `mag.f` | Float32 | 浮点数值 |
| `mag.i` | Int32 | 整数 |
| `boolVal` | Boolean | 布尔值 |
| `valWtr` | String | 字符串 |
| 其他 | Float32 | 默认 |

### 5. 描述提取

```csharp
private string GetDescription(XElement doi)
```

**职责**：
- 从 DOI 或 DAI 的 `desc` 属性提取中文描述
- 回退到使用名称作为描述

## XML 结构

### ICD 文件示例

```xml
<?xml version="1.0" encoding="UTF-8"?>
<SCL xmlns="http://www.iec.ch/61850/2003/SCL">
  <IED name="Device1">
    <AccessPoint name="S1">
      <Server>
        <LDevice inst="GISSF60MONT1">
          <LN0 lnType="LLN0" lnClass="LLN0">
            <DOI desc="device1|sensor1|temperature">
              <DAI name="stVal">
                <Val>25.5</Val>
              </DAI>
            </DOI>
            <DOI desc="device1|sensor1|humidity">
              <DAI name="mag">
                <DAI name="f">
                  <Val>60.2</Val>
                </DAI>
              </DAI>
            </DOI>
          </LN0>
          <LN lnType="SSWG1" lnClass="SSWG" inst="1">
            <DOI desc="device1|sensor1|status">
              <DAI name="stVal">
                <Val>1</Val>
              </DAI>
            </DOI>
          </LN>
        </LDevice>
      </Server>
    </AccessPoint>
  </IED>
</SCL>
```

### 解析结果

```csharp
[
  {
    "PointKey": "device1|sensor1|temperature",
    "Reference": "GISSF60MONT1/LLN0.temperature.stVal",
    "Host": "192.168.130.2:8080",
    "IcdFileName": "device1.icd",
    "DataType": "Float32",
    "Description": "温度"
  },
  {
    "PointKey": "device1|sensor1|humidity",
    "Reference": "GISSF60MONT1/LLN0.humidity.mag.f",
    "Host": "192.168.130.2:8080",
    "IcdFileName": "device1.icd",
    "DataType": "Float32",
    "Description": "湿度"
  }
]
```

## 命名空间

### SCL 命名空间

```csharp
const string SCL_NAMESPACE = "http://www.iec.ch/61850/2003/SCL";

// 查询时使用命名空间
var allDois = doc.Root!
    .Descendants("{http://www.iec.ch/61850/2003/SCL}DOI");
```

## 错误处理

### 文件加载失败

```csharp
try
{
    var doc = await Task.Run(() => XDocument.Load(icdFilePath));
}
catch (Exception ex)
{
    _logger.LogError(ex, "解析ICD文件失败: {FilePath}", icdFilePath);
    throw;
}
```

### 空点位键

```csharp
var pointKey = doi.Attribute("desc")?.Value ?? string.Empty;
if (string.IsNullOrEmpty(pointKey))
    continue;
```

### 格式验证失败

```csharp
if (!IsValidPointKey(pointKey))
    continue;  // 跳过不符合格式的点位
```

### 空引用

```csharp
var (reference, dataType) = BuildReferenceAndDataType(doi);
if (string.IsNullOrEmpty(reference))
    continue;
```

## 性能特性

### 文件加载

- 异步加载：`Task.Run(() => XDocument.Load(...))`
- 典型延迟：< 100ms（1MB 文件）

### 内存占用

- XDocument DOM：约 5-10x 文件大小
- 解析结果：约 100 bytes/映射

### 并发处理

- 每次调用独立解析
- 无共享状态（线程安全）

## 使用示例

### 解析单个文件

```csharp
var parser = new IcdFileParser(_logger);

var icdFilePath = "/path/to/device1.icd";
var host = "192.168.130.2:8080";

var maps = await parser.ParseAsync(icdFilePath, host);

foreach (var map in maps)
{
    _logger.LogInformation(
        "PointKey: {PointKey}, Reference: {Ref}, Type: {Type}",
        map.PointKey, map.Reference, map.DataType
    );
}
```

### 批量解析

```csharp
var parser = new IcdFileParser(_logger);
var allMaps = new List<Iec61850Map>();

foreach (var config in modelFiles)
{
    var icdFilePath = Path.Combine(modelDir, config.ModelFile);
    var maps = await parser.ParseAsync(icdFilePath, config.Host);
    allMaps.AddRange(maps);
}

_logger.LogInformation("总共解析映射数: {Count}", allMaps.Count);
```

## 辅助方法

### 获取测量值信息

```csharp
private XElement? GetMeasuredValueInfo(XElement dai)
```

查找包含 `mag` 或 `valWtr` 的 DAI 元素。

### 获取默认数据类型

```csharp
private Iec61850DataType GetDefaultDataType(string daName)
```

根据 DAI 名称推断数据类型。

## 日志示例

### 成功日志

```
解析ICD文件完成: device1.icd, 点位数量: 150
解析ICD文件完成: device2.icd, 点位数量: 200
```

### 错误日志

```
解析ICD文件失败: /path/to/invalid.icd
跳过无效点位键: device1||temperature
```

## 扩展点

### 自定义点位键验证

继承并重写 `IsValidPointKey` 方法：

```csharp
public class CustomIcdFileParser : IcdFileParser
{
    protected override bool IsValidPointKey(string pointKey)
    {
        // 自定义验证逻辑
        return Regex.IsMatch(pointKey, @"^[a-z0-9_]+\|.*$");
    }
}
```

### 自定义数据类型映射

重写 `GetDefaultDataType` 方法添加新类型支持。

## 相关文档

- [Iec61850MappingManager](./Iec61850MappingManager.md) - 映射管理器
- [IEC61850 数据上报](../../Features/Iec61850DataReport.md)
- [IEC61850 协议规范](https://iec.ch/61850/)
