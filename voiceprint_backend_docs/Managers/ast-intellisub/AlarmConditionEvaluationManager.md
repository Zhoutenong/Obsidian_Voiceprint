# AlarmConditionEvaluationManager — 告警条件评估管理器

## 基本信息

- **Manager名称**：`AlarmConditionEvaluationManager`
- **模块位置**：`module/ast-intellisub/Ast.IntelliSub.Application/Managers/`
- **生命周期**：`ITransientDependency` - 瞬态依赖
- **命名空间**：`Ast.IntelliSub.Application.Managers`

## Manager概述

AlarmConditionEvaluationManager 是告警系统的核心管理器，负责解析、评估和管理告警条件表达式。它提供统一的条件解析、参数构建和表达式评估功能，支持多种告警策略的代码复用。

## 核心职责

1. **条件表达式解析** - 从表达式中提取参数名称
2. **参数构建** - 构建条件评估所需的参数列表
3. **表达式评估** - 动态评估布尔表达式
4. **历史数据查询** - 获取点位的历史首个值（基准值）
5. **批量参数支持** - 支持多点位联合判断

## 依赖注入

| 依赖 | 职责 |
|------|------|
| `ILogger<AlarmConditionEvaluationManager>` | 日志记录器 |
| `AstPointDataService` | 点位数据服务（用于查询历史数据） |

## 支持的参数类型

所有参数名必须以 `$` 开头（大小写不敏感）：

| 参数名 | 说明 | 数据来源 |
|--------|------|----------|
| `$value` | 当前点位的数值 | 方法参数传入 |
| `$firstValue` | 当前点位在历史数据中的第一个值 | AstPointDataService查询 |
| `$属性名称` | BatchPointValList中其他点位的属性值 | ProcessingContext.BatchPointValList |

## 条件表达式示例

```csharp
// 简单阈值判断
"$value > 80"

// 相对变化判断
"$value > $firstValue * 5"

// 多属性联合判断
"$jxa_temperature > 100 && $jxb_temperature > 200"

// 复杂表达式
"($value > 80 && $firstValue < 50) || ($value < 20 && $firstValue > 30)"
```

## 核心方法

### 1. ExtractParameterNames - 提取参数名称

**签名**：
```csharp
public HashSet<string> ExtractParameterNames(string condition)
```

**功能**：
从条件表达式中提取所有参数名称（去除了 `$` 前缀）。

**实现逻辑**：
```csharp
// 匹配以$开头的变量名：$字母开头，可包含字母、数字、下划线
var regex = new Regex(@"\$[a-zA-Z_][a-zA-Z0-9_]*\b", RegexOptions.IgnoreCase);
var matches = regex.Matches(condition);

// 使用忽略大小写的HashSet
var parameters = new HashSet<string>(StringComparer.OrdinalIgnoreCase);
foreach (var match in matches)
{
    // 去掉$前缀，获取实际参数名称
    var paramName = match.Value.Substring(1);
    parameters.Add(paramName);
}

return parameters;
```

### 2. BuildParametersAsync - 构建参数列表

**签名**：
```csharp
public async Task<Parameter[]> BuildParametersAsync(
    ProcessingContext context, 
    double value, 
    string condition,
    FirstValueCacheRef? firstValueCache = null)
```

**功能**：
构建条件表达式评估所需的参数数组。

**处理流程**：

```
1. 添加基础value参数
   │
   ├─ parameters.Add(new Parameter("value", value))
   │
2. 解析条件表达式中的参数名称
   │
   ├─ ExtractParameterNames(condition)
   │
3. 遍历参数名称
   │
   ├─ 如果是 "value" → 已添加，跳过
   │
   ├─ 如果是 "FirstValue"
   │  ├─ 调用 GetFirstValueAsync 查询历史首个值
   │  ├─ 支持缓存机制（firstValueCache）
   │  └─ 添加参数
   │
   └─ 如果是其他属性名
      ├─ 从 BatchPointValList 中查找对应属性
      ├─ 根据 ValueType 进行类型转换
      │  ├─ Int/Enum → Convert.ToInt32
      │  ├─ Float → Convert.ToDouble
      │  └─ String/Json → 跳过（不支持）
      └─ 添加参数
```

### 3. EvaluateConditionAsync - 评估条件表达式

**签名**：
```csharp
public async Task<bool> EvaluateConditionAsync(
    string condition, 
    Parameter[] parameters, 
    string contextName = "")
```

**功能**：
动态评估包含变量的布尔表达式。

**实现逻辑**：
```csharp
// 1. 提前检查无效参数
if (string.IsNullOrWhiteSpace(condition))
    return false;

// 2. 将$参数名替换为不带$的参数名
var processedCondition = condition;
foreach (var param in parameters)
{
    // 使用大小写不敏感的正则表达式替换
    var pattern = $@"\${Regex.Escape(param.Name)}\b";
    processedCondition = Regex.Replace(processedCondition, pattern, param.Name, RegexOptions.IgnoreCase);
}

// 3. 使用共享的 Interpreter 执行计算
object result = _sharedInterpreter.Eval(processedCondition, parameters);
bool boolResult = (bool)result;

return boolResult;
```

**性能优化**：
- 使用静态单例 `Interpreter`，所有实例共享同一个表达式树缓存
- DynamicExpresso.Interpreter 是线程安全的

### 4. GetFirstValueAsync - 获取历史首个值

**签名**：
```csharp
public async Task<double?> GetFirstValueAsync(
    ProcessingContext context, 
    FirstValueCacheRef? firstValueCache = null)
```

**功能**：
获取指定点位的历史第一个值，支持缓存机制以提高性能。

**缓存机制**：
```csharp
// 优先使用缓存值
if (firstValueCache?.Value.HasValue == true)
{
    return firstValueCache.Value.Value;
}

// 查询数据库
var firstValue = await _astPointDataService.GetFirstValueAsync(
    pointGuid, 
    context.PointVal.SensorKey, 
    context.PointVal.DeviceId, 
    context.PointVal.Property);

// 缓存到状态中
if (firstValueCache != null)
{
    firstValueCache.Value = firstValueDouble;
}
```

### 5. ExtractParameterNamesFromMultipleConditions - 批量提取参数

**签名**：
```csharp
public HashSet<string> ExtractParameterNamesFromMultipleConditions(params string[] conditions)
```

**功能**：
从多个条件表达式中提取所有参数名称的集合。

## 相关类

### FirstValueCacheRef - FirstValue缓存引用

```csharp
public class FirstValueCacheRef
{
    /// <summary>
    /// 缓存的FirstValue值
    /// </summary>
    public double? Value { get; set; } = null;
}
```

**用途**：
- 在管理器和策略之间共享FirstValue缓存状态
- 避免重复查询数据库
- 提高告警条件评估性能

## 日志记录

### 调试日志 (LogDebug)

```csharp
// 参数解析
"条件表达式解析到的参数：{Parameters}，点位ID：{PointId}"

// FirstValue获取
"获取到$FirstValue：{FirstValue}，点位ID：{PointId}"
"使用缓存的FirstValue：{FirstValue}，点位ID：{PointId}"

// BatchPoint参数
"从BatchPointValList获取到参数：${ParamName}={Value}，点位ID：{PointId}"

// 条件评估
"{ContextName}条件表达式评估完成：{Condition} -> {ProcessedCondition} = {Result}"
```

### 警告日志 (LogWarning)

```csharp
// FirstValue获取失败
"无法获取$FirstValue，使用当前值作为默认值，点位ID：{PointId}"

// 参数未找到
"在BatchPointValList中未找到参数：${ParamName}"

// 类型不支持
"BatchPointValList中的参数类型不支持：${ParamName}，类型={ValueType}"
```

### 错误日志 (LogError)

```csharp
// 条件评估失败
"{ContextName}条件表达式评估失败：{Condition}，参数：{@Parameters}"

// FirstValue查询失败
"获取点位FirstValue失败，点位ID：{PointId}"
```

## 错误处理

### 异常抛出

```csharp
// 条件表达式评估失败时抛出
throw new ArgumentException(
    $"Failed to evaluate {contextName} condition: \"{condition}\". See inner exception for details.", ex);
```

### 容错处理

- 条件表达式为空时返回 `false`
- FirstValue查询失败时使用当前值作为默认值
- 不支持的参数类型跳过该参数
- 参数未找到时记录警告日志

## 数据类型转换

### 支持的类型

| ValueType | 转换方法 | 示例 |
|-----------|---------|------|
| Int | `Convert.ToInt32(value)` | `42` |
| Enum | `Convert.ToInt32(value)` | `1` |
| Float | `Convert.ToDouble(value)` | `3.14` |
| String | 跳过 | - |
| Json | 跳过 | - |

## 业务规则

### 1. 参数命名规则

- 必须以 `$` 开头
- 字母开头，可包含字母、数字、下划线
- 大小写不敏感

### 2. 特殊参数

- `$value` - 当前点位的数值，必须提供
- `$firstValue` - 历史首个值，可选参数
- `$属性名` - 从BatchPointValList中获取

### 3. 参数查找顺序

1. 首先检查是否为内置参数（value、FirstValue）
2. 然后在BatchPointValList中查找（大小写不敏感）
3. 未找到时记录警告日志

### 4. 缓存策略

- FirstValue查询结果会被缓存
- 缓存通过FirstValueCacheRef在调用间共享
- 避免重复查询数据库，提高性能

## 性能优化

### 1. 静态Interpreter单例

```csharp
// ✅ 使用静态单例 Interpreter，所有实例共享同一个表达式树缓存
private static readonly Interpreter _sharedInterpreter = new Interpreter();
```

**好处**：
- 表达式树缓存可跨实例共享
- 减少重复解析开销
- DynamicExpresso.Interpreter 是线程安全的

### 2. FirstValue缓存

```csharp
// 首次查询会从数据库获取并缓存
var firstValue = await GetFirstValueAsync(context, firstValueCache);

// 后续直接使用缓存值
if (firstValueCache?.Value.HasValue == true)
{
    return firstValueCache.Value.Value;
}
```

**好处**：
- 避免重复查询数据库
- 提高告警评估速度

### 3. 批量参数提取

```csharp
// 一次性提取多个条件的所有参数
var allParameters = ExtractParameterNamesFromMultipleConditions(condition1, condition2, condition3);
```

## 相关服务

- `AstPointDataService` - 点位数据服务（提供历史数据查询）
- `AlarmStrategy` - 告警策略（使用此管理器评估条件）
- `PointValueProcessingService` - 点位值处理服务

## 相关实体

- `ProcessingContext` - 处理上下文
- `BatchPointVal` - 批量点位值
- `PointVal` - 点位值

## 使用场景

### 1. 阈值告警

```csharp
var condition = "$value > 80";
var parameters = await BuildParametersAsync(context, value, condition);
var result = await EvaluateConditionAsync(condition, parameters);
// result = true 表示告警触发
```

### 2. 相对变化告警

```csharp
var condition = "$value > $firstValue * 5";
// 当前值大于首次值的5倍时触发告警
```

### 3. 多点位联合告警

```csharp
var condition = "$jxa_temperature > 100 && $jxb_temperature > 100 && $jxc_temperature > 100";
// 三相进线温度都超过100℃时触发告警
```

### 4. 复杂条件告警

```csharp
var condition = "($value > 80 && $firstValue < 50) || ($value < 20 && $firstValue > 30)";
// 满足以下任一条件时触发告警：
// 1. 当前值大于80且首次值小于50
// 2. 当前值小于20且首次值大于30
```

## 设计特点

1. **参数化表达式** - 支持 `$` 开头的变量参数
2. **大小写不敏感** - 参数名和表达式都忽略大小写
3. **类型安全转换** - 根据ValueType智能转换数据类型
4. **性能优化** - 静态Interpreter、FirstValue缓存
5. **批量支持** - 支持多点位联合判断
6. **容错设计** - 参数缺失时优雅降级
7. **详细日志** - 记录解析、评估的每个步骤

## 注意事项

1. **参数命名** - 必须以 `$` 开头，否则不会被识别
2. **表达式语法** - 遵循C#表达式语法
3. **数据类型** - String和Json类型的BatchPoint不支持数值计算
4. **缓存管理** - FirstValue缓存在同一次告警处理中有效
5. **异常处理** - 条件评估失败会抛出ArgumentException，需要上层捕获

---

> **最后更新**：2026-06-04  
> **源码位置**：`module/ast-intellisub/Ast.IntelliSub.Application/Managers/AlarmConditionEvaluationManager.cs`
