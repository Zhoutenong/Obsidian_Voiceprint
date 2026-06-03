# SubstationTypeService

## 概述
变电站类型服务，负责管理变电站的类型定义，如变电站、配电站、开关站等分类信息。该服务提供变电站类型的CRUD操作，并确保数据完整性和业务规则验证。

## 职责
- 管理变电站类型的增删改查
- 验证类型名称的唯一性
- 保护有关联站点的类型不被删除
- 支持按分类筛选和关键字搜索

## 主要接口

### CreateAsync
```csharp
[OperLog("创建站点类型", OperEnum.Insert)]
public override async Task<SubstationTypeDto> CreateAsync(SubstationTypeCreateUpdateDto input)
```

**功能说明**：创建新的变电站类型

**调用链**：
```
Controller → SubstationTypeService.CreateAsync → Repository.InsertAsync
```

**业务逻辑**：
1. 验证类型名称唯一性
2. 创建类型记录
3. 返回创建的类型信息

**异常处理**：
- `UserFriendlyException`: 当类型名称已存在时抛出

---

### UpdateAsync
```csharp
[OperLog("更新站点类型", OperEnum.Update)]
public override async Task<SubstationTypeDto> UpdateAsync(Guid id, SubstationTypeCreateUpdateDto input)
```

**功能说明**：更新变电站类型信息

**调用链**：
```
Controller → SubstationTypeService.UpdateAsync → Repository.UpdateAsync
```

**业务逻辑**：
1. 验证名称唯一性（排除自身）
2. 更新类型记录
3. 返回更新后的类型信息

**异常处理**：
- `UserFriendlyException`: 当类型名称已存在时抛出

---

### DeleteAsync
```csharp
[OperLog("删除站点类型", OperEnum.Delete)]
public override async Task DeleteAsync(Guid id)
```

**功能说明**：删除变电站类型

**调用链**：
```
Controller → SubstationTypeService.DeleteAsync → Repository.DeleteAsync
```

**业务逻辑**：
1. 检查是否有关联的变电站
2. 如果有关联则拒绝删除
3. 执行删除操作

**异常处理**：
- `UserFriendlyException`: 当类型下存在变电站时抛出

---

### GetListAsync
```csharp
public override async Task<PagedResultDto<SubstationTypeDto>> GetListAsync(SubstationTypeGetListInputDto input)
```

**功能说明**：获取变电站类型列表（支持分页、筛选和搜索）

**调用链**：
```
Controller → SubstationTypeService.GetListAsync → Repository._DbQueryable
```

**查询条件**：
- 按名称模糊搜索：`input.Name`
- 按分类筛选：`input.Category`

**返回数据**：
- 分页的类型列表
- 包含ID、名称、分类、描述、图标等信息

---

## 依赖服务
- `ISqlSugarRepository<SubstationTypeEntity, Guid>` - 类型数据仓储
- `ISqlSugarRepository<SubstationAggregateRoot, Guid>` - 变电站数据仓储
- `ILogger<SubstationTypeService>` - 日志记录

## 业务规则

### 名称唯一性
- 创建和更新时必须验证名称唯一性
- 更新时排除当前记录本身

### 删除保护
- 只有未被任何变电站使用的类型才能删除
- 删除前检查 `SubstationAggregateRoot.SubstationTypeId` 外键关联

## 数据实体
```csharp
public class SubstationTypeEntity
{
    public Guid Id { get; set; }
    public string Name { get; set; }        // 类型名称
    public SubstationTypeEnum? Category { get; set; }  // 分类枚举
    public string Description { get; set; }  // 描述
    public string Icon { get; set; }         // 图标
}
```

## 相关文档
- [[SubstationAggregateRoot]] - 变电站聚合根
- [[SubstationTypeEnum]] - 变电站类型枚举
- [[ISubstationTypeService]] - 服务接口定义
