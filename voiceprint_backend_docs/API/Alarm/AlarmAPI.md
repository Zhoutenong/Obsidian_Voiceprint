# 告警模块 API 文档

## 概述
告警模块提供告警查询、统计、处理等功能，是系统告警管理的核心接口。

## 基础路径
```
/api/app/voiceprint/alarms
```

## 认证
所有接口都需要 JWT Token 认证：
```
Authorization: Bearer {token}
```

---

## 告警查询

### 获取告警列表

**接口**: `GET /alarms`

**请求参数**:
```json
{
  "deviceId": "guid",
  "alarmLevel": "Critical",
  "startTime": "2026-06-01",
  "endTime": "2026-06-03",
  "status": 0,
  "pageIndex": 1,
  "pageSize": 20
}
```

**查询参数说明**:

| 参数 | 类型 | 说明 |
|-----|------|------|
| deviceId | string | 设备ID（可选） |
| alarmLevel | string | 告警级别（可选） |
| startTime | string | 开始时间（可选） |
| endTime | string | 结束时间（可选） |
| status | int | 状态：0-未处理，1-已处理 |
| keyword | string | 关键字搜索（可选） |
| pageIndex | int | 页码（默认1） |
| pageSize | int | 每页大小（默认20） |

**响应**:
```json
{
  "success": true,
  "data": {
    "items": [
      {
        "id": "guid",
        "deviceId": "guid",
        "deviceName": "1#电机",
        "pointId": "guid",
        "pointName": "振动监测",
        "alarmType": "Upper",
        "alarmLevel": "Critical",
        "alarmValue": 85.5,
        "threshold": 80.0,
        "alarmTime": "2026-06-03T10:00:00Z",
        "status": 0,
        "handleNote": null,
        "handledAt": null,
        "handledBy": null
      }
    ],
    "total": 150
  }
}
```

### 获取告警详情

**接口**: `GET /alarms/{alarmId}`

**响应**: 返回告警详细信息，包含设备信息、点位信息、历史数据等

---

## 告警统计

### 月度告警统计

**接口**: `GET /alarms/monthly-stat`

**请求参数**:
```json
{
  "deviceId": "guid",
  "startTime": "2026-05-01",
  "endTime": "2026-06-01"
}
```

**响应**:
```json
{
  "success": true,
  "data": {
    "total": 500,
    "byLevel": {
      "Critical": 50,
      "Warning": 200,
      "Info": 250
    },
    "byType": {
      "Upper": 200,
      "Lower": 150,
      "Rate": 100,
      "Sustain": 50
    },
    "byDay": [
      { "date": "2026-05-01", "count": 15 },
      { "date": "2026-05-02", "count": 20 },
      ...
    ],
    "handledRate": 0.85
  }
}
```

### 获取告警设备选项

**接口**: `GET /alarms/device-options`

**响应**:
```json
{
  "success": true,
  "data": [
    {
      "deviceId": "guid",
      "deviceName": "1#电机",
      "alarmCount": 50
    },
    ...
  ]
}
```

---

## 告警处理

### 处理告警

**接口**: `PUT /alarms/{alarmId}`

**请求参数**:
```json
{
  "status": 1,
  "handleNote": "已现场检查，设备正常，误告警"
}
```

**响应**:
```json
{
  "success": true,
  "data": {
    "id": "guid",
    "status": 1,
    "handleNote": "已现场检查，设备正常，误告警",
    "handledAt": "2026-06-03T10:30:00Z",
    "handledBy": "admin"
  }
}
```

### 批量处理告警

**接口**: `PUT /alarms/batch`

**请求参数**:
```json
{
  "alarmIds": ["guid1", "guid2", "guid3"],
  "status": 1,
  "handleNote": "批量处理"
}
```

---

## 告警分类

### 获取告警分类列表

**接口**: `GET /alarm-categories`

**请求参数**:
```json
{
  "keyword": "振动",
  "pageIndex": 1,
  "pageSize": 20
}
```

**响应**:
```json
{
  "success": true,
  "data": {
    "items": [
      {
        "id": "guid",
        "name": "振动告警",
        "code": "VIBRATION",
        "level": "Warning",
        "description": "振动值超过阈值",
        "defaultValue": 5.0,
        "unit": "mm/s"
      }
    ],
    "total": 50
  }
}
```

---

## 首页告警

### 获取首页告警信息

**接口**: `GET /dashboard/alarms`

**注意**: 此接口在仪表板模块下，路径为 `/api/app/voiceprint/dashboard/alarms`

**请求参数**:
```json
{
  "limit": 10
}
```

**响应**:
```json
{
  "success": true,
  "data": {
    "items": [
      {
        "id": "guid",
        "deviceName": "1#电机",
        "alarmLevel": "Critical",
        "alarmTime": "2026-06-03T10:00:00Z",
        "message": "振动值超标：5.2 mm/s"
      }
    ],
    "total": 25,
    "unhandledCount": 15
  }
}
```

---

## 数据结构

### 告警对象

```typescript
interface Alarm {
  id: string;
  deviceId: string;
  deviceName: string;
  pointId: string;
  pointName: string;
  alarmType: 'Upper' | 'Lower' | 'Rate' | 'Sustain';
  alarmLevel: 'Critical' | 'Warning' | 'Info';
  alarmValue: number;
  threshold: number;
  unit: string;
  alarmTime: Date;
  status: number;  // 0-未处理, 1-已处理
  handleNote?: string;
  handledAt?: Date;
  handledBy?: string;
  recoveredAt?: Date;
}
```

### 告警统计对象

```typescript
interface AlarmStatistics {
  total: number;
  byLevel: {
    Critical: number;
    Warning: number;
    Info: number;
  };
  byType: {
    Upper: number;
    Lower: number;
    Rate: number;
    Sustain: number;
  };
  byDay: Array<{
    date: string;
    count: number;
  }>;
  handledRate: number;
}
```

### 告警分类对象

```typescript
interface AlarmCategory {
  id: string;
  name: string;
  code: string;
  level: 'Critical' | 'Warning' | 'Info';
  description: string;
  defaultValue: number;
  unit: string;
  hysteresis?: number;  // 回差
}
```

---

## 告警级别说明

| 级别 | 说明 | 颜色 | 处理时限 |
|-----|------|------|---------|
| Critical | 严重告警 | 红色 | 立即处理 |
| Warning | 警告告警 | 黄色 | 24小时内 |
| Info | 信息提示 | 蓝色 | 无需处理 |

## 告警类型说明

| 类型 | 说明 | 触发条件 |
|-----|------|---------|
| Upper | 上限告警 | 当前值 > 阈值上限 |
| Lower | 下限告警 | 当前值 < 阈值下限 |
| Rate | 变化率告警 | 变化率超过阈值 |
| Sustain | 持续告警 | 值持续异常超过时间 |

---

## 错误码

| 错误码 | 说明 |
|-------|------|
| 400 | 请求参数错误 |
| 401 | 未授权（Token无效） |
| 403 | 无权限 |
| 404 | 告警不存在 |
| 409 | 告警已处理，不能重复处理 |
| 500 | 服务器错误 |

---

## 前端使用示例

### 查询告警列表

```typescript
const response = await fetch('/api/app/voiceprint/alarms?' + new URLSearchParams({
  deviceId: deviceId,
  startTime: '2026-06-01',
  endTime: '2026-06-03',
  status: '0',
  pageIndex: '1',
  pageSize: '20'
}), {
  headers: {
    'Authorization': `Bearer ${token}`
  }
});

const result = await response.json();
```

### 处理告警

```typescript
const response = await fetch(`/api/app/voiceprint/alarms/${alarmId}`, {
  method: 'PUT',
  headers: {
    'Authorization': `Bearer ${token}`,
    'Content-Type': 'application/json'
  },
  body: JSON.stringify({
    status: 1,
    handleNote: '已处理'
  })
});

const result = await response.json();
```

---

## WebSocket 实时推送

告警通过 SignalR 实时推送到前端：

**连接地址**: `/hubs/alarm-notification`

**推送方法**: `OnAlarmTriggered`

**推送数据**:
```json
{
  "alarmId": "guid",
  "deviceName": "1#电机",
  "alarmLevel": "Critical",
  "alarmValue": 85.5,
  "threshold": 80.0,
  "alarmTime": "2026-06-03T10:00:00Z"
}
```

---

## 相关文档

- [[AlarmNotificationHub]] - 告警推送 Hub
- [[AlarmCategoryService]] - 告警分类服务
- [[MonitoredPointService]] - 监测点位服务
- [[告警管理流程]] - 告警管理流程时序图

## 前端使用位置

- `src/hooks/alarm-record/useAlarmList.ts` - 告警列表
- `src/hooks/alarm-record/useAlarmStatistic.ts` - 告警统计
- `src/hooks/alarm-record/useAlarmDeviceOptions.tsx` - 设备选项
- `src/hooks/alarm-record/useAlarmActions.tsx` - 告警操作
- `src/hooks/alarm-record/useDashboardAlarmList.tsx` - 首页告警
