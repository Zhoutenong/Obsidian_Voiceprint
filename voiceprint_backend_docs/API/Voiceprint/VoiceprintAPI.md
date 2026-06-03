# 声纹模块 API 文档

## 概述
声纹模块提供声纹采集、识别、分析和报告生成的完整 API 接口。

## 基础路径
```
/api/app/voiceprint
```

## 认证
所有接口都需要 JWT Token 认证：
```
Authorization: Bearer {token}
```

---

## 仪表板

### 获取总览统计

**接口**: `GET /dashboard/overview`

**描述**: 获取声纹监控系统的总览统计数据

**响应**:
```json
{
  "success": true,
  "data": {
    "totalDays": 30,
    "totalCaptureCount": 150
  }
}
```

### 获取每日巡视记录

**接口**: `GET /dashboard/records`

**请求参数**:
```
date: 日期（默认为当天）
```

**响应**: 返回指定日期的巡视记录列表

### 获取巡视记录详情

**接口**: `GET /dashboard/records/{groupId:guid}`

**描述**: 获取指定批次ID的巡视记录详情

### 获取系统状态

**接口**: `GET /dashboard/system-status`

**描述**: 获取声纹采集和识别系统的运行状态

### 获取告警列表

**接口**: `GET /dashboard/alarms`

**描述**: 获取仪表板显示的告警列表

### 生成告警报告

**接口**: `POST /dashboard/generate-alarm-report`

**描述**: 生成告警统计报告

---

## 资产管理

### 获取资产树

**接口**: `GET /assets/tree`

**描述**: 获取监控对象的层级结构树

### 获取资产详情

**接口**: `GET /assets/{monitoredObjectId:guid}`

**描述**: 获取指定监控对象的详细信息

### 获取设备趋势

**接口**: `GET /assets/{monitoredObjectId:guid}/trend`

**描述**: 获取设备声纹趋势数据

### 获取异常混合数据

**接口**: `GET /assets/{monitoredObjectId:guid}/anomaly-mix`

**描述**: 获取设备的异常混合数据

### 获取设备日志

**接口**: `GET /assets/{monitoredObjectId:guid}/logs`

**描述**: 获取设备的操作和事件日志

### 下载已处理音频

**接口**: `GET /assets/{monitoredObjectId:guid}/processed-audios/download`

**描述**: 批量下载指定设备的已处理音频文件

---

## 采集器日志

### 获取采集器日志

**接口**: `GET /collector/logs`

**描述**: 获取声纹采集器的运行日志

**请求参数**:
```
deviceId: 设备ID（可选）
startTime: 开始时间
endTime: 结束时间
pageIndex: 页码
pageSize: 每页大小
```

**响应**:
```json
{
  "success": true,
  "data": {
    "totalCount": 100,
    "items": [
      {
        "id": "guid",
        "deviceId": "guid",
        "deviceName": "1#电机",
        "collectedAt": "2026-06-03T10:00:00Z",
        "status": "Success",
        "errorMessage": null
      }
    ]
  }
}
```

---

## 告警管理

### 获取告警列表

**接口**: `GET /alarms`

**描述**: 获取声纹告警记录列表

### 更新告警状态

**接口**: `PUT /alarms/{alarmId:guid}`

**描述**: 更新告警的处理状态

### 获取告警月度统计

**接口**: `GET /alarms/monthly-stat`

**描述**: 获取告警的月度统计数据

### 获取告警设备选项

**接口**: `GET /alarms/device-options`

**描述**: 获取可用于告警筛选的设备选项列表

---

## 标准音频库管理

### 获取标准音频列表

**接口**: `GET /standard-audios`

**请求参数**:
```json
{
  "keyword": "电机",
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
        "name": "电机正常声音",
        "deviceType": "Motor",
        "audioPath": "/path/to/audio.wav",
        "features": "...",
        "createdAt": "2026-06-01T10:00:00Z"
      }
    ],
    "total": 100
  }
}
```

### 创建标准音频

**接口**: `POST /standard-audios`

**请求参数**:
```json
{
  "name": "电机正常声音",
  "deviceType": "Motor",
  "description": "电机正常运行时的声音特征"
}
```

**响应**: 返回创建的标准音频对象

### 获取随机标准音频

**接口**: `GET /standard-audios/random`

**描述**: 获取一个随机的标准音频用于测试或对比

**响应**:
```json
{
  "success": true,
  "data": {
    "id": "guid",
    "name": "电机正常声音",
    "deviceType": "Motor",
    "audioPath": "/path/to/audio.wav",
    "features": "...",
    "createdAt": "2026-06-01T10:00:00Z"
  }
}
```


---

## 测试音频管理


### 导入测试音频并识别

**接口**: `POST /test-audios/import`

**Content-Type**: `multipart/form-data`

**请求参数**:
```
file: 音频文件
deviceId: 设备ID
deviceName: 设备名称
triggerRecognition: 是否自动识别（true/false）
```

**响应**:
```json
{
  "success": true,
  "data": {
    "audioId": "guid",
    "status": "Imported",
    "recognitionTaskId": "guid"
  }
}
```




---

## 音频记录管理

### 获取音频记录列表

**接口**: `GET /audios`

**请求参数**:
```json
{
  "deviceId": "guid",
  "startTime": "2026-06-01",
  "endTime": "2026-06-03",
  "status": "Processed",
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
        "deviceId": "guid",
        "deviceName": "1#电机",
        "originalPath": "/path/to/original.wav",
        "processedPath": "/path/to/processed.wav",
        "status": "Processed",
        "capturedAt": "2026-06-01T10:00:00Z",
        "processedAt": "2026-06-01T10:01:00Z"
      }
    ],
    "total": 200
  }
}
```

### 获取音频详情

**接口**: `GET /audios/{id}`

**响应**: 返回音频记录详情，包含识别结果、特征数据等

### 批量下载音频

**接口**: `POST /audios/download`

**请求参数**:
```json
{
  "audioIds": ["guid1", "guid2", "guid3"]
}
```

**响应**:
```json
{
  "success": true,
  "data": {
    "downloadUrl": "/api/app/voiceprint/audios/download/token",
    "filename": "audios-20260603.zip",
    "size": 10240000
  }
}
```

---

## 报告管理

### 从音频生成报告

**接口**: `POST /reports/generate-from-audio`

**请求参数**:
```json
{
  "audioId": "guid",
  "templateId": "guid",
  "reportType": "Single"
}
```

**响应**:
```json
{
  "success": true,
  "data": {
    "reportId": "guid",
    "downloadUrl": "/path/to/report.docx",
    "filename": "声纹分析报告-20260603.docx"
  }
}
```

### 导出智能巡视报表

**接口**: `POST /reports/export`

**请求参数**:
```json
{
  "startDate": "2026-06-01",
  "endDate": "2026-06-03",
  "deviceIds": ["guid1", "guid2"],
  "includeCharts": true,
  "format": "Docx"
}
```

**响应**:
```json
{
  "success": true,
  "data": {
    "taskId": "guid",
    "status": "Processing"
  }
}
```


---

## 算法管理

### 切换算法模式

**接口**: `POST /algorithm/switch`

**请求参数**:
```json
{
  "mode": "Auto",
  "config": {
    "threshold": 0.8,
    "sensitivity": "High"
  }
}
```

**响应**:
```json
{
  "success": true,
  "data": {
    "currentMode": "Auto",
    "switchedAt": "2026-06-03T10:00:00Z"
  }
}
```


---

## 数据结构

### 标准音频对象

```typescript
interface StandardAudio {
  id: string;
  name: string;
  deviceType: string;
  audioPath: string;
  features: string;  // 特征向量（Base64）
  description?: string;
  createdAt: Date;
  updatedAt: Date;
}
```

### 测试音频对象

```typescript
interface TestAudio {
  id: string;
  deviceId: string;
  deviceName: string;
  audioPath: string;
  recognitionStatus: 'Pending' | 'Processing' | 'Completed' | 'Failed';
  similarity?: number;
  matchedStandardAudioId?: string;
  matchedStandardAudioName?: string;
  errorMessage?: string;
  createdAt: Date;
  processedAt?: Date;
}
```

### 音频记录对象

```typescript
interface AudioRecord {
  id: string;
  deviceId: string;
  deviceName: string;
  originalPath: string;
  processedPath: string;
  status: 'Pending' | 'Processing' | 'Processed' | 'Failed' | 'Cleaned';
  capturedAt: Date;
  processedAt?: Date;
  cleanedAt?: Date;
  recognitionResult?: RecognitionResult;
}
```

### 报告对象

```typescript
interface Report {
  id: string;
  reportType: 'Single' | 'Patrol' | 'Batch';
  format: 'Docx' | 'Pdf';
  downloadUrl: string;
  filename: string;
  size: number;
  generatedAt: Date;
  expiresAt: Date;
}
```

---

## 错误码

| 错误码 | 说明 |
|-------|------|
| 400 | 请求参数错误 |
| 401 | 未授权（Token无效） |
| 403 | 无权限 |
| 404 | 资源不存在 |
| 409 | 资源冲突 |
| 500 | 服务器错误 |

### 错误响应格式

```json
{
  "success": false,
  "error": {
    "code": "ErrorCode",
    "message": "错误描述",
    "details": "详细错误信息"
  }
}
```

---

## 前端使用示例

### 导入并识别音频

```typescript
const formData = new FormData();
formData.append('file', audioFile);
formData.append('deviceId', deviceId);
formData.append('deviceName', deviceName);
formData.append('triggerRecognition', 'true');

const response = await fetch('/api/app/voiceprint/test-audios/import', {
  method: 'POST',
  headers: {
    'Authorization': `Bearer ${token}`
  },
  body: formData
});

const result = await response.json();
```

### 获取识别结果

```typescript
const response = await fetch(
  `/api/app/voiceprint/test-audios/${audioId}/recognition-result`,
  {
    headers: {
      'Authorization': `Bearer ${token}`
    }
  }
);

const result = await response.json();
console.log(`相似度: ${result.data.similarity}%`);
```

---

## 相关文档

- [[VoiceprintCaptureJob]] - 声纹采集任务
- [[VoiceAudioAppService]] - 声纹音频应用服务
- [[VoiceprintPortalAppService]] - 声纹门户应用服务
