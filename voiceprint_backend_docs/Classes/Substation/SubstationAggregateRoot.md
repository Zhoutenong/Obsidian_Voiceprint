# SubstationAggregateRoot

## 概述

`SubstationAggregateRoot` 是智能变电站监控系统的核心聚合根，代表一个变电站（站点）。支持站点层级结构（父子站点）和多租户场景。

## 表信息

- **表名**: `ast_substation`
- **主键**: `Id` (Guid)
- **继承**: `AggregateRoot<Guid>` + `IAuditedObject`
- **命名空间**: `Ast.IntelliSub.Domain.Entities`

## 字段列表

| 字段名 | 类型 | 数据库列名 | 说明 | 约束 |
|--------|------|-----------|------|------|
| `Id` | `Guid` | `id` | 主键 | PK |
| `Name` | `string` | `name` | 站点名称 | NOT NULL |
| `IsEnabled` | `bool` | `is_enabled` | 是否启用 | Default: true |
| `MapInfo` | `string?` | `map_info` | 站点地图信息 | TEXT (JSON) |
| `SubstationTypeId` | `Guid` | `substation_type_id` | 站点类型ID | FK |
| `ParentId` | `Guid?` | `parent_id` | 父级站点ID | FK (Self) |
| `CreationTime` | `DateTime` | `creation_time` | 创建时间 | |
| `CreatorId` | `Guid?` | `creator_id` | 创建者ID | FK (User) |
| `LastModificationTime` | `DateTime?` | `last_modification_time` | 最后修改时间 | |
| `LastModifierId` | `Guid?` | `last_modifier_id` | 最后修改者ID | FK (User) |

## 关联实体 (ER 关系)

### 导航属性

```csharp
// 站点类型
[Navigate(NavigateType.OneToOne, nameof(SubstationTypeId))]
public SubstationTypeEntity SubstationType { get; set; }

// 父级站点
[Navigate(NavigateType.OneToOne, nameof(ParentId))]
public SubstationAggregateRoot Parent { get; set; }
```

### 关系图

```
SubstationAggregateRoot (1) ←──→ (N) SubstationAggregateRoot (自关联)
         │
         ├── (N) GatewayAggregateRoot
         ├── (1) SubstationTypeEntity
         └── (N) SubstationUserEntity
```

### 级联关系

- **Gateway**: 一个站点可包含多个网关
- **SubstationUser**: 用户可访问多个站点（多对多关系）
- **PatrolTask**: 巡检任务归属于特定站点

## 被哪些服务读写

### 写入服务

- `SubstationService` — 站点 CRUD、层级管理
- `SubstationTypeService` — 站点类型管理

### 读取服务

- `GatewayService` — 查询站点下的网关列表
- `PatrolTaskService` — 获取站点巡检任务
- `RealtimeMonitoringPointService` — 站点实时监控数据
- `ReportService` — 站点报告生成

## 业务规则约束

### 1. 层级约束

- `ParentId` 支持无限层级嵌套
- 禁止循环引用（A → B → A）
- 根站点 `ParentId = null`
- 建议最大层级深度：3 层（地区 → 变电站 → 设备区）

### 2. 启用状态

- `IsEnabled = false` 的站点：
  - 不在监控界面显示
  - 不执行巡检任务
  - 停止数据采集

### 3. 地图信息 (MapInfo)

`MapInfo` 存储站点地图的 JSON 配置：

```json
{
  "center": {
    "lng": 116.4074,
    "lat": 39.9042
  },
  "zoom": 15,
  "markers": [
    {
      "id": "gateway_001",
      "position": { "lng": 116.4075, "lat": 39.9043 },
      "title": "1号网关"
    }
  ],
  "polygons": [
    {
      "points": [
        { "lng": 116.4074, "lat": 39.9042 },
        { "lng": 116.4075, "lat": 39.9043 }
      ],
      "color": "#FF0000"
    }
  ]
}
```

### 4. 删除约束

- 删除站点前需检查：
  - 是否有子站点（需先删除或移动子站点）
  - 是否有关联网关
  - 是否有关联用户

### 5. 审计字段

所有修改都记录审计信息：
- `CreationTime` + `CreatorId` — 创建记录
- `LastModificationTime` + `LastModifierId` — 最后修改

## 使用场景

### 1. 创建根站点（地区）

```csharp
var region = new SubstationAggregateRoot
{
    Id = Guid.NewGuid(),
    Name = "北京市朝阳区",
    IsEnabled = true,
    SubstationTypeId = regionTypeId,
    ParentId = null,
    MapInfo = "{\"center\":{\"lng\":116.4074,\"lat\":39.9042},\"zoom\":12}"
};
```

### 2. 创建子站点（变电站）

```csharp
var substation = new SubstationAggregateRoot
{
    Id = Guid.NewGuid(),
    Name = "朝阳变电站",
    IsEnabled = true,
    SubstationTypeId = substationTypeId,
    ParentId = region.Id,
    MapInfo = "{\"center\":{\"lng\":116.4075,\"lat\":39.9043},\"zoom\":18}"
};
```

### 3. 禁用站点

```csharp
substation.IsEnabled = false;
substation.LastModificationTime = DateTime.Now;
substation.LastModifierId = currentUserId;
```

## 索引建议

```sql
-- 父站点索引（用于查询子站点）
CREATE INDEX IX_SUBSTATION_PARENT_ID ON ast_substation(parent_id);

-- 站点类型索引
CREATE INDEX IX_SUBSTATION_TYPE_ID ON ast_substation(substation_type_id);

-- 启用状态索引（用于过滤）
CREATE INDEX IX_SUBSTATION_ENABLED ON ast_substation(is_enabled);

-- 复合索引（类型 + 启用状态）
CREATE INDEX IX_SUBSTATION_TYPE_ENABLED ON ast_substation(substation_type_id, is_enabled);
```

## 层级查询示例

### 查询所有子站点（递归）

```sql
-- SQL Server 递归 CTE
WITH SubstationTree AS (
    SELECT * FROM ast_substation WHERE Id = @ParentId
    UNION ALL
    SELECT s.* FROM ast_substation s
    INNER JOIN SubstationTree t ON s.ParentId = t.Id
)
SELECT * FROM SubstationTree;
```

```csharp
// C# 递归查询
public async Task<List<SubstationAggregateRoot>> GetChildrenAsync(Guid parentId)
{
    var children = await _repository.GetListAsync(s => s.ParentId == parentId);
    var result = new List<SubstationAggregateRoot>(children);

    foreach (var child in children)
    {
        result.AddRange(await GetChildrenAsync(child.Id));
    }

    return result;
}
```

## 相关文件

- **源码**: `module/ast-intellisub/Ast.IntelliSub.Domain/Entities/Station/SubstationAggregateRoot.cs`
- **服务**: `module/ast-intellisub/Ast.IntelliSub.Application/Services/SubstationService.cs`
- **DTO**: `module/ast-intellisub/Ast.IntelliSub.Application.Contracts/Dtos/Substation/`
- **站点类型**: `Classes/Substation/SubstationTypeEntity.md`
