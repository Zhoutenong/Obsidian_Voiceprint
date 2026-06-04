# MonitoredObjectAggregateRoot

被监测对象聚合根，表示变电站中的可监测实体（如设备、设备组、子设备）。

## 表名与主键

- **表名**: `ast_monitored_object`
- **主键**: `Id` (Guid)

## 字段列表

| 字段名 | 类型 | 说明 | 外键 |
|--------|------|------|------|
| `Id` | Guid | 主键ID | - |
| `name` | string | 名称 | - |
| `substation_id` | Guid | 所属站点ID | SubstationAggregateRoot |
| `is_enabled` | bool | 是否启用 | - |
| `parent_id` | Guid? | 父级对象ID | MonitoredObjectAggregateRoot |
| `remark` | text | 备注 | - |
| `monitored_object_type_id` | Guid | 对象类型ID | MonitoredObjectTypeEntity |
| `status` | CommonStatusEnum | 状态（枚举） | - |
| `order_num` | int? | 排序号 | - |
| `CreationTime` | DateTime | 创建时间 | - |
| `CreatorId` | Guid? | 创建者ID | - |
| `LastModificationTime` | DateTime? | 最后修改时间 | - |
| `LastModifierId` | Guid? | 最后修改者ID | - |

## 关联实体（ER关系）

### 关联从当前实体

- **SubstationAggregateRoot** (N:1)
  - 通过 `substation_id` 关联
  - 导航属性：`Substation`
  - 表示对象所属的变电站

- **MonitoredObjectTypeEntity** (N:1)
  - 通过 `monitored_object_type_id` 关联
  - 导航属性：`ObjectType`
  - 定义对象的类型和分类

- **MonitoredObjectAggregateRoot** (N:1, 自关联)
  - 通过 `parent_id` 关联
  - 导航属性：`Parent`
  - 构建对象的层级结构

### 关联到当前实体

- **MonitoredObjectAggregateRoot** (1:N, 自关联)
  - 通过 `parent_id` 关联
  - 表示对象的子级对象

- **MonitoredObjectItemRelEntity** (1:N)
  - 通过 `monitored_object_id` 关联
  - 表示对象与监测项的关系

- **MonitoredObjectPresetRelEntity** (1:N)
  - 通过 `monitored_object_id` 关联
  - 表示对象与预置位的关系

## 被哪些服务读写

### 写入服务

- **MonitoredObjectAppService** (推断)
  - 创建/更新/删除监测对象
  - 启用/禁用对象

- **SubstationDataSeed** (数据种子初始化)
  - `InitializeMonitoredObjects()` - 初始化预设监测对象

- **VoiceprintPortalAppService**
  - `TryRestoreMonitoredObjectStatusAsync()` - 尝试恢复监测对象状态

### 读取服务

- **MonitoredObjectAppService** (推断)
  - 获取监测对象列表
  - 获取监测对象详情
  - 按站点查询对象
  - 按类型查询对象

- **巡检任务服务** (推断)
  - 获取巡检路线中的监测对象
  - 生成巡检任务

- **告警服务** (推断)
  - 查询告警关联的监测对象

## 业务规则约束

1. **层级结构约束**
   - `parent_id` 可以为 null（顶级对象）
   - 不能设置自己为父级对象
   - 层级深度建议不超过 3 层

2. **站点约束**
   - `substation_id` 必须有效
   - 父子对象必须属于同一站点

3. **类型约束**
   - `monitored_object_type_id` 必须有效
   - 对象类型应与对象的实际用途匹配

4. **状态管理**
   - `status` 枚举值：Normal（正常）、Disabled（禁用）、Deleted（已删除）
   - `is_enabled = false` 时对象不应被监测

5. **命名约束**
   - 同一站点下同一层级的对象名称应该唯一
   - 建议使用描述性的命名规范

6. **审计字段**
   - 继承自 `IAuditedObject`
   - 自动记录创建和修改信息

## 对象层级结构

典型的监测对象层级：

```
变电站
├── 开关柜室 (Group, parent_id = null)
│   ├── #1 开关柜 (Device, parent_id = 开关柜室)
│   │   ├── A相断路器 (SubDevice, parent_id = #1 开关柜)
│   │   ├── B相断路器 (SubDevice, parent_id = #1 开关柜)
│   │   └── C相断路器 (SubDevice, parent_id = #1 开关柜)
│   └── #2 开关柜 (Device, parent_id = 开关柜室)
├── 变压器室 (Group, parent_id = null)
│   └── #1 接地变 (Device, parent_id = 变压器室)
└── 控制室 (Group, parent_id = null)
    └── 控制屏 (Device, parent_id = 控制室)
```

## 数据示例

典型的监测对象配置：

```csharp
// 设备组
{
  "Id": "guid-1",
  "Name": "35kV 开关室",
  "SubstationId": "substation-guid",
  "IsEnabled": true,
  "ParentId": null,
  "Remark": "35kV开关设备室",
  "MonitoredObjectTypeId": "room-type-guid",
  "Status": 0,  // Normal
  "OrderNum": 1,
  "CreationTime": "2024-01-01T00:00:00"
}

// 设备
{
  "Id": "guid-2",
  "Name": "#1 开关柜",
  "SubstationId": "substation-guid",
  "IsEnabled": true,
  "ParentId": "guid-1",  // 属于开关柜室
  "Remark": "",
  "MonitoredObjectTypeId": "switchgear-type-guid",
  "Status": 0,
  "OrderNum": 1
}

// 子设备
{
  "Id": "guid-3",
  "Name": "A相断路器",
  "SubstationId": "substation-guid",
  "IsEnabled": true,
  "ParentId": "guid-2",  // 属于#1开关柜
  "Remark": "A相断路器监测",
  "MonitoredObjectTypeId": "breaker-type-guid",
  "Status": 0,
  "OrderNum": 1
}
```

## 对象状态管理

### 正常状态 (Normal = 0)
- 对象正常启用
- 参与监测和巡检
- 显示在监控界面

### 禁用状态 (Disabled = 1)
- 对象暂时禁用
- 不参与监测和巡检
- 保留数据和历史记录

### 已删除状态 (Deleted = 2)
- 对象已标记删除
- 不显示在正常列表
- 可通过软删除恢复

## 与其他实体的关系

### 与监测项的关系
```
MonitoredObjectAggregateRoot (监测对象)
    ↓
MonitoredObjectItemRelEntity (对象监测项关系)
    ↓
MonitoredItemEntity (监测项)
```

### 与预置位的关系
```
MonitoredObjectAggregateRoot (监测对象)
    ↓
MonitoredObjectPresetRelEntity (对象预置位关系)
    ↓
PresetEntity (预置位)
```

### 与巡检任务的关系
```
PatrolTaskAggregateRoot (巡检任务)
    ↓
PatrolTaskMonitoredPointRelEntity (任务监测点位关系)
    ↓
MonitoredPointEntity (监测点位)
    ↓
MonitoredObjectItemRelEntity (对象监测项关系)
    ↓
MonitoredObjectAggregateRoot (监测对象)
```

## 索引建议

- `substation_id` - 用于按站点查询
- `monitored_object_type_id` - 用于按类型查询
- `parent_id` - 用于查询子对象
- `status` - 用于按状态筛选
- `(substation_id, parent_id, name)` - 联合索引用于层级查询

## 注意事项

1. **层级结构管理**
   - 避免创建过深的层级结构
   - 删除父对象前需要处理子对象

2. **状态同步**
   - 对象状态应与实际设备状态同步
   - 禁用对象应停止相关监测任务

3. **删除保护**
   - 删除对象前检查是否有关联的监测项
   - 删除对象前检查是否在巡检任务中
   - 建议使用软删除

4. **权限控制**
   - 不同用户可能对不同对象的操作权限不同
   - 需要实现站点级别的权限隔离

5. **性能优化**
   - 大型站点的对象数量可能很多
   - 考虑使用分页查询
   - 考虑使用缓存减少数据库访问

6. **数据迁移**
   - 对象层级结构可能需要迁移
   - 需要维护父子关系的完整性
