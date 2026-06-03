# 数据绑定项服务 (DataBindingItemService)

## 概述
数据绑定项服务负责管理系统中的数据绑定配置，将监测点位与业务数据关联起来。

## 职责
- 创建数据绑定项
- 更新绑定配置
- 删除绑定项
- 查询绑定列表
- 执行数据绑定

## 主要接口

### 创建绑定项
```csharp
Task<DataBindingItemDto> CreateAsync(CreateDataBindingItemDto input)
```

**请求参数**：
- `SourcePointId`: 源点位ID
- `TargetProperty`: 目标属性
- `TransformType`: 转换类型
- `TransformConfig`: 转换配置

### 更新绑定项
```csharp
Task<DataBindingItemDto> UpdateAsync(Guid id, UpdateDataBindingItemDto input)
```

### 删除绑定项
```csharp
Task DeleteAsync(Guid id)
```

### 获取绑定列表
```csharp
Task<PagedResultDto<DataBindingItemDto>> GetListAsync(GetDataBindingItemListDto input)
```

### 执行绑定
```csharp
Task ExecuteAsync(Guid bindingItemId)
```

## 绑定类型

| 类型 | 说明 |
|-----|------|
| Direct | 直接映射 |
| Transform | 数据转换 |
| Aggregate | 数据聚合 |
| Calculate | 计算字段 |

## 数据处理流程

```
源点位数据 → 绑定规则 → 数据转换 → 目标属性 → 保存
```

## 依赖服务

- [[MonitoredPointService]] - 监测点位服务
- [[DataStrategyService]] - 数据策略服务
- [[PointDataService]] - 点位数据服务

## 相关实体

- [[DataBindingItem]] - 数据绑定项实体
- [[MonitoredPoint]] - 监测点位
- [[DataStrategy]] - 数据策略

## API 路径

- `POST /api/app/data-binding-items` - 创建绑定项
- `PUT /api/app/data-binding-items/{id}` - 更新绑定项
- `DELETE /api/app/data-binding-items/{id}` - 删除绑定项
- `GET /api/app/data-binding-items` - 获取绑定列表
