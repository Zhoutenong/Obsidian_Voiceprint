# 数据策略服务 (DataStrategyService)

## 概述
数据策略服务定义数据的处理策略，包括数据采集频率、存储策略、告警策略等。

## 职责
- 定义数据策略
- 关联绑定项与策略
- 执行数据策略
- 管理策略状态

## 主要接口

### 创建策略
```csharp
Task<DataStrategyDto> CreateAsync(CreateDataStrategyDto input)
```

**请求参数**：
- `Name`: 策略名称
- `StrategyType`: 策略类型
- `Config`: 策略配置
- `Priority`: 优先级

### 关联绑定项
```csharp
Task AssociateAsync(Guid strategyId, Guid bindingItemId)
```

### 执行策略
```csharp
Task ExecuteAsync(Guid strategyId, dynamic data)
```

## 策略类型

| 类型 | 说明 |
|-----|------|
| Collection | 采集策略 |
| Storage | 存储策略 |
| Alarm | 告警策略 |
| Transform | 转换策略 |

## 依赖服务

- [[DataBindingItemService]] - 数据绑定项服务
- [[StrategyStateService]] - 策略状态服务
- [[BindingItemStrategyRelService]] - 绑定项策略关联服务

## 相关实体

- [[DataStrategy]] - 数据策略实体
- [[DataBindingItem]] - 数据绑定项
- [[StrategyState]] - 策略状态

## API 路径

- `POST /api/app/data-strategies` - 创建策略
- `PUT /api/app/data-strategies/{id}` - 更新策略
- `DELETE /api/app/data-strategies/{id}` - 删除策略
- `GET /api/app/data-strategies` - 获取策略列表
