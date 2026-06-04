# DataBindingItemEntity

数据绑定项实体，定义监测对象的数据属性配置。

## 表名与主键

- **表名**: `ast_data_binding_item`
- **主键**: `Id` (Guid)

## 字段列表

| 字段名 | 类型 | 说明 | 外键 |
|--------|------|------|------|
| `Id` | Guid | 主键ID | - |
| `data_binding_type_id` | Guid | 绑定类型ID | DataBindingTypeAggregateRoot |
| `property` | string | 属性名称（对应对象属性） | - |
| `name` | string | 属性显示全称 | - |
| `short_name` | string? | 属性显示简称 | - |
| `data_value_type` | DataValueTypeEnum | 数据值类型（枚举） | - |
| `unit` | text | 单位（多个单位用\|分隔） | - |
| `is_display` | bool | 是否在界面上展示 | - |
| `order_num` | int? | 排序号 | - |

## 关联实体（ER关系）

### 关联从当前实体

- **DataBindingTypeAggregateRoot** (N:1)
  - 通过 `data_binding_type_id` 关联
  - 导航属性：`BindingType`

### 关联到当前实体

- **BindingItemStrategyRelEntity** (1:N)
  - 通过 `data_binding_item_id` 关联
  - 表示绑定项与策略的关系

- **PointBindingRelEntity** (1:N)
  - 通过 `data_binding_item_id` 关联
  - 表示点位与绑定项的绑定关系

## 被哪些服务读写

### 写入服务

- **SubstationDataSeed** (数据种子初始化)
  - `InitializeDataBindingItems()` - 初始化预设绑定项配置

### 读取服务

- **DataBindingAppService** (推断)
  - 用于前端展示绑定项列表
  - 用于数据绑定配置界面

## 业务规则约束

1. **绑定项唯一性**
   - 同一绑定类型下的属性名称 (`property`) 应该唯一
   - 同一绑定类型下的显示名称 (`name`) 应该唯一

2. **数据值类型约束**
   - `data_value_type` 枚举值决定了数据的格式和验证规则
   - 常见类型：数值、字符串、布尔值、日期时间等

3. **单位格式**
   - 多个单位使用 `|` 分隔符
   - 示例：`"℃|%|°C"` 表示温度单位

4. **显示控制**
   - `is_display = false` 的绑定项不会在前端界面显示
   - `order_num` 决定了在前端的显示顺序

5. **数据绑定流程**
   - 绑定项 → 绑定类型 → 显示组件
   - 每个绑定项必须关联到一个绑定类型

## 数据示例

典型的绑定项配置：

```csharp
// 温度监测绑定项
{
  "Id": "guid-1",
  "DataBindingTypeId": "binding-type-guid",
  "Property": "Temperature",
  "Name": "A相温度",
  "ShortName": "A相温",
  "DataValueType": 0,  // 数值类型
  "Unit": "℃",
  "IsDisplay": true,
  "OrderNum": 1
}

// 湿度监测绑定项
{
  "Property": "Humidity",
  "Name": "柜内湿度",
  "ShortName": "湿度",
  "DataValueType": 0,
  "Unit": "%",
  "IsDisplay": true,
  "OrderNum": 2
}
```

## 索引建议

- `data_binding_type_id` - 用于按绑定类型查询
- `name` - 用于按名称搜索
- `(data_binding_type_id, property)` - 联合唯一索引

## 注意事项

1. **属性命名规范**
   - `property` 字段应使用驼峰命名
   - 通常对应后端对象的属性名

2. **显示名称**
   - `name` 用于完整显示
   - `short_name` 用于空间受限的场景（如表格列头）

3. **单位处理**
   - 前端需要解析 `|` 分隔的单位列表
   - 应提供单位切换功能

4. **顺序控制**
   - `order_num` 值越小，显示顺序越靠前
   - `null` 值的排序行为取决于前端实现
