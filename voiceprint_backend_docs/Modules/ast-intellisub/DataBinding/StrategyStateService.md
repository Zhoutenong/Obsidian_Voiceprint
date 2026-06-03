# 策略状态服务 (StrategyStateService)

## 概述
策略状态服务负责管理数据绑定策略的执行状态，记录策略在监测点位上的应用结果和历史状态。

## 职责
- 记录策略执行状态
- 查询策略状态历史
- 管理策略与点位的关联状态
- 批量清理策略状态
- 提供策略执行结果追踪

## 主要接口

### 获取策略状态列表
```csharp
Task<PagedResultDto<StrategyStateDto>> GetListAsync(StrategyStateGetListInputDto input)
```

**请求参数**：
- `BindingItemStrategyRelId`: 绑定项策略关系ID（可选）
- `PointBindingRelId`: 点位绑定关系ID（可选）
- `SkipCount`: 跳过记录数
- `MaxResultCount`: 最大返回数量

**响应数据**：
- 策略状态列表（按创建时间降序）
- 总记录数

### 获取单个策略状态
```csharp
Task<StrategyStateDto> GetAsync(Guid id)
```

### 创建策略状态
```csharp
Task<StrategyStateDto> CreateAsync(CreateStrategyStateDto input)
```

### 更新策略状态
```csharp
Task<StrategyStateDto> UpdateAsync(Guid id, UpdateStrategyStateDto input)
```

### 删除策略状态
```csharp
Task DeleteAsync(Guid id)
```

### 批量删除策略状态（按点位绑定关系）
```csharp
Task DeleteManyAsync(List<Guid> pointBindingRelIds)
```

### 清理策略状态

#### 按点位绑定关系清理
```csharp
Task ClearStrategyStatesByPointBindingRelIdsAsync(List<Guid> pointBindingRelIds)
```

#### 按绑定关系清理
```csharp
Task ClearStrategyStatesByBindingRelIdsAsync(List<Guid> bindingRelIds)
```

#### 清理所有策略状态
```csharp
Task ClearAllStrategyStatesAsync()
```

## 数据处理流程

```
策略执行 → 状态记录 → 状态更新 → 历史查询 → 状态清理
```

### 状态更新流程
1. 策略执行时创建状态记录
2. 根据执行结果更新状态
3. 记录执行时间和结果数据
4. 支持按条件批量清理

## 依赖服务

- [[DataBindingItemService]] - 数据绑定项服务
- [[BindingItemStrategyRelService]] - 绑定项策略关联服务
- [[PointBindingRelService]] - 点位绑定关系服务

## 相关实体

- **StrategyStateEntity** - 策略状态实体
  - `Id`: 状态ID
  - `BindingItemStrategyRelId`: 绑定项策略关系ID
  - `PointBindingRelId`: 点位绑定关系ID
  - `Status`: 状态值
  - `ExecutionTime`: 执行时间
  - `CreationTime`: 创建时间

## 配置项

无特定配置项，使用默认的数据库配置。

## 注意事项

- **数据量管理**：策略状态记录会持续增长，建议定期清理历史数据
- **查询性能**：按点位或策略关系查询时已建立索引
- **批量操作**：批量删除操作使用事务保证数据一致性
- **权限要求**：所有接口需要授权访问 `[Authorize]`

## 相关文档链接

- [[DataBindingItemService]] - 数据绑定项服务
- [[BindingItemStrategyRelService]] - 绑定项策略关联服务
- [[数据绑定架构]] - 数据绑定整体架构说明
