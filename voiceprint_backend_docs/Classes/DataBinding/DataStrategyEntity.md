# DataStrategyEntity

数据策略实体，定义数据处理策略的配置和实现。

## 表名与主键

- **表名**: `ast_data_strategy`
- **主键**: `Id` (Guid)

## 字段列表

| 字段名 | 类型 | 说明 | 外键 |
|--------|------|------|------|
| `Id` | Guid | 主键ID | - |
| `name` | string | 显示名称 | - |
| `strategy_type` | StrategyTypeEnum | 策略类型（枚举） | - |
| `class_name` | string | 策略类名（完整类名） | - |
| `parameters` | text | 策略参数（JSON格式） | - |

## 关联实体（ER关系）

### 关联到当前实体

- **BindingItemStrategyRelEntity** (1:N)
  - 通过 `data_strategy_id` 关联
  - 表示策略与绑定项的关系

### 关联通过中间表

- **DataBindingItemEntity** (M:N)
  - 通过 `BindingItemStrategyRelEntity` 中间表关联
  - 一个策略可以应用于多个绑定项
  - 一个绑定项可以应用多个策略

## 被哪些服务读写

### 写入服务

- **SubstationDataSeed** (数据种子初始化)
  - `InitializeDataStrategies()` - 初始化预设策略配置

- **DataStrategyAppService** (推断)
  - 创建/更新/删除策略配置

### 读取服务

- **DataStrategyAppService** (推断)
  - 获取策略列表
  - 获取策略详情

- **数据处理服务** (推断)
  - 运行时读取策略配置并执行
  - 根据策略类型调用对应的处理逻辑

## 业务规则约束

1. **策略类型枚举**
   - `StrategyTypeEnum.Alarm` - 告警策略
   - `StrategyTypeEnum.Protection` - 数据保护策略
   - `StrategyTypeEnum.Validation` - 数据验证策略
   - `StrategyTypeEnum.Transformation` - 数据转换策略

2. **类名规范**
   - `class_name` 必须是完整的类名（包含命名空间）
   - 类必须实现对应的策略接口
   - 类必须可以被反射实例化

3. **参数格式**
   - `parameters` 必须是有效的 JSON 格式
   - 参数结构由具体策略类定义
   - 参数值在策略执行时被解析和使用

4. **命名唯一性**
   - `name` 字段建议唯一
   - 用于标识不同的策略

## 策略类型详解

### Alarm（告警策略）
用于监测数据是否超过告警阈值：

```json
{
  "name": "温度告警",
  "strategy_type": "Alarm",
  "class_name": "Ast.IntelliSub.Domain.Strategies.AlarmStrategy",
  "parameters": {
    "threshold_high": 80,
    "threshold_low": 10,
    "level": "Warning",
    "message": "温度异常"
  }
}
```

### Protection（数据保护策略）
用于数据保护和容错处理：

```json
{
  "name": "温度数据保护",
  "strategy_type": "Protection",
  "class_name": "Ast.IntelliSub.Domain.Strategies.DataProtectionStrategy",
  "parameters": {
    "max_value": 100,
    "min_value": -50,
    "default_value": 0,
    "enable_filter": true
  }
}
```

### Validation（数据验证策略）
用于数据有效性验证：

```json
{
  "name": "数值范围验证",
  "strategy_type": "Validation",
  "class_name": "Ast.IntelliSub.Domain.Strategies.ValidationStrategy",
  "parameters": {
    "regex": "^-?\\d+(\\.\\d+)?$",
    "allow_null": false
  }
}
```

### Transformation（数据转换策略）
用于数据格式转换：

```json
{
  "name": "单位转换",
  "strategy_type": "Transformation",
  "class_name": "Ast.IntelliSub.Domain.Strategies.UnitTransformationStrategy",
  "parameters": {
    "from_unit": "F",
    "to_unit": "C",
    "formula": "(F-32)*5/9"
  }
}
```

## 数据示例

典型的策略配置：

```csharp
// 温度告警策略
{
  "Id": "guid-1",
  "Name": "温度告警",
  "StrategyType": 0,  // Alarm
  "ClassName": "Ast.IntelliSub.Domain.Strategies.TemperatureAlarmStrategy",
  "Parameters": "{\"threshold_high\":80,\"threshold_low\":10,\"level\":\"Warning\"}"
}

// 湿度告警策略
{
  "Name": "湿度告警",
  "StrategyType": 0,
  "ClassName": "Ast.IntelliSub.Domain.Strategies.HumidityAlarmStrategy",
  "Parameters": "{\"threshold_high\":90,\"threshold_low\":20,\"level\":\"Warning\"}"
}

// 数据保护策略
{
  "Name": "温度数据保护",
  "StrategyType": 1,  // Protection
  "ClassName": "",
  "Parameters": "{\"max_value\":100,\"min_value\":-50,\"default_value\":0}"
}
```

## 策略执行流程

1. **配置阶段**
   - 策略被关联到绑定项（通过 `BindingItemStrategyRelEntity`）
   - 策略配置被存储到数据库

2. **运行阶段**
   - 数据处理服务读取策略配置
   - 通过反射实例化策略类
   - 解析 JSON 参数
   - 执行策略逻辑

3. **结果处理**
   - 告警策略触发告警
   - 验证策略标记数据有效性
   - 转换策略修改数据格式

## 索引建议

- `name` - 用于按名称搜索（建议唯一索引）
- `strategy_type` - 用于按策略类型查询
- `class_name` - 用于按类名查询

## 注意事项

1. **类名验证**
   - 在保存策略时应验证类名的有效性
   - 确保类存在且可实现

2. **参数验证**
   - 参数 JSON 格式必须正确
   - 应根据策略类型验证参数结构

3. **删除保护**
   - 删除策略前需要检查是否有关联的绑定项
   - 建议实现删除前的依赖检查

4. **性能考虑**
   - 反射实例化策略类有一定性能开销
   - 考虑使用策略缓存机制

5. **扩展性**
   - 新增策略类型需要：
     1. 扩展 `StrategyTypeEnum` 枚举
     2. 实现对应的策略类
     3. 在策略工厂中注册
