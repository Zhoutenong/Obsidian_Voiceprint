# AstPointEntity — 传感器点位实体

## 基本信息

- **实体名称**：`AstPointEntity`
- **数据库表名**：`ast_point`
- **模块位置**：`module/ast-intellisub/Ast.IntelliSub.Domain/Entities/Gateway/`
- **继承关系**：`Entity<Guid>` (ABP实体基类)
- **命名空间**：`Ast.IntelliSub.Domain.Entities`

## 实体概述

AstPointEntity 是传感器点位实体，表示网关设备中某个传感器的特定属性点。它记录了从边缘设备采集的各种传感器数据点位信息，是数据采集和存储的核心元数据。

## 核心职责

1. **点位元数据** - 存储传感器点位的基本信息
2. **数据类型定义** - 定义点位值的数据类型（int、float、string、json、enum）
3. **单位管理** - 存储点位单位或枚举值描述
4. **源点位映射** - 映射到源系统的点位ID（如104点号）
5. **算法关联** - 关联AI识别算法

## 字段说明

### 主键

| 字段名 | 数据库列名 | 类型 | 说明 |
|--------|-----------|------|------|
| `Id` | `id` | `Guid` | 主键ID（继承自Entity<Guid>） |

### 核心字段

| 字段名 | 数据库列名 | 类型 | 说明 |
|--------|-----------|------|------|
| `DeviceId` | `device_id` | `string` | 设备ID |
| `SensorKey` | `sensor_key` | `string` | 传感器key |
| `Property` | `property` | `string` | 属性名称 |
| `Name` | `name` | `string` | 显示名称 |
| `Unit` | `unit` | `text` | 单位或枚举描述 |
| `ValueType` | `value_type` | `DataValueTypeEnum` | 值类型（0:int,1:float,2:string,3:json,4:enum） |
| `Type` | `type` | `AstPointTypeEnum` | 点位类型 |
| `SrcPointId` | `src_point_id` | `string` | 源点位ID（如104点号） |
| `CreationTime` | `creation_time` | `DateTime` | 创建时间 |
| `AlgorithmId` | `algorithm_id` | `Guid?` | 算法ID（可为空） |
| `RecArea` | `rec_area` | `text` | 图片识别区域（可为空） |

## 字段详解

### Unit - 单位字段

**格式说明**：
- **普通数值类型**：存储单位字符串（多个单位用 `|` 分隔）
  - 示例：`"℃"`, `"V|A"`, `"kW·h"`
- **枚举类型**：存储枚举值描述，格式为 `"值:描述,值:描述"`
  - 示例：`"0:灭,1:亮"`
  - 示例：`"0:关闭,1:开启,2:故障"`

** ValueType - 值类型枚举**

```csharp
public enum DataValueTypeEnum
{
    Int = 0,      // 整数
    Float = 1,    // 浮点数
    String = 2,   // 字符串
    Json = 3,     // JSON对象
    Enum = 4      // 枚举
}
```

**使用场景**：
- `Int` - 开关量、计数器
- `Float` - 温度、电压、电流等模拟量
- `String` - 设备状态文本
- `Json` - 复杂结构化数据
- `Enum` - 离散状态值（开/关、正常/故障）

### Type - 点位类型

```csharp
public enum AstPointTypeEnum
{
    // 具体类型定义
}
```

### RecArea - 识别区域

**格式**：JSON格式的区域坐标
- 用于AI识别时的图片裁剪区域
- 格式：`{"x":0,"y":0,"width":100,"height":100}`

### SrcPointId - 源点位ID

**用途**：
- 存储原始协议的点位标识
- IEC104：点号（如 `"1001"`）
- IEC61850：对象引用（如 `"LD0/LLN0.St.Val"`）
- Modbus：寄存器地址

## 点位标识

点位的完整标识通常由以下三部分组成：

```
{DeviceId}|{SensorKey}|{Property}
```

**示例**：
- `"gateway001|temp_01|value"` - gateway001设备的temp_01传感器的值属性
- `"gateway002|switch_01|state"` - gateway002设备的switch_01传感器的状态属性

## 数据类型示例

### 整数类型 (Int)
```json
{
  "device_id": "gw001",
  "sensor_key": "counter_01",
  "property": "value",
  "name": "计数器",
  "unit": "次",
  "value_type": 0,
  "value": 12345
}
```

### 浮点类型 (Float)
```json
{
  "device_id": "gw001",
  "sensor_key": "temp_01",
  "property": "value",
  "name": "温度",
  "unit": "℃",
  "value_type": 1,
  "value": 25.5
}
```

### 枚举类型 (Enum)
```json
{
  "device_id": "gw001",
  "sensor_key": "light_01",
  "property": "state",
  "name": "照明状态",
  "unit": "0:关,1:开",
  "value_type": 4,
  "value": 1
}
```

## 算法关联

当 `AlgorithmId` 不为空时：
- 该点位与AI识别算法关联
- `RecArea` 定义识别区域
- 用于图像识别类的点位（如仪表读数、状态识别）

## 相关实体

### DeviceEntity - 设备实体
- `DeviceId` 关联到设备实体
- 表示点位所属的设备

### SensorEntity - 传感器实体
- `SensorKey` 关联到传感器实体
- 表示点位所属的传感器

### AlgorithmEntity - 算法实体
- `AlgorithmId` 关联到算法实体
- 用于AI识别点位

## 相关服务

### PointDataService - 点位数据服务
- 查询点位最新数据
- 查询点位历史数据
- 按告警状态过滤点位

### AstPointDataService - 传感器数据管理服务
- 点位数据采集
- 时序数据存储

### Iec61850MappingManager - IEC61850映射管理器
- 使用 `DeviceId|SensorKey|Property` 格式建立映射
- PointKey格式：`{device_id}|{sensor_key}|{property}|{alarm_category_name}`

## 数据采集流程

```
1. 边缘设备采集数据
   │
2. MQTT上报到网关
   │
3. 根据DeviceId+SensorKey+Property查找AstPointEntity
   │
4. 根据ValueType解析数据
   │
5. 存储到时序数据库
   │
6. 可选：触发告警评估
```

## 使用场景

### 1. 温度监测点位
```csharp
var tempPoint = new AstPointEntity
{
    Id = Guid.NewGuid(),
    DeviceId = "gateway_001",
    SensorKey = "temp_sensor_01",
    Property = "value",
    Name = "1号变压器温度",
    Unit = "℃",
    ValueType = DataValueTypeEnum.Float,
    Type = AstPointTypeEnum.Analog,
    SrcPointId = "0101"
};
```

### 2. 开关状态点位
```csharp
var switchPoint = new AstPointEntity
{
    Id = Guid.NewGuid(),
    DeviceId = "gateway_002",
    SensorKey = "switch_01",
    Property = "state",
    Name = "照明开关状态",
    Unit = "0:关,1:开",
    ValueType = DataValueTypeEnum.Enum,
    Type = AstPointTypeEnum.Digital,
    SrcPointId = "0201"
};
```

### 3. AI识别点位
```csharp
var meterPoint = new AstPointEntity
{
    Id = Guid.NewGuid(),
    DeviceId = "camera_001",
    SensorKey = "meter_01",
    Property = "reading",
    Name = "电压表读数",
    Unit = "V",
    ValueType = DataValueTypeEnum.Float,
    Type = AstPointTypeEnum.AI,
    SrcPointId = "rec_001",
    AlgorithmId = algorithmGuid,
    RecArea = "{\"x\":100,\"y\":100,\"width\":200,\"height\":100}"
};
```

## 设计特点

1. **灵活的数据类型** - 支持5种数据类型，覆盖各种传感器
2. **单位双重用途** - 普通类型存单位，枚举类型存描述
3. **源点位映射** - 保留原始协议的点位标识
4. **算法集成** - 支持AI识别点位
5. **三段式标识** - DeviceId|SensorKey|Property 唯一标识

## 注意事项

1. **点位唯一性** - DeviceId + SensorKey + Property 组合应该唯一
2. **单位格式** - 多个单位用 `|` 分隔，枚举类型用 `:` 分隔值和描述
3. **值类型匹配** - 存储的数据必须与ValueType声明的类型一致
4. **RecArea格式** - 必须是有效的JSON格式
5. **源点位ID** - 对于需要从PLC/RTU读取的点位，SrcPointId必须正确配置

## 索引建议

```sql
-- 复合索引：用于快速查找点位
CREATE INDEX idx_point_key ON ast_point(device_id, sensor_key, property);

-- 单列索引：用于按设备查询
CREATE INDEX idx_point_device ON ast_point(device_id);

-- 算法关联索引
CREATE INDEX idx_point_algorithm ON ast_point(algorithm_id) WHERE algorithm_id IS NOT NULL;
```

## 数据采集示例

### MQTT上报数据格式
```json
{
  "device_id": "gateway_001",
  "sensor_key": "temp_01",
  "property": "value",
  "value": 25.5,
  "timestamp": "2026-06-04T10:30:00Z"
}
```

### 查询点位信息
```csharp
// 根据三段式Key查询
var point = await _pointRepository.GetFirstAsync(p =>
    p.DeviceId == "gateway_001" &&
    p.SensorKey == "temp_01" &&
    p.Property == "value");
```

---

> **最后更新**：2026-06-04  
> **源码位置**：`module/ast-intellisub/Ast.IntelliSub.Domain/Entities/Gateway/AstPointEntity.cs`  
> **数据库表**：`ast_point`
