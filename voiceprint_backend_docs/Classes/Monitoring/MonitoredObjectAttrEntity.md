# MonitoredObjectAttrEntity — 监测对象属性实体

## 基本信息

- **实体名称**：`MonitoredObjectAttrEntity`
- **数据库表**：`ast_monitored_object_attr`
- **模块位置**：`module/ast-intellisub/Ast.IntelliSub.Domain/Entities/Monitoring/`
- **继承关系**：`Entity<Guid>`

## 实体说明

监测对象属性实体用于存储监测对象的静态属性信息，如设备型号、生产厂家、额定参数等。属性可以按分组组织，支持数值型和字符串型两种值类型。

## 字段说明

### 主键与关联

| 字段名 | 数据类型 | 说明 | 约束 |
|-------|---------|------|------|
| `Id` | `Guid` | 主键 | Primary Key |
| `MonitoredObjectAttrGroupId` | `Guid` | 属性分组ID | Foreign Key |

### 属性信息

| 字段名 | 数据类型 | 说明 | 备注 |
|-------|---------|------|------|
| `Name` | `string` | 属性名称 | 如："额定电压"、"生产厂家" |
| `Val` | `long?` | 属性值（数值型） | 存储数值类型的属性值 |
| `StrVal` | `string?` | 属性值（字符串型） | 存储字符串类型的属性值 |
| `ValueType` | `AttrValueTypeEnum` | 值类型 | 见下方枚举 |
| `Description` | `string?` | 描述 | 属性的详细说明 |
| `CreationTime` | `DateTime?` | 创建时间 | - |

## 枚举类型

### AttrValueTypeEnum — 属性值类型
```csharp
public enum AttrValueTypeEnum
{
    Long = 0,    // 长整型（使用Val字段）
    String = 1   // 字符串型（使用StrVal字段）
}
```

## 导航属性

```csharp
[Navigate(NavigateType.OneToOne, nameof(MonitoredObjectAttrGroupId))]
public MonitoredObjectAttrGroupEntity AttrGroup { get; set; }
```

## 业务规则

### 属性存储规则
1. **类型区分**：通过 ValueType 决定使用 Val 还是 StrVal 字段
2. **分组组织**：通过 MonitoredObjectAttrGroupId 将属性分组管理
3. **灵活扩展**：可以动态添加任意数量的属性
4. **显示优化**：属性分组提升前端展示效果

### 属性类型选择
- **数值型（Long）**：适用于额定电压、额定容量、运行年限等
- **字符串型（String）**：适用于生产厂家、设备型号、备注信息等

## 相关实体

- [[MonitoredObjectAttrGroupEntity]] — 监测对象属性分组
- [[MonitoredObjectAggregateRoot]] — 监测对象聚合根

## 相关服务

- [[MonitoredObjectAttrService]] — 监测对象属性服务
- [[MonitoredObjectAttrGroupService]] — 属性分组服务

## 数据查询示例

### 查询某对象的所有属性
```sql
SELECT
    grp.name as group_name,
    grp.order_num as group_order,
    attr.name as attr_name,
    CASE attr.value_type
        WHEN 0 THEN CAST(attr.val AS VARCHAR)
        WHEN 1 THEN attr.str_val
    END as attr_value,
    attr.description
FROM ast_monitored_object_attr attr
JOIN ast_monitored_object_attr_group grp ON attr.monitored_object_attr_group_id = grp.id
WHERE grp.monitored_object_id = 'xxx'
ORDER BY grp.order_num, attr.name;
```

### 查询某分组的属性
```sql
SELECT * FROM ast_monitored_object_attr
WHERE monitored_object_attr_group_id = 'xxx'
ORDER BY name;
```

### 查询包含特定值的属性
```sql
-- 查询所有属性值包含"某某"的属性
SELECT * FROM ast_monitored_object_attr
WHERE str_val LIKE '%某某%'
AND value_type = 1;
```

## 索引建议

```sql
-- 属性分组ID索引（用于查询某分组的属性）
CREATE INDEX IX_attr_group_id ON ast_monitored_object_attr(monitored_object_attr_group_id);
```

## 使用场景

### 场景1：设备基础属性分组
```
分组：基础信息（OrderNum: 10）
├── 设备名称：1号主变压器
├── 设备型号：SFZ11-50000/110
├── 额定容量：50000（数值型）
├── 额定电压：110（数值型）
└── 生产厂家：某某变压器有限公司
```

### 场景2：技术参数分组
```
分组：技术参数（OrderNum: 20）
├── 额定电流：262（数值型）
├── 额定频率：50（数值型）
├── 冷却方式：ONAN
└── 联结组别：YNd11
```

### 场景3：运行数据分组
```
分组：运行数据（OrderNum: 30）
├── 投运日期：2020-01-15
├── 上次检修：2024-03-20
├── 运行年限：6（数值型）
└── 健康状态：良好
```

## 业务价值

**核心作用**：
1. **静态信息管理**：存储设备/对象的静态属性信息
2. **分组展示**：支持按分组组织属性，提升用户体验
3. **类型灵活**：支持数值型和字符串型，适应不同属性需求
4. **动态扩展**：可以灵活添加自定义属性

## 设计模式

### 值对象模式
- 属性值根据类型选择不同的存储字段
- 通过 ValueType 区分使用哪个字段
- 在应用层进行统一封装

### 分组聚合
```csharp
public class MonitoredObjectAttrGroupDto
{
    public Guid GroupId { get; set; }
    public string GroupName { get; set; }
    public int OrderNum { get; set; }
    public List<MonitoredObjectAttrDto> Attributes { get; set; }
}
```

## 前端展示建议

### 分组折叠面板
```tsx
{attributeGroups.map(group => (
  <Collapse key={group.id} defaultOpen>
    <Collapse.Header>
      <Icon name={group.icon} />
      {group.name}
    </Collapse.Header>
    <Collapse.Content>
      {group.attributes.map(attr => (
        <AttributeItem
          key={attr.id}
          name={attr.name}
          value={attr.valueType === 0 ? attr.val : attr.strVal}
          description={attr.description}
        />
      ))}
    </Collapse.Content>
  </Collapse>
))}
```

### 属性表格
```tsx
<Table>
  {attributes.map(attr => (
    <TableRow key={attr.id}>
      <TableCell>{attr.name}</TableCell>
      <TableCell>
        {attr.valueType === 0 
          ? attr.val 
          : attr.strVal}
      </TableCell>
      <TableCell>{attr.description}</TableCell>
    </TableRow>
  ))}
</Table>
```

## 性能考虑

### 批量查询
- 一次查询对象的所有属性和分组
- 避免N+1查询问题
- 使用内存缓存提升性能

### 缓存策略
```csharp
// 缓存对象属性，减少数据库查询
public async Task<List<MonitoredObjectAttrGroupDto>> GetObjectAttributesAsync(Guid objectId)
{
    var cacheKey = $"object_attrs_{objectId}";
    
    var cached = await _cache.GetAsync<List<MonitoredObjectAttrGroupDto>>(cacheKey);
    if (cached != null) return cached;
    
    // 查询数据库
    var result = await QueryObjectAttributesAsync(objectId);
    
    // 缓存1小时
    await _cache.SetAsync(cacheKey, result, TimeSpan.FromHours(1));
    
    return result;
}
```

---

> **最后更新**：2026-06-04
> **源码位置**：`module/ast-intellisub/Ast.IntelliSub.Domain/Entities/Monitoring/MonitoredObjectAttrEntity.cs`