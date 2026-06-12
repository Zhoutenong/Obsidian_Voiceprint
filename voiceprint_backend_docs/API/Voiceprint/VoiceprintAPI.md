# 声纹模块 API 文档

## 概述
声纹模块提供声纹采集、识别、分析和报告生成的完整 API 接口。

## 基础路径
```
/api/app/voiceprint
```

## 认证
所有接口（除音频上传外）需要 JWT Token 认证：
```
Authorization: Bearer {token}
```

---

## 仪表板

### 获取总览统计

**接口**: `GET /dashboard/overview`

**描述**: 获取声纹监控系统的总览统计数据

**返回**: `VoiceprintDashboardOverviewDto`

### 获取每日巡视记录

**接口**: `GET /dashboard/records`

**请求参数**:
| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `date` | DateTime | 否 | 日期（默认为当天） |

**返回**: `IReadOnlyList<VoiceprintDashboardRecordDto>`

### 获取巡视记录详情

**接口**: `GET /dashboard/records/{groupId:guid}`

**描述**: 获取指定批次ID的巡视记录详情

**返回**: `IReadOnlyList<VoiceprintRecordDetailDto>`

### 获取系统状态

**接口**: `GET /dashboard/system-status`

**描述**: 获取声纹采集和识别系统的运行状态

**返回**: `VoiceprintSystemStatusDto`

### 获取告警列表

**接口**: `GET /dashboard/alarms`

**描述**: 获取仪表板显示的告警列表

**返回**: `VoiceprintDashboardAlarmDto`

### 生成告警报告

**接口**: `POST /dashboard/generate-alarm-report`

**请求参数**:
```json
{
  "alarmIds": ["guid1", "guid2"]
}
```

**返回**: `IReadOnlyList<VoiceprintAudioReportGenerateResultDto>`

---

## 资产管理

### 获取资产树

**接口**: `GET /assets/tree`

**描述**: 获取监控对象的层级结构树

**返回**: `IReadOnlyList<VoiceprintAssetTreeNodeDto>`

### 获取资产详情

**接口**: `GET /assets/{monitoredObjectId:guid}`

**描述**: 获取指定监控对象的详细信息

**返回**: `VoiceprintDeviceInfoDto`

### 获取设备趋势

**接口**: `GET /assets/{monitoredObjectId:guid}/trend`

**请求参数**:
| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `monitoredObjectId` | Guid | 是 | 监控对象ID（路由参数） |
| `startTime` | DateTime | 否 | 起始时间 |

**返回**: `IReadOnlyList<VoiceprintTrendItemDto>`

### 获取异常混合数据

**接口**: `GET /assets/{monitoredObjectId:guid}/anomaly-mix`

**描述**: 获取设备的异常混合数据

**返回**: `IReadOnlyList<VoiceprintAnomalyMixItemDto>`

### 获取设备日志

**接口**: `GET /assets/{monitoredObjectId:guid}/logs`

**请求参数**:
| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `monitoredObjectId` | Guid | 是 | 监控对象ID（路由参数） |
| `startTime` | DateTime? | 否 | 起始时间 |
| `endTime` | DateTime? | 否 | 结束时间 |
| `page` | int | 否 | 页码（默认1） |
| `pageSize` | int | 否 | 每页大小（默认50） |

**返回**: `VoiceprintDeviceLogPagedResultDto`

### 下载已处理音频

**接口**: `GET /assets/{monitoredObjectId:guid}/processed-audios/download`

**请求参数**:
| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `monitoredObjectId` | Guid | 是 | 监控对象ID（路由参数） |
| `startTime` | DateTime | 是 | 起始时间 |
| `endTime` | DateTime | 是 | 结束时间 |

**返回**: `FileStreamResult`（ZIP 压缩包）

---

## 采集器日志

### 获取采集器日志

**接口**: `GET /collector/logs`

**请求参数**:
| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `collectorDeviceId` | string | 否 | 采集器设备ID |
| `groupId` | Guid? | 否 | 批次ID |
| `status` | string | 否 | 状态筛选 |
| `page` | int | 否 | 页码（默认1） |
| `pageSize` | int | 否 | 每页大小（默认50） |

**返回**: `VoiceprintCollectorLogPagedResultDto`

---

## 告警管理

### 获取告警列表

**接口**: `GET /alarms`

**请求参数**:
| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `startTime` | DateTime? | 否 | 起始时间 |
| `endTime` | DateTime? | 否 | 结束时间 |
| `monitoredObjectId` | Guid? | 否 | 监控对象ID |
| `page` | int | 否 | 页码（默认1） |
| `pageSize` | int | 否 | 每页大小（默认50） |

**返回**: `VoiceprintAlarmPagedResultDto`

### 更新告警状态

**接口**: `PUT /alarms/{alarmId:guid}`

**请求参数**: `ProcessVoiceprintAlarmInput`（请求体）

**返回**: `Task`（void）

### 获取告警月度统计

**接口**: `GET /alarms/monthly-stat`

**描述**: 获取近30天告警统计数据

**返回**: `VoiceprintMonthlyStatDto`

### 获取告警设备选项

**接口**: `GET /alarms/device-options`

**描述**: 获取可用于告警筛选的设备选项列表

**返回**: `IReadOnlyList<SimpleDeviceReferenceDto>`

### 按时间范围删除告警

**接口**: `DELETE /alarms/delete-by-time`

**请求参数**:
| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `startTime` | DateTime | 是 | 起始时间 |
| `endTime` | DateTime | 是 | 结束时间 |

**返回**: 包含删除统计的匿名对象

### 模拟告警

**接口**: `POST /alarms/simulate`

**描述**: 模拟生成声纹告警数据（测试工具）

**返回**: 包含模拟结果的匿名对象

---

## 标准音频库管理

### 获取标准音频列表

**接口**: `GET /standard-audios`

**描述**: 按异常类型分组返回标准音频列表，无分页

**返回**: `IReadOnlyList<VoiceprintStandardAudioGroupDto>`

### 创建标准音频

**接口**: `POST /standard-audios`

**Content-Type**: `multipart/form-data`

**请求参数**:
| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `anomalyType` | string | 是 | 异常类型 |
| `file` | IFormFile | 是 | 音频文件 |
| `audioName` | string | 是 | 音频名称 |

**返回**: `Task`（void）

### 获取随机标准音频

**接口**: `GET /standard-audios/random`

**请求参数**:
| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `anomalyType` | string | 否 | 异常类型筛选 |

**返回**: `VoiceprintStandardAudioDto`

---

## 测试音频管理

### 导入测试音频

**接口**: `POST /test-audios/import`

**Content-Type**: `multipart/form-data`

**请求参数**:
| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `file` | IFormFile | 是 | 音频文件 |

**返回**: `VoiceprintStandardAudioImportResultDto`

---

## 算法管理

### 切换算法模式

**接口**: `POST /algorithm/switch`

**请求参数**:
```json
{
  "targetMode": 1
}
```

**返回**: `VoiceprintAlgorithmSwitchResultDto`

---

## 报告管理

### 从音频生成报告

**接口**: `POST /reports/generate-from-audio`

**Content-Type**: `multipart/form-data`

**请求参数**:
| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `audioFile` | IFormFile | 是 | 音频文件 |
| `diagnosisType` | string | 否 | 诊断类型 |
| `audioFileName` | string | 否 | 音频文件名 |
| `resultFileName` | string | 否 | 结果文件名 |

**返回**: `VoiceprintAudioReportGenerateResultDto`

### 导出智能巡视报表

**接口**: `POST /reports/export`

**请求参数**: `VoiceprintReportExportInput`（请求体）

**返回**: `VoiceprintReportExportResultDto`

---

## 运维工具

### 触发音频清理

**接口**: `POST /cleanup/trigger`

**描述**: 手动触发已处理音频的清理任务

**返回**: `bool`

### 修复缺失音频

**接口**: `POST /processed-audios/repair-by-group`

**请求参数**:
| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `deviceId` | string | 是 | 设备ID |
| `sourceDeviceId` | string | 是 | 源设备ID |
| `yearMonth` | string | 是 | 年月（如 "2026-06"） |

**返回**: 包含修复统计的匿名对象

### 修复测试音频异常类型

**接口**: `POST /device-audios/fix-test-audio-anomaly-type`

**描述**: 批量修正测试音频的异常类型字段

**返回**: 包含修复统计的匿名对象

### 取消手动采集

**接口**: `POST /capture/manual-cancel`

**描述**: 终止正在进行的手动采集批次，恢复定时任务

**返回**: `VoiceprintManualCancelResultDto`

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

## 相关文档

- [[Hangfire/Voiceprint/VoiceprintCaptureJob]] - 声纹采集任务
- [[Modules/ast-voiceprint/VoiceprintAudioAppService]] - 声纹音频应用服务
- [[Modules/ast-voiceprint/VoiceprintPortalAppService]] - 声纹门户应用服务

---

**最后更新**: 2026-06-12
