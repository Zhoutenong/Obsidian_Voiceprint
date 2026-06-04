# AlgorithmEntity — 识别算法实体

## 基本信息

- **实体名称**：`AlgorithmEntity`
- **数据库表**：`ast_algorithm`
- **模块位置**：`module/ast-intellisub/Ast.IntelliSub.Domain/Entities/Monitoring/`
- **继承关系**：`CreationAuditedEntity<Guid>`

## 实体说明

识别算法实体用于定义系统中可用的AI识别算法。每个算法对应一种特定的识别类型，如设备状态识别、表计读数识别、缺陷检测等。

## 字段说明

### 主键与审计

| 字段名 | 数据类型 | 说明 | 约束 |
|-------|---------|------|------|
| `Id` | `Guid` | 主键 | Primary Key |
| `CreationTime` | `DateTime` | 创建时间 | 继承自CreationAuditedEntity |

### 算法信息

| 字段名 | 数据类型 | 说明 | 备注 |
|-------|---------|------|------|
| `Name` | `string` | 算法名称 | 如："设备状态识别" |
| `Key` | `string` | 算法键值 | 唯一标识，用于代码引用 |
| `Type` | `AlgorithmTypeEnum` | 识别类型 | 见下方枚举 |
| `Unit` | `string?` | 单位 | 算法输出的数据单位 |
| `ValueType` | `DataValueTypeEnum` | 值类型 | 输出数据的类型 |

## 枚举类型

### AlgorithmTypeEnum — 识别算法类型
```csharp
public enum AlgorithmTypeEnum
{
    DeviceStatus = 0,      // 设备状态识别
    MeterReading = 1,      // 表计读数识别
    DefectDetection = 2,    // 缺陷检测
    Temperature = 3,       // 温度识别
    AnomalyDetection = 4   // 异常检测
}
```

### DataValueTypeEnum — 数据值类型
```csharp
public enum DataValueTypeEnum
{
    Int = 0,       // 整数
    Float = 1,      // 浮点数
    String = 2,     // 字符串
    Json = 3,       // JSON对象
    Enum = 4        // 枚举值
}
```

## 业务规则

### 算法管理规则
1. **全局定义**：算法是全局可用的识别能力定义
2. **键值唯一**：Key 字段在代码中用于引用算法，必须唯一
3. **类型分类**：通过 Type 字段对算法进行分类管理
4. **输出规范**：Unit 和 ValueType 定义算法输出数据的规范

### 预置算法建议
| 算法名称 | Key | 类型 | 单位 | 值类型 |
|---------|-----|------|------|--------|
| 设备状态识别 | device-status | DeviceStatus | - | Enum(0:正常,1:异常) |
| 表计读数识别 | meter-reading | MeterReading | - | Float |
| 温度识别 | temperature-recognition | Temperature | ℃ | Float |
| 缺陷检测 | defect-detection | DefectDetection | - | Json |
| 局放检测 | partial-discharge | AnomalyDetection | - | Float |

## 相关服务

- [[AlgorithmService]] — 算法服务
- [[AIRecognitionService]] — AI识别服务
- [[RecognitionApiService]] — 识别API服务

## 数据查询示例

### 查询所有可用的识别算法
```sql
SELECT * FROM ast_algorithm
ORDER BY creation_time DESC;
```

### 查询某类型的所有算法
```sql
SELECT * FROM ast_algorithm
WHERE type = 0  -- DeviceStatus
ORDER BY creation_time DESC;
```

### 查询算法及其使用统计
```sql
SELECT
    a.key,
    a.name,
    a.type,
    a.unit,
    COUNT(DISTINCT ai.id) as usage_count
FROM ast_algorithm a
LEFT JOIN ast_ai_recognition ai ON a.key = ai.algorithm_key
GROUP BY a.key, a.name, a.type, a.unit
ORDER BY usage_count DESC;
```

## 索引建议

```sql
-- 键值索引（用于代码引用）
CREATE INDEX IX_Key ON ast_algorithm(key);

-- 类型索引（用于按类型筛选）
CREATE INDEX IX_Type ON ast_algorithm(type);

-- 创建时间索引（用于按时间排序）
CREATE INDEX IX_CreationTime ON ast_algorithm(creation_time);
```

## 业务价值

**核心作用**：
1. **算法管理**：统一管理系统中可用的识别算法
2. **类型分类**：支持按业务类型对算法分类
3. **输出规范**：定义算法输出数据的格式和单位
4. **扩展性**：便于添加新的识别算法类型

## 使用场景

### 场景1：配置算法识别
```csharp
// 获取温度识别算法
var algorithm = await _algorithmRepository.GetAsync(a => a.Key == "temperature-recognition");

// 应用算法进行识别
var result = await _aiService.RecognizeAsync(image, algorithm);

// 结果包含：
// - Value: 25.5 (float)
// - Unit: "℃" (来自算法定义)
// - ValueType: DataValueTypeEnum.Float
```

### 场景2：算法选择器
```tsx
<AlgorithmSelector>
  {algorithms.map(algo => (
    <AlgorithmOption
      key={algo.key}
      name={algo.name}
      type={algo.type}
      unit={algo.unit}
    />
  ))}
</AlgorithmSelector>
```

### 场景3：动态算法调用
```csharp
// 根据点位配置动态选择算法
var point = await _pointRepository.GetAsync(pointId);
var algorithm = await _algorithmRepository.GetAsync(a => a.Key == point.AlgorithmKey);

var recognitionResult = await _recognitionService.RecognizeAsync(
    imageData,
    algorithm.Key,
    algorithm.ValueType
);
```

## 设计模式

### 策略模式
- `AlgorithmEntity` 定义算法策略
- 运行时根据算法 Key 动态选择识别策略
- 支持算法的动态扩展和替换

### 工厂模式
```csharp
public class AlgorithmFactory
{
    public IAlgorithmAlgorithm Create(string key)
    {
        var algorithm = await _repository.GetAsync(a => a.Key == key);
        
        return key switch
        {
            "temperature-recognition" => new TemperatureAlgorithm(),
            "meter-reading" => new MeterReadingAlgorithm(),
            "defect-detection" => new DefectDetectionAlgorithm(),
            _ => throw new NotSupportedException($"算法 {key} 不支持")
        };
    }
}
```

## 与识别系统的集成

### 识别流程
```
1. 图像采集
   ↓
2. 选择识别算法（根据点位配置）
   ↓
3. 执行识别算法（调用AI服务）
   ↓
4. 格式化输出结果（根据算法的 Unit 和 ValueType）
   ↓
5. 保存识别结果（到 AstPoint 或 AI识别记录）
```

### 结果格式化
```csharp
public object FormatResult(AlgorithmEntity algorithm, dynamic rawResult)
{
    return algorithm.ValueType switch
    {
        DataValueTypeEnum.Int => Convert.ToInt32(rawResult),
        DataValueTypeEnum.Float => Convert.ToDouble(rawResult),
        DataValueTypeEnum.String => rawResult.ToString(),
        DataValueTypeEnum.Json => JsonConvert.SerializeObject(rawResult),
        DataValueTypeEnum.Enum => Convert.ToInt32(rawResult),
        _ => rawResult
    };
}
```

---

> **最后更新**：2026-06-04
> **源码位置**：`module/ast-intellisub/Ast.IntelliSub.Domain/Entities/Monitoring/AlgorithmEntity.cs`
