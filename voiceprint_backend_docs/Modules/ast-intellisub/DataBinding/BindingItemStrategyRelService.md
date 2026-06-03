# 绑定项策略关联服务 (BindingItemStrategyRelService)

## 概述
绑定项策略关联服务负责管理数据绑定项与数据策略之间的多对多关联关系，实现绑定项与策略的灵活组合。

## 职责
- 创建绑定项与策略的关联
- 更新关联关系
- 删除关联关系
- 查询关联列表
- 验证关联关系唯一性
- 级联管理相关策略状态

## 主要接口

### 获取关联列表
```csharp
Task<PagedResultDto<BindingItemStrategyRelDto>> GetListAsync(BindingItemStrategyRelGetListInputDto input)
```

**请求参数**：
- `DataBindingItemId`: 数据绑定项ID（可选）
- `DataStrategyId`: 数据策略ID（可选）
- `SkipCount`: 跳过记录数
- `MaxResultCount`: 最大返回数量

### 获取单个关联
```csharp
Task<BindingItemStrategyRelDto> GetAsync(Guid id)
```

### 创建关联
```csharp
Task<BindingItemStrategyRelDto> CreateAsync(BindingItemStrategyRelCreateUpdateDto input)
```

**请求参数**：
- `DataBindingItemId`: 数据绑定项ID
- `DataStrategyId`: 数据策略ID

**验证规则**：
- 数据绑定项必须存在
- 数据策略必须存在
- 同一绑定项与策略的关联关系必须唯一

### 更新关联
```csharp
Task<BindingItemStrategyRelDto> UpdateAsync(Guid id, BindingItemStrategyRelCreateUpdateDto input)
```

**验证规则**：
- 数据绑定项必须存在
- 数据策略必须存在
- 排除自身后，关联关系必须唯一

### 删除关联
```csharp
Task DeleteAsync(Guid id)
```

**级联操作**：
- 删除关联时会同步清理相关的策略状态记录

## 数据处理流程

```
创建/更新关联 → 验证绑定项存在 → 验证策略存在 → 验证唯一性 → 保存关联 → 记录策略状态
```

### 关联创建流程
1. 验证数据绑定项是否存在
2. 验证数据策略是否存在
3. 检查关联关系唯一性
4. 创建关联记录
5. 返回完整的关联信息

### 关联删除流程
1. 查询关联记录
2. 级联删除相关策略状态
3. 删除关联记录
4. 记录操作日志

## 依赖服务

- [[DataBindingItemService]] - 数据绑定项服务
- [[DataStrategyService]] - 数据策略服务
- [[StrategyStateService]] - 策略状态服务

## 相关实体

- **BindingItemStrategyRelEntity** - 绑定项策略关联实体
  - `Id`: 关联ID
  - `DataBindingItemId`: 数据绑定项ID
  - `DataStrategyId`: 数据策略ID
  - `CreationTime`: 创建时间

## 配置项

无特定配置项，使用默认的数据库配置。

## 注意事项

- **唯一性约束**：同一绑定项与策略的关联关系必须唯一
- **级联删除**：删除关联关系时会同步清理策略状态，需注意数据一致性
- **权限要求**：所有接口需要授权访问 `[Authorize]`
- **操作日志**：创建和更新操作会记录操作日志 `[OperLog]`

## API 路径

- `POST /api/app/binding-item-strategy-rels` - 创建关联
- `PUT /api/app/binding-item-strategy-rels/{id}` - 更新关联
- `DELETE /api/app/binding-item-strategy-rels/{id}` - 删除关联
- `GET /api/app/binding-item-strategy-rels/{id}` - 获取单个关联
- `GET /api/app/binding-item-strategy-rels` - 获取关联列表

## 相关文档链接

- [[DataBindingItemService]] - 数据绑定项服务
- [[StrategyStateService]] - 策略状态服务
- [[数据绑定架构]] - 数据绑定整体架构说明
