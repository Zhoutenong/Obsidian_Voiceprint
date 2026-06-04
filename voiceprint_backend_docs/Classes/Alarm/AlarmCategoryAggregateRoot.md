# AlarmCategoryAggregateRoot

## 概述

`AlarmCategoryAggregateRoot` 定义告警类别，用于对告警记录进行分类管理和统计分析。告警类别可以是设备类型（如：温度、振动）或业务类型（如：巡检异常、设备故障）。

## 表信息

- **表名**: `ast_alarm_category`
- **主键**: `Id` (Guid)
- **继承**: `AggregateRoot<Guid>` + `IAuditedObject`
- **命名空间**: `Ast.IntelliSub.Domain.Entities`

## 字段列表

| 字段名 | 类型 | 数据库列名 | 说明 | 约束 |
|--------|------|-----------|------|------|
| `Id` | `Guid` | `id` | 主键 | PK |
| `Name` | `string` | `name` | 类别名称 | NOT NULL, Length=100 |
| `OrderNum` | `int` | `order_num` | 排序号 | 用于前端展示顺序 |
| `Type` | `AlarmCategoryTypeEnum` | `type` | 类别类型 | Enum |
| `CreationTime` | `DateTime` | `creation_time` | 创建时间 | |
| `CreatorId` | `Guid?` | `creator_id` | 创建者ID | FK (User) |
| `LastModificationTime` | `DateTime?` | `last_modification_time` | 最后修改时间 | |
| `LastModifierId` | `Guid?` | `last_modifier_id` | 最后修改者ID | FK (User) |

## 关联实体 (ER 关系)

### 关系图

```
AlarmCategoryAggregateRoot (1) ←──→ (N) AlarmRecordItemEntity
```

告警记录项通过 `AlarmCategoryName` 字段关联告警类别（非外键，冗余存储）。

## 被哪些服务读写

### 写入服务

- `AlarmCategoryService` — 告警类别 CRUD

### 读取服务

- `AlarmRecordService` — 告警查询时按类别筛选
- `ReportService` — 告警统计报告按类别分组
- `DashboardService` — 告警数据大屏展示

## 业务规则约束

### 1. 枚举约束

#### AlarmCategoryTypeEnum

```csharp
public enum AlarmCategoryTypeEnum
{
    Device = 0,        // 设备类（温度、振动、声音等）
    Patrol = 1,       // 巡检类（巡检异常、未按时巡检）
    System = 2,       // 系统类（网络故障、存储不足）
    Security = 3      // 安防类（非法入侵、门禁异常）
}
```

### 2. 名称唯一性

```sql
-- 类别名称在同一类型下唯一
CREATE UNIQUE INDEX UX_ALARM_CATEGORY_NAME_TYPE ON ast_alarm_category(name, type);
```

### 3. 排序规则

- `OrderNum` 值越小，展示顺序越靠前
- 同一级别内按 `OrderNum` 升序排列
- 建议步长为 10，便于插入新类别

### 4. 删除约束

- 删除类别前需检查是否有告警记录使用
- 建议使用软删除或标记为 `IsDisabled`

## 使用场景

### 1. 创建设备类告警类别

```csharp
var deviceCategory = new AlarmCategoryAggregateRoot
{
    Id = Guid.NewGuid(),
    Name = "温度告警",
    OrderNum = 10,
    Type = AlarmCategoryTypeEnum.Device,
    CreationTime = DateTime.Now,
    CreatorId = adminUserId
};
```

### 2. 创建巡检类告警类别

```csharp
var patrolCategory = new AlarmCategoryAggregateRoot
{
    Id = Guid.NewGuid(),
    Name = "巡检异常",
    OrderNum = 20,
    Type = AlarmCategoryTypeEnum.Patrol,
    CreationTime = DateTime.Now,
    CreatorId = adminUserId
};
```

### 3. 查询所有类别

```csharp
public async Task<List<AlarmCategoryAggregateRoot>> GetAllAsync()
{
    return await _repository.GetListAsync(
        orderBy: c => c.OrderNum,
        orderByOrderByType: OrderByType.Asc
    );
}
```

### 4. 按类型查询

```csharp
public async Task<List<AlarmCategoryAggregateRoot>> GetByTypeAsync(AlarmCategoryTypeEnum type)
{
    return await _repository.GetListAsync(
        c => c.Type == type,
        orderBy: c => c.OrderNum,
        orderByOrderByType: OrderByType.Asc
    );
}
```

## 预置数据

系统初始化时应创建以下基础告警类别：

```sql
INSERT INTO ast_alarm_category (id, name, order_num, type, creation_time, creator_id) VALUES
-- 设备类告警
('GUID_1', '温度告警', 10, 0, GETDATE(), 'SYSTEM'),
('GUID_2', '振动告警', 20, 0, GETDATE(), 'SYSTEM'),
('GUID_3', '声音异常', 30, 0, GETDATE(), 'SYSTEM'),
('GUID_4', '设备离线', 40, 0, GETDATE(), 'SYSTEM'),

-- 巡检类告警
('GUID_10', '巡检异常', 100, 1, GETDATE(), 'SYSTEM'),
('GUID_11', '未按时巡检', 110, 1, GETDATE(), 'SYSTEM'),
('GUID_12', '预置位偏差', 120, 1, GETDATE(), 'SYSTEM'),

-- 系统类告警
('GUID_20', '网络故障', 200, 2, GETDATE(), 'SYSTEM'),
('GUID_21', '存储不足', 210, 2, GETDATE(), 'SYSTEM'),
('GUID_22', '服务异常', 220, 2, GETDATE(), 'SYSTEM'),

-- 安防类告警
('GUID_30', '非法入侵', 300, 3, GETDATE(), 'SYSTEM'),
('GUID_31', '门禁异常', 310, 3, GETDATE(), 'SYSTEM');
```

## 索引建议

```sql
-- 唯一约束（名称 + 类型）
CREATE UNIQUE INDEX UX_ALARM_CATEGORY_NAME_TYPE ON ast_alarm_category(name, type);

-- 类型索引
CREATE INDEX IX_ALARM_CATEGORY_TYPE ON ast_alarm_category(type);

-- 排序索引
CREATE INDEX IX_ALARM_CATEGORY_ORDER ON ast_alarm_category(order_num);
```

## 告警统计示例

### 按类别统计告警数量

```sql
SELECT
    ac.name AS category_name,
    ac.type AS category_type,
    COUNT(ari.id) AS alarm_count
FROM ast_alarm_category ac
LEFT JOIN ast_alarm_record_item ari ON ari.alarm_category_name = ac.name
GROUP BY ac.id, ac.name, ac.type
ORDER BY ac.order_num;
```

### C# 实现

```csharp
public async Task<List<CategoryStatisticsDto>> GetStatisticsAsync()
{
    var categories = await _repository.GetListAsync();
    var result = new List<CategoryStatisticsDto>();

    foreach (var category in categories)
    {
        var count = await _alarmRecordRepo.CountAsync(
            a => a.AlarmCategoryName == category.Name
        );

        result.Add(new CategoryStatisticsDto
        {
            CategoryId = category.Id,
            CategoryName = category.Name,
            CategoryType = category.Type,
            AlarmCount = count
        });
    }

    return result.OrderBy(r => r.OrderNum).ToList();
}
```

## 前端展示建议

### 1. 下拉选择

```tsx
<Select>
  <SelectGroup label="设备告警">
    {deviceCategories.map(c => (
      <SelectItem key={c.id} value={c.id}>
        {c.name}
      </SelectItem>
    ))}
  </SelectGroup>
  <SelectGroup label="巡检告警">
    {patrolCategories.map(c => (
      <SelectItem key={c.id} value={c.id}>
        {c.name}
      </SelectItem>
    ))}
  </SelectGroup>
</Select>
```

### 2. 统计图表

```tsx
<BarChart data={statistics}>
  <Bar dataKey="alarmCount" name="告警数量" />
  <XAxis dataKey="categoryName" />
</BarChart>
```

## 相关文件

- **源码**: `module/ast-intellisub/Ast.IntelliSub.Domain/Entities/Alarm/AlarmCategoryAggregateRoot.cs`
- **服务**: `module/ast-intellisub/Ast.IntelliSub.Application/Services/AlarmCategoryService.cs`
- **告警记录**: `Classes/Alarm/AlarmRecordItemEntity.md`
- **枚举**: `module/ast-intellisub/Ast.IntelliSub.Domain.Shared/Enums/AlarmCategoryTypeEnum.cs`
