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

### 更新标准音频

**接口**: `PUT /standard-audios/{id}`

**请求参数**: 同创建

### 删除标准音频

**接口**: `DELETE /standard-audios/{id}`

---

## 测试音频管理

### 获取测试音频列表

**接口**: `GET /test-audios`

**请求参数**:
```json
{
  "deviceId": "guid",
  "startTime": "2026-06-01",
  "endTime": "2026-06-03",
  "recognitionStatus": "Pending",
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
        "audioPath": "/path/to/audio.wav",
        "recognitionStatus": "Completed",
        "similarity": 85.5,
        "matchedStandardAudioId": "guid",
        "createdAt": "2026-06-01T10:00:00Z"
      }
    ],
    "total": 50
  }
}
```

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

### 触发识别

**接口**: `POST /test-audios/{id}/recognize`

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

### 获取识别结果

**接口**: `GET /test-audios/{id}/recognition-result`

**响应**:
```json
{
  "success": true,
  "data": {
    "similarity": 85.5,
    "matchedStandardAudioId": "guid",
    "matchedStandardAudioName": "电机正常声音",
    "features": "...",
    "recognizedAt": "2026-06-01T10:05:00Z"
  }
}
```

### 删除测试音频

**接口**: `DELETE /test-audios/{id}`

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

### 获取报告列表

**接口**: `GET /reports`

**请求参数**:
```json
{
  "startTime": "2026-06-01",
  "endTime": "2026-06-03",
  "reportType": "Patrol",
  "pageIndex": 1,
  "pageSize": 20
}
```

### 删除报告

**接口**: `DELETE /reports/{id}`

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

### 获取算法状态

**接口**: `GET /algorithm/status`

**响应**:
```json
{
  "success": true,
  "data": {
    "currentMode": "Auto",
    "isProcessing": false,
    "queueSize": 0,
    "lastProcessTime": "2026-06-03T09:55:00Z"
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
