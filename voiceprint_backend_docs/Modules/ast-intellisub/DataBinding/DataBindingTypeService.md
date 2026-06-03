# 数据绑定类型服务 (DataBindingTypeService)

## 概述
数据绑定类型服务负责管理系统中数据绑定类型的定义，将不同类型的数据绑定方式与前端展示组件关联起来，定义数据的展示组件和处理方式。

## 职责
- 管理数据绑定类型的创建、更新、删除
- 维护绑定类型与显示组件的关联关系
- 确保绑定类型名称的唯一性
- 提供绑定类型的查询和列表功能

## 主要接口

### 创建绑定类型
```csharp
Task<DataBindingTypeDto> CreateAsync(DataBindingTypeCreateUpdateDto input)
```

**功能说明**：创建新的数据绑定类型

**请求参数**：
- `Name`: 绑定类型名称（必须唯一）
- `DisplayComponentId`: 关联的显示组件ID
- `Description`: 类型描述

**业务规则**：
- 绑定类型名称必须全局唯一
- 显示组件必须已存在

**异常**：
- `UserFriendlyException`: 绑定类型名称已存在
- `UserFriendlyException`: 显示组件不存在

### 更新绑定类型
```csharp
Task<DataBindingTypeDto> UpdateAsync(Guid id, DataBindingTypeCreateUpdateDto input)
```

**功能说明**：更新现有的数据绑定类型

**请求参数**：
- `id`: 绑定类型ID
- `Name`: 绑定类型名称（必须唯一）
- `DisplayComponentId`: 关联的显示组件ID
- `Description`: 类型描述

**业务规则**：
- 绑定类型名称必须全局唯一（排除自身）
- 显示组件必须已存在

**异常**：
- `UserFriendlyException`: 绑定类型名称已存在
- `UserFriendlyException`: 显示组件不存在

### 删除绑定类型
```csharp
Task DeleteAsync(Guid id)
```

**功能说明**：删除指定的数据绑定类型

**请求参数**：
- `id`: 绑定类型ID

**调用链**：
```
Controller → DataBindingTypeService.DeleteAsync → SqlSugarRepository.DeleteAsync
```

### 获取绑定类型列表
```csharp
Task<PagedResultDto<DataBindingTypeDto>> GetListAsync(DataBindingTypeGetListInputDto input)
```

**功能说明**：分页查询数据绑定类型列表

**请求参数**：
- `Keyword`: 搜索关键字（支持按名称和描述搜索）
- `SkipCount`: 跳过记录数
- `MaxResultCount`: 最大返回记录数

**返回数据**：
- `Items`: 绑定类型列表
- `TotalCount`: 总记录数

**排序**：按创建时间升序

### 获取所有绑定类型
```csharp
Task<List<DataBindingTypeDto>> GetAllListAsync(DataBindingTypeGetListInputDto input)
```

**功能说明**：获取所有匹配的数据绑定类型（不分页）

**请求参数**：
- `Keyword`: 搜索关键字（支持按名称和描述搜索）

**返回数据**：完整的绑定类型列表

## 数据实体

### DataBindingTypeAggregateRoot
```csharp
public class DataBindingTypeAggregateRoot : AggregateRoot<Guid>
{
    public Guid Id { get; set; }
    public string Name { get; set; }
    public Guid DisplayComponentId { get; set; }
    public string Description { get; set; }
    public DateTime CreationTime { get; set; }
    public DateTime? LastModificationTime { get; set; }
    
    [Navigate(NavigateType.OneToOne)]
    public DisplayComponentEntity DisplayComponent { get; set; }
}
```

**数据库表**：`ast_data_binding_type`

## 依赖服务

- [[DisplayComponentService]] - 显示组件服务
- [[DataBindingItemService]] - 数据绑定项服务
- [[DataStrategyService]] - 数据策略服务

## 业务流程

### 创建绑定类型流程
```
1. 验证绑定类型名称唯一性
2. 验证显示组件是否存在
3. 创建绑定类型记录
4. 返回创建的绑定类型信息
```

### 更新绑定类型流程
```
1. 验证绑定类型名称唯一性（排除自身）
2. 验证显示组件是否存在
3. 更新绑定类型记录
4. 返回更新后的绑定类型信息
```

## 与其他模块的关系

```
DataBindingType (绑定类型)
    ↓ 1:N
DataBindingItem (绑定项)
    ↓ N:N
DataStrategy (数据策略)
```

## API 路径

- `POST /api/app/data-binding-types` - 创建绑定类型
- `PUT /api/app/data-binding-types/{id}` - 更新绑定类型
- `DELETE /api/app/data-binding-types/{id}` - 删除绑定类型
- `GET /api/app/data-binding-types` - 获取绑定类型列表（分页）
- `GET /api/app/data-binding-types/all` - 获取所有绑定类型

## 权限要求

- 所有接口都需要 `[Authorize]` 认证
- 创建操作记录操作日志：`OperLog("创建数据绑定类型", OperEnum.Insert)`
- 更新操作记录操作日志：`OperLog("更新数据绑定类型", OperEnum.Update)`
- 删除操作记录操作日志：`OperLog("删除数据绑定类型", OperEnum.Delete)`

## 相关文档

- [[DataBindingItemService]] - 数据绑定项服务
- [[DisplayComponentService]] - 显示组件服务
- [[DataStrategyService]] - 数据策略服务
- [[BindingItemStrategyRelService]] - 绑定项策略关联服务
