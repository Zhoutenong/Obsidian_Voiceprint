# 报告模块 API 文档

## 概述
报告模块提供智能变电站的综合运行报告生成和导出功能，支持设备运行状态汇总、告警统计和设备健康分析。

## 基础路径
```
/api/app/report
```

## 认证
所有接口都需要 JWT Token 认证：
```
Authorization: Bearer {token}
```

---

## 综合运行报告

### 生成综合运行报告

**接口**: `POST /comprehensive-operation-report`

**描述**: 生成指定变电站和时间段内的综合运行报告，包含设备运行状态、告警信息和设备健康分析。

**请求参数**:
```json
{
  "substationName": "小东庄主变电所",
  "startTime": "2026-06-01T00:00:00",
  "endTime": "2026-06-03T23:59:59"
}
```

**请求参数说明**:

| 参数 | 类型 | 必填 | 说明 |
|-----|------|------|------|
| substationName | string | 是 | 变电站名称 |
| startTime | string | 是 | 报告开始时间（ISO 8601格式） |
| endTime | string | 是 | 报告结束时间（ISO 8601格式） |

**响应**:
```json
{
  "reportTitle": "小东庄主变电所综合运行报告",
  "reportPeriodStart": "2026-06-01 00:00:00",
  "reportPeriodEnd": "2026-06-03 23:59:59",
  "deviceTotal": 45,
  "gisCount": 20,
  "groundTransformerCount": 10,
  "oilTransformerCount": 15,
  "summaryTotal": 45,
  "alarmCount": 5,
  "normalCount": 40,
  "alarmDevices": [
    {
      "index": 1,
      "alarmTime": "2026-06-02 15:30:25",
      "location": "1号开关室",
      "device": "1#GIS柜",
      "monitorPointName": "动触头温度",
      "reason": "温度超过阈值上限",
      "level": "严重",
      "count": "15",
      "monitorPointAlarmCount": "3",
      "status": "未处理"
    }
  ],
  "normalDevices": [
    {
      "index": 1,
      "location": "2号开关室",
      "device": "2#GIS柜",
      "operationStatusSummary": [
        "动触头温度: 进线25-28°C, 出线26-29°C - 正常",
        "局放超声波峰值: 15 dB - 正常",
        "操作分闸时间: 45ms - 正常"
      ]
    }
  ],
  "summary": "本周期内设备运行总体稳定，5台设备出现告警已记录，40台设备运行正常。建议关注1#GIS柜的动触头温度变化趋势。"
}
```

**响应数据结构**:

| 字段 | 类型 | 说明 |
|-----|------|------|
| reportTitle | string | 报告标题 |
| reportPeriodStart | string | 报告周期开始时间 |
| reportPeriodEnd | string | 报告周期结束时间 |
| deviceTotal | int | 设备总数 |
| gisCount | int | GIS设备数量 |
| groundTransformerCount | int | 接地变压器数量 |
| oilTransformerCount | int | 油变压器数量 |
| summaryTotal | int | 汇总总数 |
| alarmCount | int | 告警设备数量 |
| normalCount | int | 正常设备数量 |
| alarmDevices | array | 告警设备列表 |
| normalDevices | array | 正常设备列表 |
| summary | string | 日常运行结论 |

---

## 数据结构

### 请求报告数据 (RequestReportDataDto)

```typescript
interface RequestReportDataDto {
  substationName: string;      // 变电站名称
  startTime: string;           // 开始时间 (ISO 8601)
  endTime: string;             // 结束时间 (ISO 8601)
}
```

### 综合运行报告 (ComprehensiveOperationReportDto)

```typescript
interface ComprehensiveOperationReportDto {
  reportTitle: string;
  reportPeriodStart: string;
  reportPeriodEnd: string;
  deviceTotal: number;
  gisCount: number;
  groundTransformerCount: number;
  oilTransformerCount: number;
  summaryTotal: number;
  alarmCount: number;
  normalCount: number;
  alarmDevices: AlarmDeviceDto[];
  normalDevices: NormalDeviceDto[];
  summary: string;
}
```

---

## 错误码

| 错误码 | 说明 |
|-------|------|
| 400 | 请求参数错误 |
| 401 | 未授权（Token无效） |
| 403 | 无权限 |
| 404 | 变电站不存在 |
| 500 | 服务器错误 |

---

## 前端使用示例

```typescript
const response = await fetch('/api/app/report/comprehensive-operation-report', {
  method: 'POST',
  headers: {
    'Authorization': `Bearer ${token}`,
    'Content-Type': 'application/json'
  },
  body: JSON.stringify({
    substationName: '小东庄主变电所',
    startTime: '2026-06-01T00:00:00',
    endTime: '2026-06-03T23:59:59'
  })
});

const report = await response.json();
```
