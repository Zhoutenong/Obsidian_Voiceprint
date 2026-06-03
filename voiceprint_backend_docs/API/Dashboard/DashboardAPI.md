# 仪表板 API 文档

## 概述
仪表板 API 提供系统首页数据展示所需的各种统计和查询接口。

## 基础路径
```
/api/app/voiceprint/dashboard
```

## 认证
所有接口都需要 JWT Token 认证：
```
Authorization: Bearer {token}
```

---

## 巡视统计

### 获取巡视统计概览

**接口**: `GET /overview`

**响应**:
```json
{
  "success": true,
  "data": {
    "totalDays": 180,
    "totalPatrols": 360,
    "collectionCount": 10800,
    "averageDailyPatrols": 2.0,
    "todayPatrols": 2,
    "lastPatrolTime": "2026-06-03T08:00:00Z"
  }
}
```

---

## 巡视记录

### 获取指定日期的巡视记录

**接口**: `GET /records`

**请求参数**:
```json
{
  "date": "2026-06-03",
  "deviceId": "guid"
}
```

**响应**:
```json
{
  "success": true,
  "data": {
    "items": [
      {
        "groupId": "guid",
        "taskId": "guid",
        "taskName": "早班巡视",
        "deviceId": "guid",
        "deviceName": "1#电机",
        "executionTime": "2026-06-03T08:00:00Z",
        "status": "Completed",
        "pointCount": 10,
        "successCount": 10,
        "failCount": 0
      }
    ]
  }
}
```

### 获取巡视记录详情

**接口**: `GET /records/{groupId}`

**响应**:
```json
{
  "success": true,
  "data": {
    "groupId": "guid",
    "taskId": "guid",
    "taskName": "早班巡视",
    "executionTime": "2026-06-03T08:00:00Z",
    "status": "Completed",
    "devices": [
      {
        "deviceId": "guid",
        "deviceName": "1#电机",
        "points": [
          {
            "pointId": "guid",
            "pointName": "振动监测",
            "value": 2.5,
            "unit": "mm/s",
            "status": "Normal",
            "imagePath": "/path/to/image.jpg"
          }
        ]
      }
    ],
    "summary": {
      "totalDevices": 5,
      "totalPoints": 50,
      "normalPoints": 48,
      "abnormalPoints": 2,
      "alarms": 1
    }
  }
}
```

---

## 系统状态（电子沙盘）

### 获取系统状态统计

**接口**: `GET /system-status`

**响应**:
```json
{
  "success": true,
  "data": {
    "totalDevices": 100,
    "statusDistribution": {
      "Online": 85,
      "Offline": 10,
      "Fault": 5
    },
    "alarmDistribution": {
      "Critical": 3,
      "Warning": 15,
      "Info": 20
    },
    "devices": [
      {
        "deviceId": "guid",
        "deviceName": "1#电机",
        "status": "Online",
        "hasAlarm": true,
        "alarmLevel": "Warning",
        "location": {
          "x": 100,
          "y": 200,
          "floor": 1
        }
      }
    ]
  }
}
```

---

## 首页告警

### 获取首页告警信息

**接口**: `GET /alarms`

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
        "pointName": "振动监测",
        "alarmLevel": "Critical",
        "alarmValue": 8.5,
        "threshold": 5.0,
        "alarmTime": "2026-06-03T10:00:00Z",
        "message": "振动值超标：8.5 mm/s"
      }
    ],
    "total": 25,
    "unhandledCount": 15
  }
}
```

---

## 报告生成

### 根据告警生成报告

**接口**: `POST /generate-alarm-report`

**请求参数**:
```json
{
  "alarmId": "guid",
  "templateId": "guid",
  "includeHistory": true,
  "includeImages": true
}
```

**响应**:
```json
{
  "success": true,
  "data": {
    "reportId": "guid",
    "downloadUrl": "/path/to/report.docx",
    "filename": "告警分析报告-20260603.docx"
  }
}
```

---

## 数据结构

### 巡视统计对象

```typescript
interface PatrolOverview {
  totalDays: number;
  totalPatrols: number;
  collectionCount: number;
  averageDailyPatrols: number;
  todayPatrols: number;
  lastPatrolTime: Date;
}
```

### 巡视记录对象

```typescript
interface PatrolRecord {
  groupId: string;
  taskId: string;
  taskName: string;
  deviceId: string;
  deviceName: string;
  executionTime: Date;
  status: 'Pending' | 'Running' | 'Completed' | 'Failed';
  pointCount: number;
  successCount: number;
  failCount: number;
}
```

### 巡视记录详情对象

```typescript
interface PatrolRecordDetail {
  groupId: string;
  taskId: string;
  taskName: string;
  executionTime: Date;
  status: string;
  devices: Array<{
    deviceId: string;
    deviceName: string;
    points: Array<{
      pointId: string;
      pointName: string;
      value: number;
      unit: string;
      status: string;
      imagePath?: string;
    }>;
  }>;
  summary: {
    totalDevices: number;
    totalPoints: number;
    normalPoints: number;
    abnormalPoints: number;
    alarms: number;
  };
}
```

### 系统状态对象

```typescript
interface SystemStatus {
  totalDevices: number;
  statusDistribution: {
    Online: number;
    Offline: number;
    Fault: number;
  };
  alarmDistribution: {
    Critical: number;
    Warning: number;
    Info: number;
  };
  devices: Array<{
    deviceId: string;
    deviceName: string;
    status: string;
    hasAlarm: boolean;
    alarmLevel?: string;
    location: {
      x: number;
      y: number;
      floor: number;
    };
  }>;
}
```

---

## 前端使用示例

### 获取首页数据

```typescript
// 并行获取多个数据源
const [overview, systemStatus, alarms] = await Promise.all([
  fetch('/api/app/voiceprint/dashboard/overview').then(r => r.json()),
  fetch('/api/app/voiceprint/dashboard/system-status').then(r => r.json()),
  fetch('/api/app/voiceprint/dashboard/alarms?limit=10').then(r => r.json())
]);

// 渲染首页
renderDashboard({ overview, systemStatus, alarms });
```

### 获取巡视记录

```typescript
const response = await fetch('/api/app/voiceprint/dashboard/records?' + new URLSearchParams({
  date: '2026-06-03'
}), {
  headers: {
    'Authorization': `Bearer ${token}`
  }
});

const result = await response.json();
```

### 生成告警报告

```typescript
const response = await fetch('/api/app/voiceprint/dashboard/generate-alarm-report', {
  method: 'POST',
  headers: {
    'Authorization': `Bearer ${token}`,
    'Content-Type': 'application/json'
  },
  body: JSON.stringify({
    alarmId: alarmId,
    templateId: templateId,
    includeHistory: true,
    includeImages: true
  })
});

const result = await response.json();

// 下载报告
window.location.href = result.data.downloadUrl;
```

---

## WebSocket 实时推送

仪表板数据通过 SignalR 实时推送：

**连接地址**: `/hubs/dashboard`

**推送方法**:
- `OnSystemStatusChanged` - 系统状态变化
- `OnNewAlarm` - 新增告警
- `OnPatrolCompleted` - 巡视完成

---

## 相关文档

- [[仪表板与巡视流程]] - 仪表板与巡视流程时序图
- [[PatrolTaskService]] - 巡检任务服务
- [[PatrolRecordService]] - 巡检记录服务
- [[DeviceService]] - 设备服务

## 前端使用位置

- `src/hooks/patrol/usePatrolStatistic.ts` - 巡视统计
- `src/hooks/patrol/usePatrolRecord.ts` - 巡视记录
- `src/hooks/patrol/usePatrolResult.ts` - 巡视结果
- `src/hooks/ledger/useDeviceStatistics.ts` - 设备统计
- `src/hooks/alarm-record/useDashboardAlarmList.tsx` - 首页告警
