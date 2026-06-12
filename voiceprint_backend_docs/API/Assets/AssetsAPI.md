# 设备台账 API 文档

## 概述
设备台账模块提供设备树形结构、基础信息、运行趋势、异常分布、巡检日志和音频下载等功能。

## 基础路径
```
/api/app/voiceprint/assets
```

## 认证
所有接口都需要 JWT Token 认证：
```
Authorization: Bearer {token}
```

---

## 设备台账树

### 获取设备台账树

**接口**: `GET /assets/tree`

**描述**: 获取所有监测对象的树形结构，支持层级关系展示

**请求参数**: 无

**响应**:
```json
{
  "success": true,
  "data": [
    {
      "id": "guid",
      "name": "1号主变",
      "status": "Normal",
      "parentId": null,
      "children": [
        {
          "id": "guid",
          "name": "1#电机",
          "status": "Normal",
          "parentId": "guid",
          "children": []
        }
      ]
    }
  ]
}
```

**数据结构**:
```typescript
interface VoiceprintAssetTreeNode {
  id: string;
  name: string;
  status: 'Normal' | 'Abnormal';
  parentId?: string;
  children: VoiceprintAssetTreeNode[];
}
```

---

## 设备基础信息

### 获取设备基础信息

**接口**: `GET /assets/{monitoredObjectId}`

**描述**: 获取指定设备的详细信息，包含属性组和属性值

**请求参数**:

| 参数 | 类型 | 说明 |
|-----|------|------|
| monitoredObjectId | guid | 设备ID（路径参数） |

**响应**:
```json
{
  "success": true,
  "data": {
    "id": "guid",
    "name": "1#电机",
    "status": "Normal",
    "parent": {
      "id": "guid",
      "name": "1号主变"
    },
    "groups": [
      {
        "id": "guid",
        "name": "基本信息",
        "attributes": [
          {
            "id": "guid",
            "name": "额定功率",
            "value": "1000kW",
            "description": "设备额定功率"
          }
        ]
      }
    ]
  }
}
```

**数据结构**:
```typescript
interface VoiceprintDeviceInfo {
  id: string;
  name: string;
  status: 'Normal' | 'Abnormal';
  parent?: SimpleDeviceReference;
  groups: VoiceprintDeviceAttrGroup[];
}

interface SimpleDeviceReference {
  id: string;
  name: string;
}

interface VoiceprintDeviceAttrGroup {
  id: string;
  name: string;
  attributes: VoiceprintDeviceAttr[];
}

interface VoiceprintDeviceAttr {
  id: string;
  name: string;
  value: string;
  description?: string;
}
```

---

## 设备运行趋势

### 获取设备运行趋势

**接口**: `GET /assets/{monitoredObjectId}/trend`

**描述**: 获取指定设备在指定时间范围内的运行趋势数据

**请求参数**:

| 参数 | 类型 | 说明 |
|-----|------|------|
| monitoredObjectId | guid | 设备ID（路径参数） |
| startTime | datetime | 开始时间（可选，默认7天前） |

**响应**:
```json
{
  "success": true,
  "data": [
    {
      "collectedAt": "2026-06-03T10:00:00Z",
      "isNormal": true
    },
    {
      "collectedAt": "2026-06-03T11:00:00Z",
      "isNormal": false
    }
  ]
}
```

**数据结构**:
```typescript
interface VoiceprintTrendItem {
  collectedAt: Date;
  isNormal: boolean;
}
```

---

## 设备异常分布

### 获取设备异常分布

**接口**: `GET /assets/{monitoredObjectId}/anomaly-mix`

**描述**: 获取指定设备的异常类型分布统计

**请求参数**:

| 参数 | 类型 | 说明 |
|-----|------|------|
| monitoredObjectId | guid | 设备ID（路径参数） |

**响应**:
```json
{
  "success": true,
  "data": [
    {
      "anomalyType": "运行正常",
      "count": 150,
      "ratio": 0.75
    },
    {
      "anomalyType": "风扇轴承摩擦",
      "count": 30,
      "ratio": 0.15
    },
    {
      "anomalyType": "夹件螺栓松动",
      "count": 20,
      "ratio": 0.10
    }
  ]
}
```

**数据结构**:
```typescript
interface VoiceprintAnomalyMixItem {
  anomalyType: string;
  count: number;
  ratio: number;
}
```

**异常类型说明**:
- 运行正常
- 夹件螺栓松动
- 风扇轴承摩擦
- 局部放电
- 励磁
- 连接螺栓松动
- 焊渣异物
- 铁芯松动50%励磁
- 线圈完全松动励磁

---

## 设备巡检日志

### 获取设备巡检日志

**接口**: `GET /assets/{monitoredObjectId}/logs`

**描述**: 获取指定设备的巡检记录列表，支持分页查询

**请求参数**:

| 参数 | 类型 | 说明 |
|-----|------|------|
| monitoredObjectId | guid | 设备ID（路径参数） |
| startTime | datetime? | 开始时间（可选） |
| endTime | datetime? | 结束时间（可选） |
| page | int | 页码（默认1） |
| pageSize | int | 每页大小（默认50，最大200） |

**响应**:
```json
{
  "success": true,
  "data": {
    "totalCount": 150,
    "items": [
      {
        "recordId": "guid",
        "anomalyType": "运行正常",
        "audioPath": "/audio/processed/device_20260603100000.wav",
        "collectedAt": "2026-06-03T10:00:00Z",
        "isNormal": true,
        "standardNormalAudioPath": "/audio/library/normal_sample.wav",
        "standardAnomalyAudioPath": null
      }
    ]
  }
}
```

**数据结构**:
```typescript
interface VoiceprintDeviceLogPagedResult {
  totalCount: number;
  items: VoiceprintDeviceLog[];
}

interface VoiceprintDeviceLog {
  recordId: string;
  anomalyType: string;
  audioPath: string;
  collectedAt: Date;
  isNormal: boolean;
  standardNormalAudioPath?: string;
  standardAnomalyAudioPath?: string;
}
```

---

## 批量下载音频

### 批量下载处理后的音频

**接口**: `POST /assets/{monitoredObjectId}/processed-audios/download`

**描述**: 批量下载指定设备在时间范围内的处理后音频文件，打包为ZIP格式

**请求参数**:

| 参数 | 类型 | 说明 |
|-----|------|------|
| monitoredObjectId | guid | 设备ID（路径参数） |
| startTime | datetime | 开始时间（可选，默认7天前） |
| endTime | datetime | 结束时间（可选，默认当前时间） |

**请求体**:
```json
{
  "startTime": "2026-06-01T00:00:00Z",
  "endTime": "2026-06-03T00:00:00Z"
}
```

**响应**:
```
Content-Type: application/zip
Content-Disposition: attachment; filename="1号电机_202606010000_202606030000.zip"
```

返回 ZIP 文件流，文件命名规则：`{设备名称}_{开始时间}_{结束时间}.zip`

ZIP 内部文件命名规则：`{设备名称}_{采集时间}.wav`

---

## 数据结构汇总

### 设备状态枚举
```typescript
type DeviceStatus = 'Normal' | 'Abnormal';
```

### 异常类型枚举
```typescript
type AnomalyType =
  | '运行正常'
  | '夹件螺栓松动'
  | '风扇轴承摩擦'
  | '局部放电'
  | '励磁'
  | '连接螺栓松动'
  | '焊渣异物'
  | '铁芯松动50%励磁'
  | '线圈完全松动励磁';
```

---

## 错误码

| 错误码 | 说明 |
|-------|------|
| 400 | 请求参数错误 |
| 401 | 未授权（Token无效） |
| 403 | 无权限 |
| 404 | 设备不存在 |
| 500 | 服务器错误 |

---

## 前端使用示例

### 获取设备树
```typescript
const response = await fetch('/api/app/voiceprint/assets/tree', {
  headers: {
    'Authorization': `Bearer ${token}`
  }
});

const result = await response.json();
```

### 获取设备基础信息
```typescript
const response = await fetch(`/api/app/voiceprint/assets/${deviceId}`, {
  headers: {
    'Authorization': `Bearer ${token}`
  }
});

const result = await response.json();
```

### 获取设备运行趋势
```typescript
const startTime = new Date(Date.now() - 7 * 24 * 60 * 60 * 1000);
const response = await fetch(
  `/api/app/voiceprint/assets/${deviceId}/trend?startTime=${startTime.toISOString()}`,
  {
    headers: {
      'Authorization': `Bearer ${token}`
    }
  }
);

const result = await response.json();
```

### 获取设备巡检日志
```typescript
const response = await fetch(
  `/api/app/voiceprint/assets/${deviceId}/logs?page=1&pageSize=50`,
  {
    headers: {
      'Authorization': `Bearer ${token}`
    }
  }
);

const result = await response.json();
```

### 批量下载音频
```typescript
const response = await fetch(
  `/api/app/voiceprint/assets/${deviceId}/processed-audios/download?startTime=${start}&endTime=${end}`,
  {
    headers: {
      'Authorization': `Bearer ${token}`
    }
  });

const blob = await response.blob();
const url = window.URL.createObjectURL(blob);
const a = document.createElement('a');
a.href = url;
a.download = response.headers.get('Content-Disposition')?.match(/filename=(.*)/)?.[1] || 'audio.zip';
a.click();
window.URL.revokeObjectURL(url);
```

---

## 相关文档

- [[VoiceprintPortalAppService]] - 设备门户服务
- [[MonitoredObjectService]] - 监测对象服务
- [[Modules/ast-intellisub/DeviceService]] - 设备管理流程
- [[声纹分析流程]] - 声纹分析流程

## 前端使用位置

- `src/hooks/assets/useAssetTree.ts` - 设备树
- `src/hooks/assets/useDeviceInfo.ts` - 设备信息
- `src/hooks/assets/useDeviceTrend.ts` - 设备趋势
- `src/hooks/assets/useDeviceAnomalyMix.ts` - 异常分布
- `src/hooks/assets/useDeviceLogs.ts` - 巡检日志
