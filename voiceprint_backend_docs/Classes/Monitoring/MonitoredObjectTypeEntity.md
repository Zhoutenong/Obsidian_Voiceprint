# MonitoredObjectTypeEntity

被监测对象类型实体，定义监测对象的分类和展示配置。

## 表名与主键

- **表名**: `ast_monitored_object_type`
- **主键**: `Id` (Guid)

## 字段列表

| 字段名 | 类型 | 说明 | 外键 |
|--------|------|------|------|
| `Id` | Guid | 主键ID | - |
| `key` | string | 展示监测对象UI形状的key标识 | - |
| `name` | string | 显示名称 | - |
| `category` | MonitoredObjectCategoryEnum | 分类（枚举） | - |
| `description` | string? | 描述 | - |
| `icon` | string? | 图标 | - |

## 关联实体（ER关系）

### 关联到当前实体

- **MonitoredObjectAggregateRoot** (1:N)
  - 通过 `monitored_object_type_id` 关联
  - 一个类型可以被多个监测对象使用

## 被哪些服务读写

### 写入服务

- **SubstationDataSeed** (数据种子初始化)
  - `InitializeMonitoredObjectTypes()` - 初始化预设对象类型

- **MonitoredObjectTypeAppService** (推断)
  - 创建/更新/删除对象类型

### 读取服务

- **MonitoredObjectTypeAppService** (推断)
  - 获取对象类型列表
  - 获取对象类型详情
  - 按分类查询类型

- **前端渲染服务** (推断)
  - 根据类型配置渲染对应的UI形状
  - 显示对象的图标和样式

## 业务规则约束

1. **Key 唯一性**
   - `key` 字段必须唯一
   - 对应前端UI中的形状标识

2. **命名约束**
   - `name` 应该清晰描述对象类型
   - 建议使用标准的设备命名

3. **分类约束**
   - `category` 枚举值决定对象的层级位置
   - 不能随意修改已有类型的分类

4. **图标格式**
   - `icon` 应该是有效的图标标识
   - 可以是图标文件名或图标类名

## 对象类型分类

### Group（分组）= 0
设备组类型，用于组织和管理设备：

```csharp
{
  "key": "switchgear_room",
  "name": "开关柜室",
  "category": 0,
  "description": "开关设备室",
  "icon": "room-icon"
}
```

常见的分组类型：
- 开关柜室
- 变压器室
- 控制室
- 继电保护室
- 电容器室
- 电抗器室

### Device（设备）= 1
设备类型，表示具体的监测设备：

```csharp
{
  "key": "switchgear",
  "name": "开关柜",
  "category": 1,
  "description": "高压开关柜设备",
  "icon": "switchgear-icon"
}
```

常见的设备类型：
- 开关柜
- 变压器
- 接地变
- 互感器
- 断路器
- 隔离开关
- 电容器
- 电抗器

### SubDevice（子设备）= 2
子设备类型，表示设备的组成部分：

```csharp
{
  "key": "breaker",
  "name": "断路器",
  "category": 2,
  "description": "断路器子设备",
  "icon": "breaker-icon"
}
```

常见的子设备类型：
- A相断路器
- B相断路器
- C相断路器
- A相传感器
- B相传感器
- C相传感器
- 温度传感器
- 湿度传感器

## 数据示例

典型的对象类型配置：

```csharp
// 开关柜室
{
  "Id": "guid-1",
  "Key": "switchgear_room",
  "Name": "开关柜室",
  "Category": 0,  // Group
  "Description": "开关设备室",
  "Icon": "icon-room"
}

// 变压器
{
  "Key": "transformer",
  "Name": "变压器",
  "Category": 1,  // Device
  "Description": "电力变压器",
  "Icon": "icon-transformer"
}

// 断路器
{
  "Key": "breaker",
  "Name": "断路器",
  "Category": 2,  // SubDevice
  "Description": "高压断路器",
  "Icon": "icon-breaker"
}
```

## 预设对象类型

### 站点分组类型

| Key | Name | Category | Description |
|-----|------|----------|-------------|
| `switchgear_room` | 开关柜室 | 0 | 开关设备室 |
| `transformer_room` | 变压器室 | 0 | 变压器设备室 |
| `control_room` | 控制室 | 0 | 控制设备室 |
| `relayprotection_room` | 继电保护室 | 0 | 继电保护设备室 |
| `capacitor_room` | 电容器室 | 0 | 电容器设备室 |
| `reactor_room` | 电抗器室 | 0 | 电抗器设备室 |

### 设备类型

| Key | Name | Category | Description |
|-----|------|----------|-------------|
| `switchgear` | 开关柜 | 1 | 高压开关柜 |
| `transformer` | 变压器 | 1 | 电力变压器 |
| `grounding_transformer` | 接地变 | 1 | 接地变压器 |
| `current_transformer` | 电流互感器 | 1 | 电流互感器 |
| `voltage_transformer` | 电压互感器 | 1 | 电压互感器 |
| `disconnector` | 隔离开关 | 1 | 隔离开关 |
| `capacitor` | 电容器 | 1 | 电力电容器 |
| `reactor` | 电抗器 | 1 | 电力电抗器 |

### 子设备类型

| Key | Name | Category | Description |
|-----|------|----------|-------------|
| `breaker` | 断路器 | 2 | 高压断路器 |
| `sensor_temp` | 温度传感器 | 2 | 温度监测传感器 |
| `sensor_humidity` | 湿度传感器 | 2 | 湿度监测传感器 |
| `sensor_vibration` | 振动传感器 | 2 | 振动监测传感器 |
| `sensor_pd` | 局放传感器 | 2 | 局部放电传感器 |

## 类型与层级结构

对象类型的分类决定了对象的层级结构：

```
Group (category = 0) - 顶级
    ↓
Device (category = 1) - 中级
    ↓
SubDevice (category = 2) - 底级
```

规则：
1. Group 的 `parent_id` 必须为 null
2. Device 的 `parent_id` 必须指向 Group
3. SubDevice 的 `parent_id` 必须指向 Device

## UI渲染

### Key 的作用
`key` 字段用于前端渲染对应的UI形状：

```typescript
// 前端组件映射
const componentMap = {
  'switchgear_room': RoomComponent,
  'transformer': TransformerComponent,
  'breaker': BreakerComponent,
  'sensor_temp': SensorComponent
};

// 根据类型渲染
const Component = componentMap[objectType.key];
return <Component data={monitoredObject} />;
```

### Icon 的作用
`icon` 字段用于显示对象的图标：

- 可以是图标文件名（如 `icon-transformer.svg`）
- 可以是图标库的图标类名（如 `fa fa-transformer`）
- 可以是 Emoji（如 `🔌`）

## 索引建议

- `key` - 用于按key查询（建议唯一索引）
- `category` - 用于按分类查询
- `name` - 用于按名称搜索

## 注意事项

1. **类型定义规范**
   - `key` 应该使用英文、小写、下划线分隔
   - `name` 使用中文描述

2. **分类约束**
   - 不要随意修改已有类型的分类
   - 修改分类可能影响对象层级结构

3. **前端依赖**
   - `key` 必须与前端组件映射一致
   - 新增类型需要前端实现对应组件

4. **图标管理**
   - 图标文件应该统一管理
   - 建议使用SVG图标以支持缩放

5. **扩展性**
   - 新增对象类型需要：
     1. 创建类型记录
     2. 在前端实现对应的UI组件
     3. 更新组件映射表

6. **数据迁移**
   - 类型定义的修改需要考虑数据迁移
   - 删除类型前检查是否有关联的监测对象
