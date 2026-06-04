# DataBindingTypeAggregateRoot

数据绑定类型聚合根，定义数据绑定的类型及其显示组件配置。

## 表名与主键

- **表名**: `ast_data_binding_type`
- **主键**: `Id` (Guid)

## 字段列表

| 字段名 | 类型 | 说明 | 外键 |
|--------|------|------|------|
| `Id` | Guid | 主键ID | - |
| `name` | string | 显示名称 | - |
| `display_component_id` | Guid | 显示组件ID | DisplayComponentEntity |
| `description` | string | 描述 | - |
| `CreationTime` | DateTime | 创建时间 | - |
| `CreatorId` | Guid? | 创建者ID | - |
| `LastModificationTime` | DateTime? | 最后修改时间 | - |
| `LastModifierId` | Guid? | 最后修改者ID | - |

## 关联实体（ER关系）

### 关联从当前实体

- **DisplayComponentEntity** (N:1)
  - 通过 `display_component_id` 关联
  - 导航属性：`DisplayComponent`
  - 定义该绑定类型使用的UI组件

### 关联到当前实体

- **DataBindingItemEntity** (1:N)
  - 通过 `data_binding_type_id` 关联
  - 一个绑定类型包含多个绑定项

## 被哪些服务读写

### 写入服务

- **DataBindingTypeAppService** (推断)
  - 创建/更新/删除绑定类型

### 读取服务

- **DataBindingTypeAppService** (推断)
  - 获取绑定类型列表
  - 获取绑定类型详情
- **SubstationDataSeed**
  - 数据种子初始化时读取预设配置

## 业务规则约束

1. **命名唯一性**
   - `name` 字段应该唯一
   - 用于标识不同的监测类型（如"温度监测"、"湿度监测"）

2. **显示组件关联**
   - 每个绑定类型必须关联一个显示组件
   - 显示组件定义了该类型数据在前端的展示方式

3. **审计字段**
   - 继承自 `IAuditedObject`
   - 自动记录创建和修改信息

4. **绑定项关系**
   - 删除绑定类型前需要检查是否有关联的绑定项
   - 建议使用软删除或级联删除

## 数据示例

典型的绑定类型配置：

```csharp
// 温度监测类型
{
  "Id": "guid-1",
  "Name": "进线温度",
  "DisplayComponentId": "component-guid",
  "Description": "用于显示进线温度数据的绑定类型",
  "CreationTime": "2024-01-01T00:00:00",
  "CreatorId": "admin-user-guid"
}

// 声纹监测类型
{
  "Name": "声纹声音",
  "DisplayComponentId": "audio-player-component",
  "Description": "声纹分析数据展示"
}

// 振动监测类型
{
  "Name": "振动监测",
  "DisplayComponentId": "chart-component",
  "Description": "设备振动数据实时监测"
}
```

## 常见绑定类型

根据系统配置，常见的绑定类型包括：

| 绑定类型 | 说明 | 典型绑定项 |
|----------|------|-----------|
| 分闸监测 | 断路器分闸状态监测 | A相分闸、B相分闸、C相分闸 |
| 合闸监测 | 断路器合闸状态监测 | A相合闸、B相合闸、C相合闸 |
| 进线温度 | 进线端温度监测 | A相温度、B相温度、C相温度 |
| 出线温度 | 出线端温度监测 | A相温度、B相温度、C相温度 |
| 暂态地局放 | 暂态地电压局部放电监测 | PD幅值、PD脉冲数 |
| 超声波局放 | 超声波局部放电监测 | 超声波幅值、超声波频谱 |
| 振动监测 | 设备振动状态监测 | 振动加速度、振动速度 |
| 柜内温湿度 | 柜内环境监测 | 温度、湿度 |
| 高频电流局放 | 高频电流局部放电监测 | HFCT幅值、放电量 |
| 声纹声音 | 声纹监测 | 音频数据、声纹特征 |
| 铁芯接地电流 | 铁芯接地电流监测 | 接地电流值 |
| 特高频局放 | 特高频局部放电监测 | UHF信号、放电强度 |

## 索引建议

- `name` - 用于按名称搜索（建议唯一索引）
- `display_component_id` - 用于按显示组件查询

## 注意事项

1. **类型定义**
   - 绑定类型是数据绑定的核心分类
   - 决定了数据的展示方式和处理逻辑

2. **显示组件选择**
   - 不同类型的监测数据使用不同的显示组件
   - 例如：温度用仪表盘、声纹用音频播放器、振动用波形图

3. **删除保护**
   - 删除绑定类型会影响所有关联的绑定项
   - 建议实现删除前的依赖检查

4. **扩展性**
   - 新增监测类型时需要创建对应的绑定类型
   - 绑定类型的命名应该规范和统一
