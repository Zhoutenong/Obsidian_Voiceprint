# 点位绑定关系服务 (PointBindingRelService)

## 概述
点位绑定关系服务负责管理监测点位与物理传感器点位的映射关系，是数据绑定系统的核心关联服务，实现了业务监测点位与实际传感器数据的桥接。

## 职责
- 创建点位绑定关系
- 更新绑定关系配置
- 删除绑定关系
- 查询绑定关系列表
- 验证绑定关系完整性
- 创建或更新绑定关系（Upsert）
- 管理绑定关系状态

## 主要接口

### 创建绑定关系
```csharp
Task<PointBindingRelDto> CreateAsync(PointBindingRelCreateUpdateDto input)
```

**功能说明**：创建新的点位绑定关系，建立监测点位与传感器点位的映射

**请求参数**：
- `MonitoredObjectItemRelId`: 监测对象与监测项关系ID
- `MonitoredPointId`: 监测点位ID
- `DataBindingItemId`: 数据绑定项ID
- `AstPointId`: 传感器点位ID（可选）
- `Status`: 绑定关系状态

**验证流程**：
1. 验证监测项关系是否存在
2. 验证监测点位是否存在
3. 验证监测点位是否属于该监测项
4. 验证数据绑定项是否存在
5. 验证数据绑定项类型是否匹配监测点位
6. 验证传感器点表是否存在（如果提供）
7. 验证绑定关系唯一性

**调用链**：
```
CreateAsync → 验证关联实体 → 检查唯一性 → 保存绑定关系 → 返回DTO
```

**异常情况**：
- `UserFriendlyException`: 监测项关系不存在
- `UserFriendlyException`: 监测点位不存在
- `UserFriendlyException`: 监测点位不属于该监测项
- `UserFriendlyException`: 数据绑定项不存在
- `UserFriendlyException`: 数据绑定项类型不匹配
- `UserFriendlyException`: 传感器点表不存在
- `UserFriendlyException`: 绑定关系已存在

### 更新绑定关系
```csharp
Task<PointBindingRelDto> UpdateAsync(Guid id, PointBindingRelCreateUpdateDto input)
```

**功能说明**：更新已存在的绑定关系配置

**参数说明**：
- `id`: 绑定关系ID
- `input`: 更新的绑定关系数据

**验证流程**：
与创建操作相同的验证流程，但增加：
- 验证绑定关系唯一性时排除自身

**调用链**：
```
UpdateAsync → 验证关联实体 → 检查唯一性(排除自身) → 更新绑定关系 → 返回DTO
```

### 获取绑定关系列表
```csharp
Task<PagedResultDto<PointBindingRelDto>> GetListAsync(PointBindingRelGetListInputDto input)
```

**功能说明**：分页查询点位绑定关系列表，支持多条件筛选

**查询参数**：
- `MonitoredObjectItemRelId`: 监测项关系ID（可选）
- `MonitoredPointId`: 监测点位ID（可选）
- `DataBindingItemId`: 数据绑定项ID（可选）
- `SkipCount`: 跳过记录数
- `MaxResultCount`: 最大返回数量

**查询逻辑**：
```sql
SELECT rel.*, item.*
FROM ast_point_binding_rel rel
LEFT JOIN ast_data_binding_item item ON rel.data_binding_item_id = item.id
WHERE rel.monitored_object_item_rel_id = ?        -- 可选条件
  AND rel.monitored_point_id = ?                  -- 可选条件
  AND rel.data_binding_item_id = ?                 -- 可选条件
ORDER BY rel.creation_time DESC
LIMIT ?, ?
```

**关联查询**：
- 自动关联数据绑定项信息
- 返回绑定项的详细属性（名称、单位、类型等）

### 创建或更新绑定关系
```csharp
[HttpPost("point-binding-rel/create-or-update")]
Task<PointBindingRelDto> CreateOrUpdateAsync(PointBindingRelCreateUpdateDto input)
```

**功能说明**：智能创建或更新绑定关系，根据唯一性约束自动判断操作类型

**请求路径**：`POST /api/app/point-binding-rel/create-or-update`

**操作流程**：
1. 执行完整的验证流程（与创建相同）
2. 查询是否存在相同的绑定关系
3. 如果不存在 → 创建新的绑定关系
4. 如果存在 → 更新现有绑定关系

**使用场景**：
- 配置同步时避免重复创建
- 批量导入时的幂等操作
- 前端无需预先判断是否存在

**操作日志**：
```csharp
[OperLog("创建或更新点位绑定", OperEnum.Update)]
```

## 核心验证逻辑

### 监测项关系验证
```csharp
var monitoredObjectItemRel = await _monitoredObjectItemRelRepository.GetAsync(input.MonitoredObjectItemRelId);
if (monitoredObjectItemRel == null)
{
    throw new UserFriendlyException("监测项关系不存在");
}
```

### 监测点位归属验证
```csharp
var isPointBelongToItem = await _monitoredPointRepository._DbQueryable
    .Where(x => x.Id == input.MonitoredPointId && x.MonitoredItemId == monitoredObjectItemRel.MonitoredItemId)
    .AnyAsync();
if (!isPointBelongToItem)
{
    throw new UserFriendlyException("监测点位不属于该监测项");
}
```

### 数据绑定项类型验证
```csharp
var isBindingItemMatchPoint = await _monitoredPointRepository._DbQueryable
    .Where(x => x.Id == input.MonitoredPointId && x.DataBindingTypeId == dataBindingItem.DataBindingTypeId)
    .AnyAsync();
if (!isBindingItemMatchPoint)
{
    throw new UserFriendlyException("数据绑定项不属于该监测点位的绑定类型");
}
```

### 绑定关系唯一性验证
```csharp
var exists = await _repository._DbQueryable
    .AnyAsync(x => x.MonitoredObjectItemRelId == input.MonitoredObjectItemRelId &&
                  x.MonitoredPointId == input.MonitoredPointId &&
                  x.DataBindingItemId == input.DataBindingItemId);
if (exists)
{
    throw new UserFriendlyException("该点位绑定关系已存在");
}
```

## 数据模型

### 实体定义
```csharp
[SugarTable("ast_point_binding_rel")]
public class PointBindingRelEntity : Entity<Guid>
{
    public Guid MonitoredObjectItemRelId { get; set; }      // 监测项关系ID
    public Guid MonitoredPointId { get; set; }              // 监测点位ID
    public Guid DataBindingItemId { get; set; }              // 数据绑定项ID
    public Guid? AstPointId { get; set; }                   // 传感器点位ID（可选）
    public CommonStatusEnum Status { get; set; }             // 状态
}
```

### 实体关系
```
PointBindingRelEntity (点位绑定关系)
    ↓
    ├── MonitoredObjectItemRelEntity (监测项关系)
    │   └── MonitoredObjectEntity (监测对象)
    │   └── MonitoredItemEntity (监测项)
    │
    ├── MonitoredPointEntity (监测点位)
    │   └── MonitoredItemEntity (监测项)
    │   └── DataBindingTypeEntity (绑定类型)
    │
    ├── DataBindingItemEntity (数据绑定项)
    │   └── DataBindingTypeEntity (绑定类型)
    │   └── Property, Unit, DataValueType...
    │
    └── AstPointEntity (传感器点位) [可选]
        └── DeviceId, SensorKey, Property...
```

### DTO 定义
```csharp
public class PointBindingRelDto
{
    public Guid Id { get; set; }
    public Guid MonitoredObjectItemRelId { get; set; }
    public Guid MonitoredPointId { get; set; }
    public Guid DataBindingItemId { get; set; }
    public Guid? AstPointId { get; set; }
    public CommonStatusEnum Status { get; set; }

    // 关联数据
    public DataBindingItemDto BindingItem { get; set; }
}
```

## 唯一性约束

### 组合唯一键
```
(MonitoredObjectItemRelId, MonitoredPointId, DataBindingItemId)
```

**含义**：
- 同一个监测项关系
- 同一个监测点位
- 同一个数据绑定项
- 只能有一个绑定关系

**业务解释**：
在一个特定的监测对象和监测项组合下，一个监测点位只能绑定一次到同一个数据绑定项。

**示例**：
```
监测对象: 1号变压器
监测项: 温度监测
监测点位: 顶部温度传感器
数据绑定项: 油温

✅ 只能创建一个绑定关系
❌ 重复创建会抛出异常
```

## 业务流程

### 创建绑定关系完整流程

```
前端提交绑定请求
    ↓
验证监测项关系存在
    ↓
验证监测点位存在
    ↓
验证监测点位属于该监测项
    ↓
验证数据绑定项存在
    ↓
验证数据绑定项类型匹配
    ↓
验证传感器点表存在（可选）
    ↓
验证绑定关系唯一性
    ↓
创建绑定关系记录
    ↓
返回创建结果
```

### 数据流转路径

```
传感器数据采集 (MQTT)
    ↓
AstPointId (传感器点位)
    ↓
PointBindingRel (点位绑定关系)
    ↓
MonitoredPoint (监测点位)
    ↓
MonitoredItem (监测项)
    ↓
DataBindingItem (数据绑定项)
    ↓
业务属性数据 (温度、压力等)
```

## 依赖服务

### 核心仓储
- `ISqlSugarRepository<PointBindingRelEntity>` - 点位绑定关系仓储
- `ISqlSugarRepository<MonitoredObjectItemRelEntity>` - 监测项关系仓储
- `ISqlSugarRepository<MonitoredPointEntity>` - 监测点位仓储
- `ISqlSugarRepository<DataBindingItemEntity>` - 数据绑定项仓储
- `ISqlSugarRepository<AstPointEntity>` - 传感器点位仓储

### 关联服务
- [[MonitoredObjectService]] - 监测对象服务
- [[MonitoredPointService]] - 监测点位服务
- [[DataBindingItemService]] - 数据绑定项服务
- [[PointValueProcessingService]] - 点位值处理服务

## API 端点

### CRUD 接口
- `POST /api/app/point-binding-rel` - 创建绑定关系
- `PUT /api/app/point-binding-rel/{id}` - 更新绑定关系
- `DELETE /api/app/point-binding-rel/{id}` - 删除绑定关系
- `GET /api/app/point-binding-rel/{id}` - 获取单个绑定关系
- `GET /api/app/point-binding-rel` - 获取绑定关系列表

### 扩展接口
- `POST /api/app/point-binding-rel/create-or-update` - 创建或更新绑定关系

## 权限要求

```csharp
[Authorize]
public class PointBindingRelService : YiCrudAppService<...>
```
- 所有接口需要授权访问
- 支持基于角色的权限控制
- 操作日志记录（创建或更新操作）

## 注意事项

### 数据完整性
- **强验证**: 所有关联实体都进行存在性验证
- **类型匹配**: 严格检查绑定项类型与点位类型的一致性
- **唯一性**: 通过组合唯一键防止重复绑定

### 性能考虑
- **批量查询**: 使用 LeftJoin 减少数据库往返
- **索引优化**: 建议在唯一键字段上创建复合索引
- **分页查询**: 支持大数据量的分页加载

### 错误处理
- **友好提示**: 使用 UserFriendlyException 提供清晰的错误信息
- **验证顺序**: 按业务逻辑顺序验证，快速失败
- **日志记录**: 关键操作都有操作日志

### 扩展性
- **可选传感器点位**: AstPointId 可选，支持不同数据源
- **状态管理**: 支持启用/禁用绑定关系
- **软删除**: 基础 CRUD 服务支持软删除

## 相关文档

- [[DataBindingItemService]] - 数据绑定项服务
- [[PointValueProcessingService]] - 点位值处理服务
- [[MonitoredPointService]] - 监测点位服务
- [[数据绑定架构]] - 数据绑定整体架构
- [[点位绑定流程]] - 完整的点位绑定流程说明
