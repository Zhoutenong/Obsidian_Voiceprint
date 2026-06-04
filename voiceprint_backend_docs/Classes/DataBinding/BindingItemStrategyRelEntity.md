# BindingItemStrategyRelEntity

**关联表：绑定项-数据策略多对多关系**

## 概述

`BindingItemStrategyRelEntity` 是数据绑定项与数据处理策略之间的多对多关联表，用于定义每个绑定项应用哪些数据处理策略（如解析策略、告警策略、保护策略等）。一个绑定项可以应用多个策略，策略执行的顺序通过 `order_num` 控制。

## 表信息

| 属性 | 值 |
|------|-----|
| **表名** | `ast_binding_item_strategy_rel` |
| **主键** | `Id` (GUID) |
| **审计** | 实现 `IAuditedObject` |

## 字段说明

| 字段名 | 类型 | 说明 | 外键 |
|--------|------|------|------|
| `Id` | `Guid` | 主键 ID | - |
| `data_binding_item_id` | `Guid` | 数据绑定项 ID | → `DataBindingItemEntity.Id` |
| `data_strategy_id` | `Guid` | 数据策略 ID | → `DataStrategyEntity.Id` |
| `order_num` | `int?` | 排序号（可空） | - |
| `creation_time` | `DateTime` | 创建时间（默认：当前时间） | - |
| `creator_id` | `Guid?` | 创建者 ID（可空） | - |
| `last_modification_time` | `DateTime?` | 最后修改时间（可空） | - |
| `last_modifier_id` | `Guid?` | 最后修改者 ID（可空） | - |

## 关联实体 (ER 关系)

```
DataBindingItemEntity (1) ←→ (*) DataStrategyEntity
           │                          │
           └── data_binding_item_id   data_strategy_id
                     │           │
                     ▼           ▼
          BindingItemStrategyRelEntity
```

**关联端：**

| 导航属性 | 目标实体 | 关联类型 |
|----------|----------|----------|
| `BindingItem` | `DataBindingItemEntity` | Many-to-One |
| `Strategy` | `DataStrategyEntity` | Many-to-One |

## 服务使用

### 写入服务

- **`BindingItemStrategyRelService`** - 专门服务管理绑定项-策略关联
  - 创建绑定项时分配默认策略
  - 更新策略顺序
  - 删除不需要的策略

### 读取服务

- **`DataBindingItemService`** - 获取绑定项时包含关联策略
- **`DataStrategyService`** - 查询策略应用范围

## 业务规则

1. **策略执行顺序**
   - `order_num` 决定策略执行顺序
   - 数值越小优先级越高
   - 为空时默认按创建顺序

2. **级联删除**
   - 删除绑定项时，自动删除对应的策略关联
   - 删除策略时，自动删除对应的关联记录

3. **审计要求**
   - 记录创建者和创建时间
   - 记录最后修改者和修改时间

4. **策略类型**
   - 一个绑定项可关联多种策略：
     - 解析策略（`PrecisionParseStrategy`）
     - 告警策略（`ConditionAlarmStrategy`、`VoiceprintAlarmStrategy` 等）
     - 保护策略（`TemperatureJumpProtectionStrategy` 等）

## 源码位置

```
module/ast-intellisub/
└── Ast.IntelliSub.Domain/
    └── Entities/
        └── DataBinding/
            └── BindingItemStrategyRelEntity.cs
```

## 相关文档

- [DataBindingItemEntity](DataBindingItemEntity.md) - 数据绑定项实体
- [DataStrategyEntity](DataStrategyEntity.md) - 数据策略实体
- [PointBindingRelEntity](PointBindingRelEntity.md) - 点位绑定关联表
