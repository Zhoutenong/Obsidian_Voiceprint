# 显示组件服务 (DisplayComponentService)

## 概述
显示组件服务负责管理前端展示组件的配置，定义各种UI组件的标识、名称、参数和描述，为数据绑定类型提供可复用的展示组件库。

## 职责
- 管理显示组件的创建、更新、删除
- 维护组件标识（Key）的唯一性
- 定义组件的参数配置（JSON格式）
- 提供组件的查询和列表功能

## 主要接口

### 创建显示组件
```csharp
Task<DisplayComponentDto> CreateAsync(DisplayComponentCreateUpdateDto input)
```

**功能说明**：创建新的显示组件

**请求参数**：
- `Key`: 组件标识（必须唯一，用于前端组件识别）
- `Name`: 组件显示名称
- `Description`: 组件描述
- `Parameters`: 组件参数（JSON格式）

**业务规则**：
- 组件标识（Key）必须全局唯一
- 组件名称必须全局唯一

**异常**：
- `UserFriendlyException`: 组件标识已存在
- `UserFriendlyException`: 组件名称已存在

### 更新显示组件
```csharp
Task<DisplayComponentDto> UpdateAsync(Guid id, DisplayComponentCreateUpdateDto input)
```

**功能说明**：更新现有的显示组件

**请求参数**：
- `id`: 组件ID
- `Key`: 组件标识（必须唯一）
- `Name`: 组件显示名称
- `Description`: 组件描述
- `Parameters`: 组件参数（JSON格式）

**业务规则**：
- 组件标识（Key）必须全局唯一（排除自身）
- 组件名称必须全局唯一（排除自身）

**异常**：
- `UserFriendlyException`: 组件标识已存在
- `UserFriendlyException`: 组件名称已存在

### 删除显示组件
```csharp
Task DeleteAsync(Guid id)
```

**功能说明**：删除指定的显示组件

**请求参数**：
- `id`: 组件ID

**调用链**：
```
Controller → DisplayComponentService.DeleteAsync → SqlSugarRepository.DeleteAsync
```

### 获取组件列表
```csharp
Task<PagedResultDto<DisplayComponentDto>> GetListAsync(DisplayComponentGetListInputDto input)
```

**功能说明**：分页查询显示组件列表

**请求参数**：
- `Keyword`: 搜索关键字（支持按Key、名称、描述搜索）
- `SkipCount`: 跳过记录数
- `MaxResultCount`: 最大返回记录数

**返回数据**：
- `Items`: 组件列表
- `TotalCount`: 总记录数

**排序**：按组件标识（Key）升序

**搜索范围**：Key、Name、Description

### 获取所有组件
```csharp
Task<List<DisplayComponentDto>> GetAllListAsync(DisplayComponentGetListInputDto input)
```

**功能说明**：获取所有匹配的显示组件（不分页）

**请求参数**：
- `Keyword`: 搜索关键字（支持按Key、名称、描述搜索）

**返回数据**：完整的组件列表

## 数据实体

### DisplayComponentEntity
```csharp
public class DisplayComponentEntity : Entity<Guid>
{
    public Guid Id { get; set; }
    public string Key { get; set; }
    public string Name { get; set; }
    public string Description { get; set; }
    public string Parameters { get; set; }
}
```

**数据库表**：`ast_display_component`

## 依赖服务

- [[DataBindingTypeService]] - 数据绑定类型服务
- [[DataBindingItemService]] - 数据绑定项服务

## 业务流程

### 创建显示组件流程
```
1. 验证组件标识（Key）唯一性
2. 验证组件名称唯一性
3. 创建显示组件记录
4. 返回创建的组件信息
```

### 更新显示组件流程
```
1. 验证组件标识（Key）唯一性（排除自身）
2. 验证组件名称唯一性（排除自身）
3. 更新显示组件记录
4. 返回更新后的组件信息
```

## 组件参数格式

组件参数使用 JSON 格式存储，示例如下：

```json
{
  "type": "chart",
  "chartType": "line",
  "options": {
    "responsive": true,
    "maintainAspectRatio": false,
    "scales": {
      "y": {
        "beginAtZero": true
      }
    }
  }
}
```

## 常见组件类型

| Key | 说明 | 典型参数 |
|-----|------|---------|
| `text-display` | 文本显示 | `{ "fontSize": 14, "color": "#333" }` |
| `chart-line` | 折线图 | `{ "chartType": "line", "showPoints": true }` |
| `chart-bar` | 柱状图 | `{ "chartType": "bar", "stacked": false }` |
| `gauge` | 仪表盘 | `{ "min": 0, "max": 100, "unit": "%" }` |
| `status-indicator` | 状态指示器 | `{ "states": ["正常", "告警", "故障"] }` |
| `table` | 数据表格 | `{ "columns": [...], "pagination": true }` |

## 与其他模块的关系

```
DisplayComponent (显示组件)
    ↑ 1:N
DataBindingType (绑定类型)
    ↓ 1:N
DataBindingItem (绑定项)
```

## API 路径

- `POST /api/app/display-components` - 创建显示组件
- `PUT /api/app/display-components/{id}` - 更新显示组件
- `DELETE /api/app/display-components/{id}` - 删除显示组件
- `GET /api/app/display-components` - 获取组件列表（分页）
- `GET /api/app/display-components/all` - 获取所有组件

## 权限要求

- 所有接口都需要 `[Authorize]` 认证
- 创建操作记录操作日志：`OperLog("创建显示组件", OperEnum.Insert)`
- 更新操作记录操作日志：`OperLog("更新显示组件", OperEnum.Update)`
- 删除操作记录操作日志：`OperLog("删除显示组件", OperEnum.Delete)`

## 相关文档

- [[DataBindingTypeService]] - 数据绑定类型服务
- [[DataBindingItemService]] - 数据绑定项服务
- [[DataStrategyService]] - 数据策略服务
