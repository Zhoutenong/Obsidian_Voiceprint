# 传感器数据 API 文档

## 概述
传感器数据模块提供点位数据的查询、上报和历史数据聚合等功能，支持实时数据查询、批量获取和历史趋势分析。

## 基础路径
```
/api/app/voiceprint/point-data
```

## 认证
所有接口都需要 JWT Token 认证：
```
Authorization: Bearer {token}
```

---

## 最新点位数据

### 获取最新点位数据

**接口**: `GET /point-data/latest`

**描述**: 根据点位ID列表获取每个点位的最新数据

**请求参数**:

| 参数 | 类型 | 说明 |
|-----|------|------|
| pointIds[] | string[] | 点位ID列表 |
| deviceId | string? | 设备ID（可选） |
| sensorKey | string? | 传感器键值（可选） |

**查询字符串示例**:
```
?pointIds[]=id1&pointIds[]=id2&deviceId=device123&sensorKey=temp01
```

**响应**:
```json
{
  "success": true,
  "data": [
    {
      "ts": "2026-06-03T10:30:00Z",
      "pointId": "point-001",
      "groupId": "group-001",
      "property": "temperature",
      "value": 25.5,
      "valueType": 1,
      "deviceId": "device-001",
      "sensorKey": "temp01",
      "alarmLevel": 0
    },
    {
      "ts": "2026-06-03T10:30:00Z",
      "pointId": "point-002",
      "groupId": "group-001",
      "property": "vibration",
      "value": 3.2,
      "valueType": 1,
      "deviceId": "device-001",
      "sensorKey": "vib01",
      "alarmLevel": 0
    }
  ]
}
```

**数据结构**:
```typescript
interface PointDataDto {
  ts: Date;                    // 事件时间戳
  pointId: string;             // 业务点位ID
  groupId: string;              // 分组ID
  property: string;             // 属性名称
  value: number | string | object;  // 属性值（根据ValueType返回不同类型）
  valueType: number;            // 值类型：0:int, 1:float, 2:string, 3:json, 4:enum
  deviceId: string;             // 设备ID
  sensorKey: string;            // 传感器键值
  alarmLevel: AlarmLevelEnum;   // 告警级别：0-正常, 1-预警, 2-告警
  extInfoObject?: object;       // 扩展信息对象
}

enum AlarmLevelEnum {
  Normal = 0,      // 正常
  Warning = 1,     // 预警
  Alarm = 2        // 告警
}
```

---

## 历史点位数据

### 获取历史点位数据（聚合查询）

**接口**: `GET /point-data/history`

**描述**: 获取指定时间范围内的历史数据，支持降采样聚合查询，提供时序数据的统计分析

**请求参数**:

| 参数 | 类型 | 说明 |
|-----|------|------|
| deviceId | string | 设备ID（必填） |
| sensorKey | string | 传感器键值（必填） |
| pointId | guid? | 点位ID（可选） |
| groupId | string? | 分组ID（可选） |
| property | string? | 属性名称（可选） |
| startTime | datetime | 查询开始时间（必填） |
| endTime | datetime | 查询结束时间（必填） |
| intervalInMinutes | int | 聚合时间间隔（1-10080分钟，必填） |
| skipCount | int | 跳过记录数（用于分页，默认0） |
| maxResultCount | int | 最大返回记录数（默认10，最大500） |

**查询字符串示例**:
```
?deviceId=device001&sensorKey=temp01&startTime=2026-06-01T00:00:00Z&endTime=2026-06-03T00:00:00Z&intervalInMinutes=60&skipCount=0&maxResultCount=48
```

**响应**:
```json
{
  "success": true,
  "data": {
    "totalCount": 48,
    "items": [
      {
        "timestamp": "2026-06-01T00:00:00Z",
        "avgValue": 24.5,
        "maxValue": 26.0,
        "minValue": 23.0
      },
      {
        "timestamp": "2026-06-01T01:00:00Z",
        "avgValue": 24.8,
        "maxValue": 26.5,
        "minValue": 23.2
      }
    ]
  }
}
```

**数据结构**:
```typescript
interface DataPointDto {
  timestamp: Date;      // 时间窗口起始时间
  avgValue?: number;   // 平均值
  maxValue?: number;   // 最大值
  minValue?: number;   // 最小值
}
```

**聚合说明**:
- 数据按指定时间间隔（intervalInMinutes）进行分组
- 每个时间窗口返回该时间段内的统计值
- 适用于大数据量的历史趋势分析，减少前端处理压力
- 时间间隔范围：1分钟 到 10080分钟（7天）

---

## 批量获取点位数据

### 批量获取点位数据列表

**接口**: `GET /point-data/batch`

**描述**: 按点位ID列表查询数据，支持分页和时间范围过滤

**请求参数**:

| 参数 | 类型 | 说明 |
|-----|------|------|
| pointIds[] | string[] | 点位ID列表 |
| deviceId | string? | 设备ID（可选） |
| sensorKey | string? | 传感器键值（可选） |
| startTime | datetime | 开始时间（必填） |
| endTime | datetime? | 结束时间（可选） |
| skipCount | int | 跳过记录数（默认0） |
| maxResultCount | int | 最大返回记录数（默认20） |

**查询字符串示例**:
```
?pointIds[]=p1&pointIds[]=p2&startTime=2026-06-01T00:00:00Z&endTime=2026-06-03T00:00:00Z&skipCount=0&maxResultCount=50
```

**响应**:
```json
{
  "success": true,
  "data": {
    "totalCount": 150,
    "items": [
      {
        "ts": "2026-06-03T10:00:00Z",
        "pointId": "p1",
        "groupId": "group-001",
        "property": "temperature",
        "value": 25.5,
        "valueType": 1,
        "deviceId": "device-001",
        "sensorKey": "temp01",
        "alarmLevel": 0
      }
    ]
  }
}
```

---

## 点位数据列表

### 获取点位数据列表（通用查询）

**接口**: `GET /point-data`

**描述**: 通用的点位数据查询接口，支持多维度过滤和分页

**请求参数**:

| 参数 | 类型 | 说明 |
|-----|------|------|
| deviceId | string? | 设备ID |
| sensorKey | string? | 传感器键值 |
| pointId | string? | 业务点位ID |
| groupId | string? | 分组ID |
| property | string? | 属性名称 |
| val | double? | 数值属性值过滤 |
| strVal | string? | 字符串属性值过滤 |
| startTime | datetime | 开始时间（必填） |
| endTime | datetime? | 结束时间 |
| isAsc | bool | 是否按时间升序（默认false-降序） |
| skipCount | int | 跳过记录数（默认0） |
| maxResultCount | int | 最大返回记录数（默认20） |
| sorting | string | 排序字段（如 "ts desc"） |

**查询字符串示例**:
```
?deviceId=device001&sensorKey=temp01&startTime=2026-06-01T00:00:00Z&endTime=2026-06-03T00:00:00Z&skipCount=0&maxResultCount=50
```

**响应**:
```json
{
  "success": true,
  "data": {
    "totalCount": 500,
    "items": [
      {
        "ts": "2026-06-03T10:00:00Z",
        "pointId": "point-001",
        "groupId": "group-001",
        "property": "temperature",
        "value": 25.5,
        "valueType": 1,
        "deviceId": "device-001",
        "sensorKey": "temp01",
        "alarmLevel": 0
      }
    ]
  }
}
```

---

## 上报点位数据

### 上报点位数据

**接口**: `POST /point-data`

**描述**: 新增点位数据记录，用于设备或边缘设备上报传感器数据

**请求体**:
```json
{
  "ts": "2026-06-03T10:30:00Z",
  "pointId": "point-001",
  "groupId": "group-001",
  "property": "temperature",
  "value": 25.5,
  "valueType": 1,
  "extInfo": "{\"unit\":\"Celsius\",\"location\":\"room1\"}",
  "deviceId": "device-001",
  "sensorKey": "temp01",
  "alarmLevel": 0
}
```

**请求参数说明**:

| 参数 | 类型 | 必填 | 说明 |
|-----|------|------|------|
| ts | datetime | 是 | 事件时间戳（主键） |
| pointId | string | 是 | 业务点位ID |
| groupId | string | 是 | 分组ID（标识同一批次数据） |
| property | string | 是 | 属性名称 |
| value | object | 是 | 属性值 |
| valueType | int? | 否 | 值类型：0:int, 1:float, 2:string, 3:json, 4:enum |
| extInfo | string? | 否 | 扩展信息（JSON格式） |
| deviceId | string | 是 | 设备ID |
| sensorKey | string | 是 | 传感器键值 |
| alarmLevel | int | 否 | 告警级别：0-正常, 1-预警, 2-告警（默认0） |

**响应**:
```json
{
  "success": true,
  "data": {
    "ts": "2026-06-03T10:30:00Z",
    "pointId": "point-001",
    "groupId": "group-001",
    "property": "temperature",
    "value": 25.5,
    "valueType": 1,
    "deviceId": "device-001",
    "sensorKey": "temp01",
    "alarmLevel": 0
  }
}
```

---

## 按告警状态过滤

### 根据告警状态过滤点位数据

**接口**: `GET /point-data/filtered-by-alarm-status`

**描述**: 查询正常或异常的点位数据，通过与告警记录关联判断数据状态

**请求参数**:

| 参数 | 类型 | 说明 |
|-----|------|------|
| isOpenPointId | guid | IsOpen属性对应的点位ID（必填） |
| isOpen | int | IsOpen值过滤（必填，通常为1） |
| isNormal | bool? | 是否查询正常数据（true-正常，false-异常） |
| deviceId | string? | 设备ID |
| sensorKey | string? | 传感器键值 |
| startTime | datetime? | 开始时间 |
| endTime | datetime? | 结束时间 |
| propertyPrefix | string? | 属性前缀过滤 |
| isAsc | bool | 是否按时间升序（默认false） |
| skipCount | int | 跳过记录数（默认0） |
| maxResultCount | int | 最大返回记录数（默认20） |

**查询字符串示例**:
```
?isOpenPointId=guid-here&isOpen=1&isNormal=true&startTime=2026-06-01T00:00:00Z&endTime=2026-06-03T00:00:00Z
```

**响应**:
```json
{
  "success": true,
  "data": {
    "totalCount": 50,
    "items": [
      {
        "ts": "2026-06-03T10:00:00Z",
        "pointId": "point-001",
        "groupId": "group-001",
        "property": "temperature",
        "value": 25.5,
        "valueType": 1,
        "deviceId": "device-001",
        "sensorKey": "temp01",
        "alarmLevel": 0
      }
    ]
  }
}
```

**过滤逻辑说明**:
- `isNormal = true`: 查询正常数据（在告警记录表中没有对应记录）
- `isNormal = false`: 查询异常数据（在告警记录表中有对应记录）
- 通过关联 `ast_alarm_record_item` 表判断数据状态

---

## 数据结构汇总

### 值类型枚举
```typescript
enum DataValueTypeEnum {
  Int = 0,        // 整数
  Float = 1,      // 浮点数
  String = 2,     // 字符串
  Json = 3,       // JSON对象
  Enum = 4        // 枚举
}
```

### 告警级别枚举
```typescript
enum AlarmLevelEnum {
  Normal = 0,     // 正常
  Warning = 1,    // 预警
  Alarm = 2       // 告警
}
```

### 分页结果结构
```typescript
interface PagedResultDto<T> {
  totalCount: number;
  items: T[];
}
```

---

## 错误码

| 错误码 | 说明 |
|-------|------|
| 400 | 请求参数错误（如缺少必填参数、参数格式错误） |
| 401 | 未授权（Token无效） |
| 403 | 无权限 |
| 500 | 服务器错误 |
| 502 | 数据库连接失败 |
| 504 | 查询超时 |

---

## 前端使用示例

### 获取最新点位数据
```typescript
const pointIds = ['point-001', 'point-002'];
const params = new URLSearchParams();
pointIds.forEach(id => params.append('pointIds[]', id));
params.append('deviceId', 'device-001');
params.append('sensorKey', 'temp01');

const response = await fetch(`/api/app/voiceprint/point-data/latest?${params}`, {
  headers: {
    'Authorization': `Bearer ${token}`
  }
});

const result = await response.json();
```

### 获取历史数据（聚合）
```typescript
const params = new URLSearchParams({
  deviceId: 'device-001',
  sensorKey: 'temp01',
  startTime: '2026-06-01T00:00:00Z',
  endTime: '2026-06-03T00:00:00Z',
  intervalInMinutes: '60',  // 每小时一个数据点
  skipCount: '0',
  maxResultCount: '48'
});

const response = await fetch(`/api/app/voiceprint/point-data/history?${params}`, {
  headers: {
    'Authorization': `Bearer ${token}`
  }
});

const result = await response.json();
// 使用 result.data.items 绘制趋势图
```

### 上报点位数据
```typescript
const data = {
  ts: new Date().toISOString(),
  pointId: 'point-001',
  groupId: 'group-001',
  property: 'temperature',
  value: 25.5,
  valueType: 1,
  deviceId: 'device-001',
  sensorKey: 'temp01',
  alarmLevel: 0
};

const response = await fetch('/api/app/voiceprint/point-data', {
  method: 'POST',
  headers: {
    'Authorization': `Bearer ${token}`,
    'Content-Type': 'application/json'
  },
  body: JSON.stringify(data)
});

const result = await response.json();
```

### 查询点位数据列表
```typescript
const params = new URLSearchParams({
  deviceId: 'device-001',
  sensorKey: 'temp01',
  startTime: '2026-06-01T00:00:00Z',
  endTime: '2026-06-03T00:00:00Z',
  skipCount: '0',
  maxResultCount: '50',
  sorting: 'ts desc'
});

const response = await fetch(`/api/app/voiceprint/point-data?${params}`, {
  headers: {
    'Authorization': `Bearer ${token}`
  }
});

const result = await response.json();
```

---

## 性能优化建议

### 历史数据查询优化
1. **使用聚合接口**: 对于大数据量查询，优先使用 `/point-data/history` 接口
2. **合理设置时间间隔**: 根据展示需求设置合适的 `intervalInMinutes`
   - 实时监控: 1-5 分钟
   - 趋势分析: 30-60 分钟
   - 长期统计: 1440 分钟（1天）
3. **限制查询范围**: 单次查询时间范围不超过7天

### 批量查询优化
1. **使用批量接口**: 多个点位查询使用 `/point-data/batch` 或 `/point-data/latest`
2. **控制批量大小**: 单次查询点位数量不超过100个
3. **添加必要的过滤条件**: 如 `deviceId`、`sensorKey` 等

### 数据上报优化
1. **使用分组ID**: 同一批次的数据使用相同的 `groupId`
2. **批量上报**: 一次请求包含多个点位数据
3. **设置合理的告警级别**: 减少误报

---

## 相关文档

- [[PointDataService]] - 点位数据服务
- [[IAstPointDataService]] - 点位数据核心服务
- [[传感器数据管理服务]] - 传感器数据管理流程
- [[PointDataCleanupJob]] - 数据清理任务
- [[TDengine集成]] - TDengine时序数据库集成

## 前端使用位置

- `src/hooks/point-data/useLatestPointData.ts` - 最新点位数据
- `src/hooks/point-data/useHistoryPointData.ts` - 历史数据
- `src/hooks/point-data/usePointDataList.ts` - 数据列表
- `src/hooks/point-data/useFilteredPointData.ts` - 按状态过滤
- `src/components/charts/TrendChart.tsx` - 趋势图组件
