# PointBindingRelEntity

**关联表：点位-绑定项-传感器点表多对多关系**

## 概述

`PointBindingRelEntity` 是监测点位、数据绑定项和传感器点表之间的关联表，建立了三者之间的绑定关系。每个监测点位可以绑定多个数据绑定项（如温度、湿度等），每个绑定项最终映射到具体的传感器点表地址。这是数据采集的核心配置表，定义了如何从边缘设备获取数据。

## 表信息

| 属性 | 值 |
|------|-----|
| **表名** | `ast_point_binding_rel` |
| **主键** | `Id` (GUID) |
| **唯一性约束** | 已定义（代码中注释掉，通过逻辑控制） |

## 字段说明

| 字段名 | 类型 | 说明 | 外键 |
|--------|------|------|------|
| `Id` | `Guid` | 主键 ID | - |
| `monitored_object_item_rel_id` | `Guid` | 监测对象项关系 ID | → `MonitoredObjectItemRelEntity.Id` |
| `monitored_point_id` | `Guid` | 监测点位 ID | → `MonitoredPointEntity.Id` |
| `data_binding_item_id` | `Guid` | 数据绑定项 ID | → `DataBindingItemEntity.Id` |
| `ast_point_id` | `Guid?` | 传感器点表 ID（可空） | → `AstPointEntity.Id` |
| `status` | `CommonStatusEnum` | 绑定状态枚举 | - |

### 状态枚举 (CommonStatusEnum)

| 值 | 说明 |
|-----|------|
| `Enabled` | 启用 - 数据采集正常 |
| `Disabled` | 禁用 - 暂停数据采集 |
| `Deleted` | 已删除 - 标记删除 |

## 关联实体 (ER 关系)

```
MonitoredObjectItemRelEntity (1)
            │
            └── monitored_object_item_rel_id
                     │
                     ▼
         PointBindingRelEntity
            │           │           │
            │           │           └── data_binding_item_id → DataBindingItemEntity
            │           │
            │           └── monitored_point_id → MonitoredPointEntity
            │
            └── ast_point_id (可空) → AstPointEntity
```

**关联端：**

| 导航属性 | 目标实体 | 关联类型 |
|----------|----------|----------|
| `MonitoredObjectItem` | `MonitoredObjectItemRelEntity` | Many-to-One |
| `MonitoredPoint` | `MonitoredPointEntity` | Many-to-One |
| `BindingItem` | `DataBindingItemEntity` | Many-to-One |
| `AstPoint` | `AstPointEntity` | Many-to-One（可选） |

## 服务使用

### 写入服务

- **`PointBindingRelService`** - 专门服务管理点位绑定关系
  - `CreateAsync()` - 创建绑定关系
  - `UpdateAsync()` - 更新绑定配置
  - `DeleteAsync()` - 删除绑定（软删除）

- **`MonitoredPointService`** - 管理点位时同步维护绑定关系

### 读取服务

- **`MonitoredPointService`** - 获取点位详情时包含绑定信息
- **`DataBindingItemService`** - 查询绑定项应用范围
- **`PointValueProcessingService`** - 数据处理时查询绑定配置

## 业务规则

1. **唯一性约束**
   - 同一 `(monitored_object_item_rel_id, monitored_point_id, data_binding_item_id)` 组合唯一
   - 通过业务逻辑层控制，数据库未启用唯一索引

2. **级联删除**
   - 删除监测点位时，自动删除对应的绑定记录
   - 删除绑定项时，自动删除对应的绑定记录

3. **状态管理**
   - 使用软删除机制（`status = Deleted`）
   - 禁用状态（`Disabled`）保留数据但暂停采集

4. **传感器映射**
   - `ast_point_id` 可为空，表示未配置具体点表
   - 为空时该绑定项不参与实际数据采集

## 数据流

```
边缘设备数据 → AstPoint → PointBindingRel → DataBindingItem → DataStrategy → 告警判断
```

## 源码位置

```
module/ast-intellisub/
└── Ast.IntelliSub.Domain/
    └── Entities/
        └── DataBinding/
            └── PointBindingRelEntity.cs
```

## 相关文档

- [MonitoredPointEntity](Monitoring/MonitoredPointEntity.md) - 监测点位实体
- [DataBindingItemEntity](DataBindingItemEntity.md) - 数据绑定项实体
- [BindingItemStrategyRelEntity](BindingItemStrategyRelEntity.md) - 绑定项-策略关联表
- [AstPointEntity](Gateway/AstPointEntity.md) - 传感器点表实体
