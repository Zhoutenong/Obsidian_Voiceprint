# MonitoredPointAlarmCategoryRelEntity

**关联表：监测点位-告警分类多对多关系**

## 概述

`MonitoredPointAlarmCategoryRelEntity` 是监测点位与告警分类之间的多对多关联表，用于定义每个监测点位支持哪些类型的告警。通过此表，系统可以灵活配置不同点位的告警分类，支持按优先级排序展示。每个点位可关联多个告警分类，便于告警统计和分类管理。

## 表信息

| 属性 | 值 |
|------|-----|
| **表名** | `ast_monitored_point_alarm_category_rel` |
| **主键** | `Id` (GUID) |
| **审计** | 实现 `ICreationAuditedObject` |

## 字段说明

| 字段名 | 类型 | 说明 | 外键 |
|--------|------|------|------|
| `Id` | `Guid` | 主键 ID | - |
| `monitored_point_id` | `Guid` | 监测点位 ID | → `MonitoredPointEntity.Id` |
| `alarm_category_id` | `Guid` | 告警分类 ID | → `AlarmCategoryAggregateRoot.Id` |
| `order_num` | `int` | 排序号 | - |
| `creation_time` | `DateTime` | 创建时间 | - |
| `creator_id` | `Guid?` | 创建者 ID（可空） | - |

## 关联实体 (ER 关系)

```
MonitoredPointEntity (1) ←→ (*) AlarmCategoryAggregateRoot
          │                          │
          └── monitored_point_id     alarm_category_id
                     │          │
                     ▼          ▼
    MonitoredPointAlarmCategoryRelEntity
```

**关联端：**

| 导航属性 | 目标实体 | 关联类型 |
|----------|----------|----------|
| `MonitoredPoint` | `MonitoredPointEntity` | Many-to-One |
| `AlarmCategory` | `AlarmCategoryAggregateRoot` | Many-to-One |

## 服务使用

### 写入服务

- **`MonitoredPointAlarmCategoryRelService`** - 专门服务管理点位-告警分类关联
  - 创建点位时分配默认告警分类
  - 更新告警分类优先级（`order_num`）
  - 删除不需要的告警分类

### 读取服务

- **`MonitoredPointService`** - 获取点位详情时包含关联的告警分类
- **`AlarmCategoryService`** - 查询告警分类应用范围
- **`AlarmProcessingService`** - 告警处理时验证点位支持的分类

## 业务规则

1. **级联删除**
   - 删除监测点位时，自动删除对应的告警分类关联
   - 删除告警分类时，自动删除对应的关联记录

2. **排序规则**
   - `order_num` 决定告警分类的展示优先级
   - 数值越小优先级越高

3. **审计要求**
   - 记录创建者和创建时间
   - 不追踪修改历史

4. **告警分类用途**
   - 用于告警统计和分类管理
   - 支持按告警级别、类型进行筛选
   - 可在告警中心按分类查看告警记录

## 源码位置

```
module/ast-intellisub/
└── Ast.IntelliSub.Domain/
    └── Entities/
        └── Monitoring/
            └── MonitoredPointAlarmCategoryRelEntity.cs
```

## 相关文档

- [MonitoredPointEntity](Monitoring/MonitoredPointEntity.md) - 监测点位实体
- [AlarmCategoryAggregateRoot](Alarm/AlarmCategoryAggregateRoot.md) - 告警分类聚合根
- [AlarmRecordAggregateRoot](Alarm/AlarmRecordAggregateRoot.md) - 告警记录聚合根
