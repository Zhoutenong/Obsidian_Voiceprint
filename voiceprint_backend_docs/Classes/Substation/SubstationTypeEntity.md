# SubstationTypeEntity

## 概述

`SubstationTypeEntity` 定义变电站的类型和分类，用于站点分组、图标展示和业务逻辑区分。

## 表信息

- **表名**: `ast_substation_type`
- **主键**: `Id` (Guid)
- **继承**: `Entity<Guid>`
- **命名空间**: `Ast.IntelliSub.Domain.Entities`

## 字段列表

| 字段名 | 类型 | 数据库列名 | 说明 | 约束 |
|--------|------|-----------|------|------|
| `Id` | `Guid` | `id` | 主键 | PK |
| `Name` | `string` | `name` | 类型名称 | NOT NULL |
| `Category` | `SubstationCategoryEnum` | `category` | 分类 | Enum |
| `Description` | `string?` | `description` | 描述 | |
| `Icon` | `string?` | `icon` | 图标 | 图标名称/URL |

## 关联实体 (ER 关系)

### 关系图

```
SubstationTypeEntity (1) ←──→ (N) SubstationAggregateRoot
```

所有使用该类型的站点都通过 `SubstationTypeId` 外键关联。

## 被哪些服务读写

### 写入服务

- `SubstationTypeService` — 站点类型 CRUD

### 读取服务

- `SubstationService` — 创建/更新站点时选择类型
- `ReportService` — 按站点类型统计报告

## 业务规则约束

### 1. 枚举约束

#### SubstationCategoryEnum

```csharp
public enum SubstationCategoryEnum
{
    Group = 0,    // 站点分组（如：华北地区、华东地区）
    Substation = 1 // 站点（如：朝阳变电站、丰台变电站）
}
```

### 2. 分类使用规则

- **Group (0)**: 用于地区或区域分组
  - 不直接关联设备
  - 仅用于层级组织
  - 示例：`"北京市"`, `"上海市"`

- **Substation (1)**: 实际变电站
  - 关联网关和设备
  - 执行巡检任务
  - 示例：`"朝阳变电站"`, `"丰台变电站"`

### 3. 图标格式

`Icon` 字段支持多种格式：

#### 1. Iconify 图标名称（推荐）

```csharp
Icon = "mdi:transmission-tower"
Icon = "mdi:factory"
Icon = "mdi:city"
```

#### 2. 自定义图标 URL

```csharp
Icon = "/icons/substation/custom.svg"
Icon = "https://example.com/icons/nuclear-plant.png"
```

#### 3. Base64 图标（不推荐）

```csharp
Icon = "data:image/svg+xml;base64,PHN2ZyB..."
```

### 4. 删除约束

- 删除类型前需检查是否有站点使用
- 建议使用软删除或标记为 `IsDisabled`

## 使用场景

### 1. 创建地区分组

```csharp
var region = new SubstationTypeEntity
{
    Id = Guid.NewGuid(),
    Name = "北京市",
    Category = SubstationCategoryEnum.Group,
    Description = "北京市及周边地区变电站分组",
    Icon = "mdi:city"
};
```

### 2. 创建变电站类型

```csharp
var substationType = new SubstationTypeEntity
{
    Id = Guid.NewGuid(),
    Name = "220kV变电站",
    Category = SubstationCategoryEnum.Substation,
    Description = "220千伏电压等级变电站",
    Icon = "mdi:transmission-tower"
};
```

### 3. 查询所有分组

```csharp
public async Task<List<SubstationTypeEntity>> GetGroupsAsync()
{
    return await _substationTypeRepo.GetListAsync(
        t => t.Category == SubstationCategoryEnum.Group
    );
}
```

### 4. 查询所有变电站类型

```csharp
public async Task<List<SubstationTypeEntity>> GetSubstationTypesAsync()
{
    return await _substationTypeRepo.GetListAsync(
        t => t.Category == SubstationCategoryEnum.Substation
    );
}
```

## 预置数据

系统初始化时应创建以下基础类型：

```sql
INSERT INTO ast_substation_type (id, name, category, description, icon) VALUES
-- 分组
('GUID_1', '北京市', 0, '北京市地区', 'mdi:city'),
('GUID_2', '上海市', 0, '上海市地区', 'mdi:city'),
('GUID_3', '广州市', 0, '广州市地区', 'mdi:city'),

-- 变电站类型
('GUID_10', '500kV变电站', 1, '500千伏变电站', 'mdi:transmission-tower'),
('GUID_11', '220kV变电站', 1, '220千伏变电站', 'mdi:transmission-tower'),
('GUID_12', '110kV变电站', 1, '110千伏变电站', 'mdi:transmission-tower'),
('GUID_13', '35kV变电站', 1, '35千伏变电站', 'mdi:transmission-tower'),
('GUID_14', '10kV开关站', 1, '10千伏开关站', 'mdi:electrical-posts');
```

## 索引建议

```sql
-- 分类索引（用于查询分组或变电站）
CREATE INDEX IX_SUBSTATION_TYPE_CATEGORY ON ast_substation_type(category);

-- 名称索引（用于搜索）
CREATE INDEX IX_SUBSTATION_TYPE_NAME ON ast_substation_type(name);
```

## 前端展示建议

### 1. 树形结构

```typescript
interface SubstationTypeNode {
  id: string;
  name: string;
  category: 'Group' | 'Substation';
  icon?: string;
  children?: SubstationTypeNode[];
}
```

### 2. 图标渲染

```tsx
import { Icon } from '@iconify/react';

function SubstationTypeIcon({ iconName }: { iconName: string }) {
  return <Icon icon={iconName} width={24} height={24} />;
}
```

### 3. 下拉选择

```tsx
<Select>
  <SelectGroup label="地区分组">
    {groups.map(g => (
      <SelectItem key={g.id} value={g.id}>
        {g.name}
      </SelectItem>
    ))}
  </SelectGroup>
  <SelectGroup label="变电站类型">
    {substations.map(s => (
      <SelectItem key={s.id} value={s.id}>
        {s.name}
      </SelectItem>
    ))}
  </SelectGroup>
</Select>
```

## 相关文件

- **源码**: `module/ast-intellisub/Ast.IntelliSub.Domain/Entities/Station/SubstationTypeEntity.cs`
- **服务**: `module/ast-intellisub/Ast.IntelliSub.Application/Services/SubstationTypeService.cs`
- **枚举**: `module/ast-intellisub/Ast.IntelliSub.Domain.Shared/Enums/SubstationCategoryEnum.cs`
- **站点实体**: `Classes/Substation/SubstationAggregateRoot.md`
