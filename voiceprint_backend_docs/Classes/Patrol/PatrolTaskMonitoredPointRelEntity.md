# PatrolTaskMonitoredPointRelEntity

**关联表：巡检任务-监测点位多对多关系**

## 概述

`PatrolTaskMonitoredPointRelEntity` 是巡检任务与监测点位之间的多对多关联表，用于定义巡检任务需要检查的监测点位及执行顺序。每个巡检任务可以包含多个监测点位，每个点位也可以被多个巡检任务引用。

## 表信息

| 属性 | 值 |
|------|-----|
| **表名** | `ast_patrol_task_monitored_point_rel` |
| **主键** | `Id` (GUID) |
| **复合主键** | 无（使用单主键 Id） |

## 字段说明

| 字段名 | 类型 | 说明 | 外键 |
|--------|------|------|------|
| `Id` | `Guid` | 主键 ID | - |
| `patrol_task_id` | `Guid` | 巡检任务 ID | → `PatrolTaskAggregateRoot.Id` |
| `monitored_object_item_rel_id` | `Guid` | 监测对象项关系 ID | → `MonitoredObjectItemRelEntity.Id` |
| `monitored_point_id` | `Guid` | 监测点位 ID | → `MonitoredPointEntity.Id` |
| `order_num` | `int` | 排序号（默认值：1） | - |

### 索引

| 索引名 | 字段 | 类型 |
|--------|------|------|
| `IX_PatrolTaskId` | `patrol_task_id` | ASC |
| `IX_MonitoredPointId` | `monitored_point_id` | ASC |
| `IX_OrderNum` | `order_num` | ASC |

## 关联实体 (ER 关系)

```
PatrolTaskAggregateRoot (1) ←→ (*) MonitoredPointEntity
         │                              │
         └── patrol_task_id             monitored_point_id
                      │              │
                      ▼              ▼
          PatrolTaskMonitoredPointRelEntity
                      │
                      │ monitored_object_item_rel_id
                      ▼
          MonitoredObjectItemRelEntity
```

**关联端：**

| 导航属性 | 目标实体 | 关联类型 |
|----------|----------|----------|
| `PatrolTask` | `PatrolTaskAggregateRoot` | Many-to-One |
| `MonitoredPoint` | `MonitoredPointEntity` | Many-to-One |
| `MonitoredObjectItemRel` | `MonitoredObjectItemRelEntity` | Many-to-One |

## 服务使用

### 写入服务

- **`PatrolTaskService`** - 创建/更新/删除巡检任务时管理关联点位
  - `CreateAsync()` - 创建任务时初始化点位关联
  - `UpdateAsync()` - 更新任务时重新构建点位关联
  - `ValidateMonitoredPointBindingsAsync()` - 验证点位绑定有效性

### 读取服务

- **`PatrolTaskService`** - 查询任务的监测点位列表
- **`PatrolExecutionService`** - 执行巡检时获取点位顺序

## 业务规则

1. **级联删除**
   - 删除巡检任务时，自动删除对应的关联记录

2. **顺序控制**
   - 同一巡检任务内，`order_num` 决定点位检查顺序
   - 默认值为 1，创建后可通过服务层调整

3. **数据完整性**
   - `monitored_object_item_rel_id` 必须关联有效的监测对象项
   - `monitored_point_id` 必须已配置数据绑定（通过 `PointBindingRelEntity`）

## 源码位置

```
module/ast-intellisub/
└── Ast.IntelliSub.Domain/
    └── Entities/
        └── Patrol/
            └── PatrolTaskMonitoredPointRelEntity.cs
```

## 相关文档

- [PatrolTaskAggregateRoot](../../Aggregates/Patrol/PatrolTaskAggregateRoot.md) - 巡检任务聚合根
- [MonitoredPointEntity](Monitoring/MonitoredPointEntity.md) - 监测点位实体
- [MonitoredObjectItemRelEntity](Monitoring/MonitoredObjectItemRelEntity.md) - 监测对象项关联表
