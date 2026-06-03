# 点位值处理服务 (PointValueProcessingService)

## 概述
点位值处理服务是数据绑定流程的核心服务，负责将传感器点位数据转换为可用的业务数据，执行数据解析、保护和告警策略。

## 职责
- 创建数据处理上下文（ProcessingContext）
- 执行数据解析策略（Parse）
- 执行数据保护策略（Protection）
- 执行告警判断策略（Alarm）
- 自动更新策略执行状态
- 管理策略链执行流程
- 处理批次数据关联分析

## 主要接口

### 创建处理上下文
```csharp
Task<List<ProcessingContext>> CreateProcessingContextsAsync(MqPointValueItemDto pointVal, List<MqPointValueItemDto> batchPointValList)
```

**功能说明**：为单个点位值创建完整的处理上下文，包含绑定关系、策略配置等所有必要信息

**参数说明**：
- `pointVal`: 当前处理的点位值数据
- `batchPointValList`: 同一批次上传的所有点位数据（用于多点位联合分析）

**返回值**：处理上下文列表（一个点位可能对应多个绑定项）

**调用链**：
```
MQTT消息接收 → CreateProcessingContextsAsync → 查询绑定关系 → 查询策略配置 → 返回上下文列表
```

**处理流程**：
1. 解析点位ID，查询点位绑定关系
2. 遍历每个绑定关系，获取数据绑定项信息
3. 获取监测对象与监测项关系
4. 查询绑定项关联的策略列表（Parse/Protection/Alarm）
5. 构建 ProcessingContext 对象
6. 过滤同批次的点位数据用于策略间关联

### 执行策略
```csharp
Task ProcessStrategiesAsync(ProcessingContext context, StrategyTypeEnum strategyType)
```

**功能说明**：执行指定类型的所有策略，按顺序依次执行，支持策略链中断机制

**参数说明**：
- `context`: 处理上下文，包含点位数据、策略配置等信息
- `strategyType`: 策略类型（Parse/Protection/Alarm）

**策略类型**：
- `Parse`: 数据解析策略，负责原始数据格式转换
- `Protection`: 数据保护策略，负责数据有效性验证
- `Alarm`: 告警判断策略，负责异常检测和告警生成

**执行流程**：
1. 检查上下文是否被中止或丢弃
2. 筛选指定类型的策略并按排序字段排序
3. 依次执行每个策略：
   - 设置当前策略上下文
   - 通过工厂创建策略实例
   - 执行策略逻辑
   - 自动更新策略状态
   - 检查是否需要中止后续处理
4. 清理当前策略上下文

**调用链**：
```
ProcessStrategiesAsync → StrategyFactory.CreateStrategy → IStrategyMarker.ExecuteAsync → UpdateStrategyStateAsync
```

### 批量处理点位值
```csharp
Task<List<ProcessingContext>> ProcessPointValuesAsync(List<MqPointValueItemDto> pointVals)
```

**功能说明**：批量处理多个点位值，完整执行所有阶段的策略处理

**处理阶段**：
1. **上下文创建阶段**：为所有点位创建处理上下文
2. **Parse阶段**：执行所有上下文的解析策略
3. **Protection阶段**：执行所有上下文的保护策略
4. **Alarm阶段**：执行所有上下文的告警策略

**执行顺序**：
```
输入点位列表 → 创建所有上下文 → Parse策略 → Protection策略 → Alarm策略 → 返回结果
```

**返回统计**：
- 输入点位数量
- 创建的处理上下文数量
- 有策略配置的上下文数量
- 产生告警的上下文数量
- 被丢弃的上下文数量

### 获取绑定策略列表
```csharp
private Task<List<BIStrategyRelDto>> GetBindingStrategiesAsync(Guid dataBindingItemId, Guid pointBindingRelId)
```

**功能说明**：获取数据绑定项关联的所有策略配置及状态

**查询流程**：
1. 查询策略基本信息（类名、参数、类型等）
2. 批量查询策略执行状态
3. 合并策略配置和状态信息
4. 验证策略配置完整性

**状态缓存**：策略状态通过 IStrategyStateService 获取，支持策略执行历史追踪

### 更新策略状态
```csharp
private Task UpdateStrategyStateAsync(IStrategyMarker strategy, ProcessingContext context)
```

**功能说明**：自动保存策略执行后的状态信息

**更新流程**：
1. 通过反射获取策略的 State 属性
2. 序列化状态为 JSON 字符串
3. 调用 IStrategyStateService 保存或更新状态
4. 记录更新日志

**状态存储格式**：JSON 字符串，支持复杂对象状态

## 业务逻辑

### 完整数据处理流程

```
MQTT接收点位数据
    ↓
创建处理上下文（查询绑定关系和策略）
    ↓
执行Parse策略（数据格式转换）
    ↓
执行Protection策略（数据有效性验证）
    ↓
执行Alarm策略（异常检测和告警）
    ↓
更新策略状态（自动持久化）
    ↓
返回处理结果（包含告警信息）
```

### 策略执行顺序

在同一上下文中，策略按以下顺序执行：

1. **Parse策略**：数据解析和转换
   - 原始字符串 → 数值类型
   - 单位转换
   - 精度处理
   - 数据标准化

2. **Protection策略**：数据保护
   - 范围验证
   - 异常值检测
   - 离线值识别（如 -100）
   - 数据丢弃标记

3. **Alarm策略**：告警判断
   - 阈值检测
   - 趋势分析
   - 多点位联合判断
   - 告警信息生成

### 错误处理机制

**上下文创建失败**：
- 记录错误日志，返回空列表
- 不影响其他点位的处理
- 使用 DEBUG 级别记录无绑定关系的情况

**策略执行失败**：
- 单个策略失败不影响其他策略
- 记录错误日志，继续执行下一个策略
- 标记策略类名和ID便于排查

**数据异常处理**：
- 支持数据丢弃机制（MarkAsDiscarded）
- 支持处理中止机制（AbortProcessing）
- 保留原始值（RawValue）用于追溯

## 依赖服务

### 核心依赖
- [[DataBindingItemService]] - 数据绑定项服务
- [[PointBindingRelService]] - 点位绑定关系服务
- [[BindingItemStrategyRelService]] - 绑定项策略关联服务
- [[DataStrategyService]] - 数据策略服务
- [[StrategyStateService]] - 策略状态服务

### 工厂和服务
- **StrategyFactory** - 策略工厂，负责创建策略实例
- **IStrategyStateService** - 策略状态管理服务

### 数据仓储
- `ISqlSugarRepository<DataBindingItemEntity>` - 数据绑定项仓储
- `ISqlSugarRepository<MonitoredObjectItemRelEntity>` - 监测项关系仓储
- `ISqlSugarRepository<PointBindingRelEntity>` - 点位绑定关系仓储
- `ISqlSugarRepository<BindingItemStrategyRelEntity>` - 策略关联仓储
- `ISqlSugarRepository<DataStrategyEntity>` - 数据策略仓储

## 核心对象

### ProcessingContext（处理上下文）

处理上下文包含以下核心信息：

**输入数据**：
- `PointVal`: 当前处理的点位值
- `BatchPointValList`: 同批次其他点位数据

**绑定关系**：
- `PointBindingRel`: 点位与绑定项的关系
- `DataBindingItem`: 数据绑定项配置
- `ItemRel`: 监测对象与监测项关系

**策略配置**：
- `Strategies`: 策略列表（按类型分组）
- `CurrentStrategy`: 当前执行的策略

**处理状态**：
- `IsAlarm`: 是否产生告警
- `AlarmInfo`: 告警信息集合
- `IsDiscarded`: 数据是否被丢弃
- `ShouldAbortProcessing`: 是否需要中止处理

**上下文方法**：
- `AddAlarm()`: 添加告警信息
- `ReplaceValue()`: 替换点位值
- `MarkAsDiscarded()`: 标记数据为丢弃
- `AbortProcessing()`: 请求中止处理

## 配置项

**无特定配置项**，策略配置通过数据库管理：
- 策略类名存储在 `ast_data_strategy.class_name`
- 策略参数存储在 `ast_data_strategy.parameters`
- 执行顺序存储在 `ast_binding_item_strategy_rel.order_num`

## 注意事项

### 性能优化
- 批量处理时先创建所有上下文，再分阶段执行策略
- 使用批量查询减少数据库访问次数
- 策略状态批量获取和更新

### 数据一致性
- 单个点位处理失败不影响其他点位
- 单个策略执行失败不影响其他策略
- 所有异常都有详细日志记录

### 日志级别
- `DEBUG`: 正常流程记录、无绑定关系
- `INFO`: 策略中止、数据丢弃
- `WARNING`: 策略配置无效、策略创建失败
- `ERROR`: 数据查询失败、策略执行失败

### 策略开发规范
- 策略必须实现 `IStrategyMarker` 接口
- 策略应包含可序列化的 `State` 属性
- 策略通过 `ProcessingContext` 操作数据
- 策略可以调用 `AbortProcessing()` 中止后续处理

## 相关文档

- [[DataBindingItemService]] - 数据绑定项服务
- [[PointBindingRelService]] - 点位绑定关系服务
- [[DataStrategyService]] - 数据策略服务
- [[StrategyStateService]] - 策略状态服务
- [[ProcessingContext]] - 处理上下文说明
- [[数据绑定架构]] - 数据绑定整体架构
