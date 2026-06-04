# StrategyStateEntity

策略状态实体，存储数据处理策略的运行时状态信息。

## 表名与主键

- **表名**: `ast_strategy_state`
- **主键**: `Id` (Guid)

## 字段列表

| 字段名 | 类型 | 说明 | 外键 |
|--------|------|------|------|
| `Id` | Guid | 主键ID | - |
| `binding_item_strategy_rel_id` | Guid | 数据绑定项策略关系ID | BindingItemStrategyRelEntity |
| `point_binding_rel_id` | Guid | 点位绑定关系ID | PointBindingRelEntity |
| `strategy_state` | string(4000) | 策略状态数据（JSON格式） | - |
| `creation_time` | DateTime | 创建时间 | - |
| `last_modification_time` | DateTime? | 最后修改时间 | - |

## 关联实体（ER关系）

### 关联从当前实体

- **BindingItemStrategyRelEntity** (N:1)
  - 通过 `binding_item_strategy_rel_id` 关联
  - 表示该状态对应的绑定项策略关系

- **PointBindingRelEntity** (N:1)
  - 通过 `point_binding_rel_id` 关联
  - 表示该状态对应的点位绑定关系

### 关联通过中间表

- **DataStrategyEntity** (N:1)
  - 通过 `BindingItemStrategyRelEntity` 关联
  - 表示该状态对应的策略配置

- **DataBindingItemEntity** (N:1)
  - 通过 `BindingItemStrategyRelEntity` 关联
  - 表示该状态对应的绑定项

## 被哪些服务读写

### 写入服务

- **数据处理策略服务** (推断)
  - 创建策略状态记录
  - 更新策略运行时状态
  - 删除过期的策略状态

### 读取服务

- **数据处理策略服务** (推断)
  - 读取策略历史状态
  - 分析策略执行效果
  - 故障排查和调试

## 业务规则约束

1. **状态唯一性**
   - `(binding_item_strategy_rel_id, point_binding_rel_id)` 组合应该唯一
   - 每个点位绑定关系的每个策略只有一个状态记录

2. **状态数据格式**
   - `strategy_state` 必须是有效的 JSON 格式
   - 具体结构由策略类型决定

3. **时间戳**
   - `creation_time` 在创建时自动设置
   - `last_modification_time` 在状态更新时自动更新

4. **级联关系**
   - 删除 `BindingItemStrategyRelEntity` 时应删除对应的状态记录
   - 删除 `PointBindingRelEntity` 时应删除对应的状态记录

## 策略状态数据结构

### 告警策略状态

```json
{
  "last_alarm_time": "2024-01-01T10:30:00",
  "alarm_count": 5,
  "current_level": "Warning",
  "last_value": 85.5,
  "threshold_high": 80,
  "threshold_low": 10,
  "is_alarm_active": true,
  "alarm_duration": 300
}
```

### 数据保护策略状态

```json
{
  "protected_count": 120,
  "invalid_count": 5,
  "last_protected_time": "2024-01-01T10:30:00",
  "last_invalid_value": 150.5,
  "default_value_used": 3
}
```

### 数据验证策略状态

```json
{
  "validation_count": 500,
  "failed_count": 2,
  "success_rate": 0.996,
  "last_failed_time": "2024-01-01T09:15:00",
  "last_error_message": "Value out of range"
}
```

### 数据转换策略状态

```json
{
  "transformation_count": 1000,
  "last_transformation_time": "2024-01-01T10:30:00",
  "input_range": {"min": 32, "max": 212},
  "output_range": {"min": 0, "max": 100},
  "conversion_errors": 0
}
```

## 数据示例

典型的策略状态记录：

```csharp
// 温度告警策略状态
{
  "Id": "guid-1",
  "BindingItemStrategyRelId": "rel-guid-1",
  "PointBindingRelId": "point-rel-guid-1",
  "StrategyState": "{\"last_alarm_time\":\"2024-01-01T10:30:00\",\"alarm_count\":5,\"current_level\":\"Warning\"}",
  "CreationTime": "2024-01-01T00:00:00",
  "LastModificationTime": "2024-01-01T10:30:00"
}

// 数据保护策略状态
{
  "BindingItemStrategyRelId": "rel-guid-2",
  "PointBindingRelId": "point-rel-guid-2",
  "StrategyState": "{\"protected_count\":120,\"invalid_count\":5,\"default_value_used\":3}",
  "CreationTime": "2024-01-01T00:00:00",
  "LastModificationTime": "2024-01-01T10:30:00"
}
```

## 策略状态生命周期

### 1. 创建阶段
```
点位绑定关系创建
    ↓
绑定项策略关系创建
    ↓
策略状态记录初始化
    ↓
strategy_state = "{}"
```

### 2. 运行阶段
```
数据到达
    ↓
策略执行
    ↓
状态更新
    ↓
strategy_state 更新
last_modification_time 更新
```

### 3. 清理阶段
```
点位绑定关系删除
    ↓
关联策略状态删除
    ↓
或：定期清理过期状态
```

## 索引建议

- `(binding_item_strategy_rel_id, point_binding_rel_id)` - 联合唯一索引
- `point_binding_rel_id` - 用于按点位查询状态
- `binding_item_strategy_rel_id` - 用于按策略关系查询状态
- `creation_time` - 用于时间范围查询
- `last_modification_time` - 用于查找最近更新的状态

## 注意事项

1. **状态大小限制**
   - `strategy_state` 最大长度为 4000 字符
   - 大型状态数据应考虑使用独立表或压缩

2. **性能优化**
   - 策略状态更新频繁，注意数据库性能
   - 考虑使用缓存减少数据库访问

3. **数据清理**
   - 建议实现定期清理机制
   - 删除长期未使用的状态记录
   - 删除已删除绑定的状态记录

4. **并发控制**
   - 同一状态可能被多个线程更新
   - 考虑使用乐观锁或悲观锁

5. **故障恢复**
   - 状态数据可用于故障恢复
   - 系统重启后可恢复策略执行状态

6. **监控指标**
   - 策略状态可用于性能监控
   - 统计策略执行次数、失败率等指标
