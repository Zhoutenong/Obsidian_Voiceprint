# AlgorithmService

## 概述
算法管理服务，负责管理系统中所有算法的配置、切换和同步。该服务维护算法库与监测点位的关联关系，并确保算法单位变更时数据的一致性。

## 职责
- 管理算法的增删改查
- 维护算法与监测点位的关联关系
- 同步算法单位变更到相关数据表
- 提供算法选项列表供前端下拉选择
- 按类型和键值查询算法

## 主要接口

### CreateAsync
```csharp
[OperLog("创建算法", OperEnum.Insert)]
public override async Task<AlgorithmDto> CreateAsync(AlgorithmCreateUpdateDto input)
```

**功能说明**：创建新算法

**调用链**：
```
Controller → AlgorithmService.CreateAsync → Repository.InsertAsync
```

**业务逻辑**：
1. 验证算法名称唯一性
2. 创建算法记录
3. 返回创建的算法信息

**验证规则**：
- 算法名称必须唯一
- 算法键值（Key）建议唯一（注释掉的代码）

---

### UpdateAsync
```csharp
[OperLog("更新算法", OperEnum.Update)]
public override async Task<AlgorithmDto> UpdateAsync(Guid id, AlgorithmCreateUpdateDto input)
```

**功能说明**：更新算法信息，并同步单位变更到相关表

**调用链**：
```
Controller → AlgorithmService.UpdateAsync → Repository.UpdateAsync
                                    ↓
                            SyncUnitToRelatedTablesAsync
                                    ↓
                    ┌───────────────┴───────────────┐
                    ↓                               ↓
            更新 ast_point 表              更新 ast_patrol_record_item 表
```

**业务逻辑**：
1. 获取原有算法信息
2. 验证名称唯一性（排除自身）
3. 检查单位（Unit）是否有变化
4. 使用事务确保数据一致性：
   - 更新算法本身
   - 如果单位变更，同步更新相关表
5. 记录操作日志

**同步更新范围**：
- `ast_point` 表中使用该算法的所有点位
- `ast_patrol_record_item` 表中的巡视记录项

---

### DeleteAsync
```csharp
[OperLog("删除算法", OperEnum.Delete)]
public override async Task DeleteAsync(Guid id)
```

**功能说明**：删除算法

**调用链**：
```
Controller → AlgorithmService.DeleteAsync → Repository.DeleteAsync
```

**业务逻辑**：
1. 检查是否有点位正在使用该算法
2. 如果有关联则拒绝删除
3. 执行删除操作

**保护机制**：
- 只有未被任何点位使用的算法才能删除
- 检查 `AstPointEntity.AlgorithmId` 外键关联

---

### GetListAsync
```csharp
public override async Task<PagedResultDto<AlgorithmDto>> GetListAsync(AlgorithmGetListInputDto input)
```

**功能说明**：获取算法列表（支持分页、筛选和排序）

**查询条件**：
- 按名称模糊搜索：`input.Name`
- 按键值模糊搜索：`input.Key`
- 按类型筛选：`input.Type`
- 按值类型筛选：`input.ValueType`
- 按单位搜索：`input.Unit`

**排序**：
- 默认按创建时间倒序
- 支持自定义排序字段

---

### GetAlgorithmsByTypeAsync
```csharp
public async Task<List<AlgorithmDto>> GetAlgorithmsByTypeAsync(AlgorithmTypeEnum type)
```

**功能说明**：根据算法类型获取算法列表

**返回结果**：按名称排序的算法列表

**用途**：为前端提供特定类型的算法选项

---

### GetAlgorithmByKeyAsync
```csharp
public async Task<AlgorithmDto?> GetAlgorithmByKeyAsync(string key)
```

**功能说明**：根据算法键值获取算法

**调用链**：
```
Service → AlgorithmService.GetAlgorithmByKeyAsync → Repository._DbQueryable.FirstAsync
```

**返回结果**：算法信息或 null

**用途**：通过唯一键值快速查找算法

---

### IsKeyExistsAsync
```csharp
public async Task<bool> IsKeyExistsAsync(string key, Guid? excludeId = null)
```

**功能说明**：检查算法键值是否存在

**参数**：
- `key`: 待检查的键值
- `excludeId`: 排除的算法ID（用于更新时检查自身）

**返回结果**：是否存在（true/false）

---

### GetAlgorithmOptionsAsync
```csharp
public async Task<List<AlgorithmOptionDto>> GetAlgorithmOptionsAsync(AlgorithmTypeEnum? type = null)
```

**功能说明**：获取算法选项列表（用于下拉框）

**返回数据**：
```csharp
public class AlgorithmOptionDto
{
    public Guid Id { get; set; }
    public string Name { get; set; }
    public string Key { get; set; }
}
```

**用途**：前端下拉选择框的数据源

---

## 核心方法：SyncUnitToRelatedTablesAsync

```csharp
private async Task SyncUnitToRelatedTablesAsync(Guid algorithmId, string newUnit, string oldUnit)
```

**功能说明**：同步算法单位变更到相关数据表

**同步策略**：
1. **更新 ast_point 表**
   - 查询所有使用该算法的点位
   - 批量更新点位单位
   - 记录更新数量

2. **更新 ast_patrol_record_item 表**
   - 通过点位关联找到相关巡视记录项
   - 批量更新巡视记录项单位
   - 记录更新数量

3. **日志记录**
   - 记录同步操作的详细信息
   - 包含算法ID、更新数量、新旧单位

**异常处理**：
- 任何步骤失败都会抛出 `UserFriendlyException`
- 事务会自动回滚
- 详细错误日志记录

---

## 依赖服务
- `ISqlSugarRepository<AlgorithmEntity, Guid>` - 算法数据仓储
- `ISqlSugarRepository<AstPointEntity, Guid>` - 监测点位仓储
- `ISqlSugarRepository<PatrolRecordItemEntity, Guid>` - 巡视记录项仓储
- `ILogger<AlgorithmService>` - 日志记录

## 业务规则

### 算法名称唯一性
- 创建和更新时必须验证名称唯一性
- 更新时排除当前记录本身

### 删除保护
- 只有未被任何点位使用的算法才能删除
- 删除前检查 `AstPointEntity.AlgorithmId` 外键关联

### 单位同步
- 算法单位变更时自动同步到相关表
- 使用事务确保数据一致性
- 记录详细的同步日志

## 数据实体
```csharp
public class AlgorithmEntity
{
    public Guid Id { get; set; }
    public string Name { get; set; }           // 算法名称
    public string Key { get; set; }            // 算法键值
    public AlgorithmTypeEnum Type { get; set; } // 算法类型
    public DataValueTypeEnum ValueType { get; set; }  // 值类型
    public string Unit { get; set; }           // 单位
    public string Formula { get; set; }        // 计算公式
    public string Description { get; set; }    // 描述
}
```

## 相关文档
- [[AlgorithmEntity]] - 算法实体
- [[AstPointEntity]] - 监测点位实体
- [[PatrolRecordItemEntity]] - 巡视记录项实体
- [[AlgorithmTypeEnum]] - 算法类型枚举
- [[DataValueTypeEnum]] - 数据值类型枚举
- [[IAlgorithmService]] - 服务接口定义
