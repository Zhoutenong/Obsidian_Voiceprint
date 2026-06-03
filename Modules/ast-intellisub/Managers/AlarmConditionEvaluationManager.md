# AlarmConditionEvaluationManager

**路径**: `module/ast-intellisub/Ast.IntelliSub.Application/Managers/AlarmConditionEvaluationManager.cs`

**依赖**: `ITransientDependency`

## 概述

告警条件表达式求值管理器，负责评估动态告警条件表达式。基于 `DynamicExpresso` 库实现安全的表达式求值，支持告警策略、多条件告警和丢弃保护策略中的条件判断。

## 核心功能

### 1. 条件表达式求值

```csharp
public async Task<bool> EvaluateConditionAsync(
    string condition, 
    Parameter[] parameters, 
    string contextName = "")
```

**职责**：
- 将 `$参数名` 格式转换为 DynamicExpresso 兼容格式（大小写不敏感）
- 使用共享 `Interpreter` 执行表达式求值
- 返回布尔结果

**支持的参数类型**：
- `$value` - 当前点位的数值
- `$firstValue` - 历史首个值（基准对比）
- `$属性名称` - 批量点位中的其他属性值

**表达式示例**：
```csharp
"$value > 60"                    // 当前值大于60
"$value > $firstValue * 3"       // 超过首次值3倍
"$temp_A > 80 && $temp_B > 90"   // 多属性联合判断
```

### 2. 参数名提取

```csharp
public HashSet<string> ExtractParameterNames(string condition)
```

**职责**：
- 使用正则表达式提取 `$` 开头的参数名
- 返回大小写不敏感的参数集合
- 用于识别表达式中依赖的数据源

**正则模式**：`\$[a-zA-Z_][a-zA-Z0-9_]*\b`

### 3. 参数列表构建

```csharp
public async Task<List<Parameter>> BuildParametersAsync(
    ProcessingContext context, 
    HashSet<string> parameterNames)
```

**职责**：
- 根据提取的参数名动态构建参数列表
- 支持从上下文获取 `$value`、`$firstValue`
- 支持从批量点位数据获取自定义属性值
- 异步查询首次值并缓存（提升性能）

**数据源映射**：
| 参数名 | 数据来源 |
|--------|----------|
| `value` | `context.PointVal.Value` |
| `firstValue` | 历史查询或缓存 |
| 其他属性 | `context.BatchPointValList` |

## 使用场景

### 条件告警策略 (ConditionAlarmStrategy)

```csharp
// 单条件告警
var result = await _evaluationManager.EvaluateConditionAsync(
    "$value > 80",
    new[] { new Parameter("value", 85.5) },
    "温度告警"
);
```

### 多条件告警策略 (MultiConditionAlarmStrategy)

```csharp
// 预警条件
bool isWarning = await _evaluationManager.EvaluateConditionAsync(
    "$value > $firstValue * 2",
    parameters,
    "预警判断"
);

// 告警条件
bool isAlarm = await _evaluationManager.EvaluateConditionAsync(
    "$value > $firstValue * 5",
    parameters,
    "告警判断"
);
```

### 丢弃保护策略 (ConditionDiscardProtectionStrategy)

```csharp
// 判断是否应丢弃数据
bool shouldDiscard = await _evaluationManager.EvaluateConditionAsync(
    "$value == 0 || $value > 1000",
    parameters,
    "异常值过滤"
);
```

## 技术特性

### 性能优化

**共享 Interpreter**：
- 使用单例 `Interpreter` 实例而非每次创建
- 减少内存分配和编译开销

**首次值缓存**：
- 通过 `FirstValueCacheRef` 引传缓存
- 避免重复数据库查询

### 安全性

**输入验证**：
- 空表达式检查
- 参数名格式验证
- 异常捕获和日志记录

**错误处理**：
```csharp
catch (Exception ex)
{
    _logger.LogError(ex, "条件表达式评估失败：{Condition}", condition);
    return false;  // 失败时返回安全默认值
}
```

### 大小写不敏感

```csharp
// 使用 RegexOptions.IgnoreCase
var pattern = $@"\${Regex.Escape(param.Name)}\b";
processedCondition = Regex.Replace(processedCondition, pattern, param.Name, 
    RegexOptions.IgnoreCase);
```

## 依赖关系

### 输入依赖

- **ProcessingContext** - 处理上下文（包含点位数据）
- **Parameter[]** - DynamicExpresso 参数数组

### 输出依赖

- **bool** - 条件判断结果
- **HashSet<string>** - 提取的参数名集合
- **List<Parameter>** - 构建的参数列表

### 协作对象

- **AstPointDataService** - 点位数据服务（查询首次值）
- **ILogger** - 日志记录
- **Interpreter (DynamicExpresso)** - 表达式求值引擎

## 配置要求

无特殊配置项，依赖 `DynamicExpresso` NuGet 包。

## 扩展点

### 自定义参数解析器

可通过继承并重写 `BuildParametersAsync` 扩展参数解析逻辑。

### 自定义表达式函数

在 `Interpreter` 初始化时注册自定义函数：

```csharp
_sharedInterpreter = new Interpreter()
    .RegisterFunction("Math.Abs", (double value) => Math.Abs(value));
```

## 相关文档

- [条件告警策略](../Strategies/Alarms/ConditionAlarmStrategy.md)
- [多条件告警策略](../Strategies/Alarms/MultiConditionAlarmStrategy.md)
- [丢弃保护策略](../Strategies/Protections/ConditionDiscardProtectionStrategy.md)
