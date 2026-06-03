# VoiceprintAudio 内部运维接口文档

## 文档说明

**重要提示**：本文档描述的是声纹模块的**内部运维接口**，主要用于系统内部服务调用、运维管理和测试。

- **调用方**：Python 算法服务、后台任务、运维脚本
- **认证方式**：API Key（通过 `X-Api-Key` 请求头）
- **前端使用**：请参考 [前端接口文档](../../../../frontend-api-interfaces.md) 中的 23 个前端专用接口

---

## 接口概览

| 序号 | 接口路径 | 方法 | 功能 | 调用方 | 使用场景 |
|------|---------|------|------|--------|---------|
| 1 | `/voiceprint/result` | POST | Python 回写识别结果 | Python 算法服务 | 识别完成后回写结果 |
| 2 | `/voiceprint/cleanup/trigger` | POST | 触发清理任务 | 运维/调试 | 手动触发音频清理 |
| 3 | `/voiceprint/processed-audios/repair-by-group` | POST | 按分组修复音频 | 运维 | 补齐缺失的处理后音频 |
| 4 | `/voiceprint/capture/manual-cancel` | POST | 手动取消采集 | 运维 | 终止手动采集批次 |
| 5 | `/voiceprint/alarms/simulate` | POST | 模拟告警 | 测试 | 生成测试告警数据 |
| 6 | `/voiceprint/alarms/delete-by-time` | DELETE | 批量删除告警 | 运维 | 清理历史告警数据 |
| 7 | `/voiceprint/device-audios/fix-test-audio-anomaly-type` | POST | 修复测试音频类型 | 运维 | 批量修正异常类型 |

---

## 基础配置

### API Key 认证

所有接口都使用 API Key 进行认证，配置在 `appsettings.json`：

```json
{
  "VoiceprintStorage": {
    "ApiKey": "your-secret-api-key"
  }
}
```

**请求头**：
```
X-Api-Key: your-secret-api-key
```

**注意**：如果 `ApiKey` 未配置，则不进行认证检查（仅用于开发环境）。

---

## 接口详情

### 1. Python 回写识别结果

**接口路径**：`POST /api/app/voiceprint/result`

**功能说明**：Python 算法服务完成声纹识别后，将识别结果回写到数据库。

**调用方**：Python 算法服务

**请求头**：
```
Content-Type: application/json
X-Api-Key: {your-api-key}
```

**请求体**：
```json
{
  "groupId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
  "monitoredObjectId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
  "deviceId": "device_001",
  "collectedAt": "2026-06-03T10:30:00Z",
  "durationSeconds": "30.5",
  "processedFilePath": "/audio/processed/device_001_20260603103000.wav",
  "resultLabel": "风扇轴承摩擦",
  "score": 0.92
}
```

**字段说明**：
| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| groupId | Guid | 是 | 采集批次 ID（与上传时的 groupId 对应） |
| monitoredObjectId | Guid | 是 | 监测对象 ID |
| deviceId | string | 是 | 设备 ID |
| collectedAt | DateTime | 是 | 采集时间 |
| durationSeconds | string | 是 | 音频时长（秒） |
| processedFilePath | string | 是 | 处理后音频路径 |
| resultLabel | string | 是 | 识别结果标签 |
| score | number | 是 | 相似度分数（0-1） |

**响应**：
```json
{
  "success": true,
  "message": "Result saved"
}
```

**业务逻辑**：
1. 查找或创建 `vp_device_audio_record` 记录
2. 更新识别结果和音频路径
3. 连续 5 次异常才判定为异常（减少误报）
4. 更新监测对象状态（正常/异常）
5. 如为异常，自动创建告警记录

**cURL 示例**：
```bash
curl -X POST "http://localhost:19001/api/app/voiceprint/result" \
  -H "Content-Type: application/json" \
  -H "X-Api-Key: your-secret-api-key" \
  -d '{
    "groupId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
    "monitoredObjectId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
    "deviceId": "device_001",
    "collectedAt": "2026-06-03T10:30:00Z",
    "durationSeconds": "30.5",
    "processedFilePath": "/audio/processed/device_001_20260603103000.wav",
    "resultLabel": "风扇轴承摩擦",
    "score": 0.92
  }'
```

**C# 示例**：
```csharp
using var httpClient = new HttpClient();
httpClient.DefaultRequestHeaders.Add("X-Api-Key", "your-secret-api-key");

var payload = new
{
    groupId = Guid.Parse("3fa85f64-5717-4562-b3fc-2c963f66afa6"),
    monitoredObjectId = Guid.Parse("3fa85f64-5717-4562-b3fc-2c963f66afa6"),
    deviceId = "device_001",
    collectedAt = DateTime.UtcNow,
    durationSeconds = "30.5",
    processedFilePath = "/audio/processed/device_001_20260603103000.wav",
    resultLabel = "风扇轴承摩擦",
    score = 0.92
};

var response = await httpClient.PostAsJsonAsync(
    "http://localhost:19001/api/app/voiceprint/result",
    payload);

var result = await response.Content.ReadFromJsonAsync<VoiceprintResultUploadResponseDto>();
```

**注意事项**：
- 此接口允许匿名访问（`[AllowAnonymous]`），但需要 API Key 认证
- 异常类型会自动规范化（支持别名映射）
- 连续异常判定逻辑减少误报
- 识别标签支持中英文自动映射

---

### 2. 触发清理任务

**接口路径**：`POST /api/app/voiceprint/cleanup/trigger`

**功能说明**：手动触发已处理音频的清理任务（Hangfire 后台任务）。

**调用方**：运维人员、调试脚本

**请求头**：
```
X-Api-Key: {your-api-key}
```

**请求体**：无

**响应**：
```json
true
```

**业务逻辑**：
- 调用 `RecurringJob.Trigger("voiceprint-processed-audio-cleanup")`
- 立即执行清理任务，不等待定时调度

**清理任务详情**：参考 [VoiceprintProcessedCleanupJob](../../../Hangfire/System/VoiceprintProcessedCleanupJob.md)

**cURL 示例**：
```bash
curl -X POST "http://localhost:19001/api/app/voiceprint/cleanup/trigger" \
  -H "X-Api-Key: your-secret-api-key"
```

**C# 示例**：
```csharp
using var httpClient = new HttpClient();
httpClient.DefaultRequestHeaders.Add("X-Api-Key", "your-secret-api-key");

var response = await httpClient.PostAsync(
    "http://localhost:19001/api/app/voiceprint/cleanup/trigger",
    null);

var result = await response.Content.ReadFromJsonAsync<bool>();
// result = true
```

**注意事项**：
- 此接口为临时调试用途
- 生产环境建议依赖定时任务，无需手动触发
- 清理规则在 `appsettings.json` 中配置

---

### 3. 按分组修复音频

**接口路径**：`POST /api/app/voiceprint/processed-audios/repair-by-group`

**功能说明**：补齐指定设备在指定年月缺失的处理后音频文件，使用同 groupId 下源设备的音频复制。

**调用方**：运维脚本

**请求头**：
```
Content-Type: application/x-www-form-urlencoded
X-Api-Key: {your-api-key}
```

**请求参数**：
```
deviceId=device_001
&sourceDeviceId=device_002
&yearMonth=2026-06
```

**字段说明**：
| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| deviceId | string | 是 | 目标设备 ID（需要补齐的设备） |
| sourceDeviceId | string | 是 | 源设备 ID（音频完整的设备） |
| yearMonth | string | 是 | 年月（格式：yyyy-MM） |

**响应**：
```json
{
  "success": true,
  "deviceId": "device_001",
  "sourceDeviceId": "device_002",
  "yearMonth": "2026-06",
  "total": 100,
  "missing": 15,
  "repaired": 12,
  "skippedNoSource": 3,
  "failed": 0
}
```

**业务逻辑**：
1. 查询目标设备在指定年月的所有记录
2. 检查 `ProcessedFilePath` 对应的文件是否存在
3. 对于缺失文件，查找同 groupId 下源设备的音频
4. 复制源音频文件并重命名为目标文件名
5. 更新数据库记录的 `ProcessedFilePath`

**cURL 示例**：
```bash
curl -X POST "http://localhost:19001/api/app/voiceprint/processed-audios/repair-by-group" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -H "X-Api-Key: your-secret-api-key" \
  -d "deviceId=device_001&sourceDeviceId=device_002&yearMonth=2026-06"
```

**C# 示例**：
```csharp
using var httpClient = new HttpClient();
httpClient.DefaultRequestHeaders.Add("X-Api-Key", "your-secret-api-key");

var formData = new Dictionary<string, string>
{
    { "deviceId", "device_001" },
    { "sourceDeviceId", "device_002" },
    { "yearMonth", "2026-06" }
};

var content = new FormUrlEncodedContent(formData);
var response = await httpClient.PostAsync(
    "http://localhost:19001/api/app/voiceprint/processed-audios/repair-by-group",
    content);

var result = await response.Content.ReadFromJsonAsync<object>();
```

**注意事项**：
- `sourceDeviceId` 不能与 `deviceId` 相同
- 只修复同一 `groupId` 下的音频
- 如果源设备音频也不存在，则跳过该记录
- 文件命名规则：`{deviceId}_{timestamp}.wav`

---

### 4. 手动取消采集

**接口路径**：`POST /api/app/voiceprint/capture/manual-cancel`

**功能说明**：终止当前手动采集批次，恢复原定时采集流程。

**调用方**：运维人员、前端（通过算法切换接口间接调用）

**请求头**：
```
X-Api-Key: {your-api-key}
```

**请求体**：无

**响应**：
```json
{
  "success": true,
  "restored": true,
  "cancelledGroupId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
  "message": "已终止手动批次并恢复原流程"
}
```

**业务逻辑**：
1. 调用 `VoiceprintCaptureRuntimeStateService.CancelManualCapture()`
2. 清除当前运行的手动采集批次状态
3. 恢复定时采集调度

**相关接口**：
- `/voiceprint/algorithm/switch` - 切换算法时也会触发此操作

**cURL 示例**：
```bash
curl -X POST "http://localhost:19001/api/app/voiceprint/capture/manual-cancel" \
  -H "X-Api-Key: your-secret-api-key"
```

**C# 示例**：
```csharp
using var httpClient = new HttpClient();
httpClient.DefaultRequestHeaders.Add("X-Api-Key", "your-secret-api-key");

var response = await httpClient.PostAsync(
    "http://localhost:19001/api/app/voiceprint/capture/manual-cancel",
    null);

var result = await response.Content.ReadFromJsonAsync<VoiceprintManualCancelResultDto>();
```

**注意事项**：
- 如果当前无手动批次运行，返回 `restored: false`
- 取消后不会自动触发新的采集
- 建议通过 `/voiceprint/algorithm/switch` 间接使用

---

### 5. 模拟告警

**接口路径**：`POST /api/app/voiceprint/alarms/simulate`

**功能说明**：模拟生成声纹告警数据，用于测试告警功能。

**调用方**：测试脚本、开发调试

**请求头**：
```
X-Api-Key: {your-api-key}
```

**请求体**：无

**响应**：
```json
{
  "success": true,
  "table": "vp_device_audio_record",
  "count": 2,
  "items": [
    {
      "recordId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
      "monitoredObjectId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
      "anomalyType": "夹件螺栓松动",
      "isNormal": false,
      "updatedAt": "2026-06-03T10:30:00Z"
    }
  ]
}
```

**业务逻辑**：
1. 获取最新的 2 条声纹记录
2. 随机选择故障类型（从预设列表中）
3. 更新记录的 `AnomalyType` 和 `IsNormal`
4. 更新监测对象状态
5. 自动创建告警记录

**故障类型列表**：
- 夹件螺栓松动
- 焊渣异物
- 风扇轴承摩擦
- 局部放电
- 铁芯松动50%励磁

**cURL 示例**：
```bash
curl -X POST "http://localhost:19001/api/app/voiceprint/alarms/simulate" \
  -H "X-Api-Key: your-secret-api-key"
```

**C# 示例**：
```csharp
using var httpClient = new HttpClient();
httpClient.DefaultRequestHeaders.Add("X-Api-Key", "your-secret-api-key");

var response = await httpClient.PostAsync(
    "http://localhost:19001/api/app/voiceprint/alarms/simulate",
    null);

var result = await response.Content.ReadFromJsonAsync<object>();
```

**注意事项**：
- 此接口仅用于测试环境
- 会修改真实数据，请勿在生产环境使用
- 会触发真实的告警推送和状态更新

---

### 6. 批量删除告警

**接口路径**：`DELETE /api/app/voiceprint/alarms/delete-by-time`

**功能说明**：删除指定时间段内的声纹告警数据。

**调用方**：运维脚本

**请求头**：
```
X-Api-Key: {your-api-key}
```

**请求参数**：
```
startTime=2026-06-01T00:00:00Z
&endTime=2026-06-02T00:00:00Z
```

**字段说明**：
| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| startTime | DateTime | 是 | 开始时间（包含） |
| endTime | DateTime | 是 | 结束时间（不包含） |

**响应**：
```json
{
  "success": true,
  "table": "vp_voiceprint_alarm",
  "startTime": "2026-06-01T00:00:00Z",
  "endTime": "2026-06-02T00:00:00Z",
  "deleted": 25
}
```

**cURL 示例**：
```bash
curl -X DELETE "http://localhost:19001/api/app/voiceprint/alarms/delete-by-time?startTime=2026-06-01T00:00:00Z&endTime=2026-06-02T00:00:00Z" \
  -H "X-Api-Key: your-secret-api-key"
```

**C# 示例**：
```csharp
using var httpClient = new HttpClient();
httpClient.DefaultRequestHeaders.Add("X-Api-Key", "your-secret-api-key");

var url = "http://localhost:19001/api/app/voiceprint/alarms/delete-by-time"
    + "?startTime=2026-06-01T00:00:00Z"
    + "&endTime=2026-06-02T00:00:00Z";

var response = await httpClient.DeleteAsync(url);

var result = await response.Content.ReadFromJsonAsync<object>();
```

**注意事项**：
- 此操作不可逆，请谨慎使用
- 只删除告警记录表数据，不影响音频记录
- `startTime` 必须 < `endTime`

---

### 7. 修复测试音频类型

**接口路径**：`POST /api/app/voiceprint/device-audios/fix-test-audio-anomaly-type`

**功能说明**：批量将 `anomaly_type` 为"测试音频"或"线圈完全松动励磁"的记录更新为"运行正常"。

**调用方**：运维脚本、数据修复

**请求头**：
```
X-Api-Key: {your-api-key}
```

**请求体**：无

**响应**：
```json
{
  "success": true,
  "table": "vp_device_audio_record",
  "fromTypes": ["测试音频", "线圈完全松动励磁"],
  "toType": "运行正常",
  "affected": 150
}
```

**业务逻辑**：
```csharp
var fromTypes = new[] { "测试音频", "线圈完全松动励磁" };
const string toType = "运行正常";

UPDATE vp_device_audio_record
SET anomaly_type = '运行正常',
    is_normal = true
WHERE anomaly_type IN ('测试音频', '线圈完全松动励磁')
```

**cURL 示例**：
```bash
curl -X POST "http://localhost:19001/api/app/voiceprint/device-audios/fix-test-audio-anomaly-type" \
  -H "X-Api-Key: your-secret-api-key"
```

**C# 示例**：
```csharp
using var httpClient = new HttpClient();
httpClient.DefaultRequestHeaders.Add("X-Api-Key", "your-secret-api-key");

var response = await httpClient.PostAsync(
    "http://localhost:19001/api/app/voiceprint/device-audios/fix-test-audio-anomaly-type",
    null);

var result = await response.Content.ReadFromJsonAsync<object>();
```

**注意事项**：
- 此操作不可逆，建议先备份数据
- 会同时更新 `is_normal` 字段为 `true`
- 影响所有符合条件的历史记录

---

## 相关接口（其他文档）

### 算法切换接口

**接口路径**：`POST /api/app/voiceprint/algorithm/switch`

**说明**：此接口在前端使用，详细文档请参考：
- [前端接口文档](../../../../frontend-api-interfaces.md) - 接口 #14
- [VoiceprintAPI.md](../Voiceprint/VoiceprintAPI.md) - 算法管理章节

**主要功能**：
- 切换算法模式（手动增强 / 原流程）
- 触发 60 秒手动采集
- 自动调用 `/voiceprint/capture/manual-cancel`（切回原流程时）

---

## 数据结构

### VoiceprintResultUploadInput（识别结果入参）

```csharp
public class VoiceprintResultUploadInput
{
    public Guid GroupId { get; set; }
    public Guid MonitoredObjectId { get; set; }
    public string DeviceId { get; set; }
    public DateTime CollectedAt { get; set; }
    public string DurationSeconds { get; set; }
    public string ProcessedFilePath { get; set; }
    public string ResultLabel { get; set; }
    public double Score { get; set; }
}
```

### VoiceprintResultUploadResponseDto（识别结果响应）

```csharp
public class VoiceprintResultUploadResponseDto
{
    public bool Success { get; set; }
    public string Message { get; set; }
}
```

### VoiceprintManualCancelResultDto（手动取消响应）

```csharp
public class VoiceprintManualCancelResultDto
{
    public bool Success { get; set; }
    public bool Restored { get; set; }
    public Guid? CancelledGroupId { get; set; }
    public string Message { get; set; }
}
```

---

## 错误处理

### 错误响应格式

```json
{
  "error": {
    "code": "ErrorCode",
    "message": "错误描述",
    "details": "详细错误信息"
  }
}
```

### 常见错误

| 错误码 | 说明 | 解决方法 |
|-------|------|---------|
| Invalid api key | API Key 无效 | 检查 `X-Api-Key` 请求头 |
| groupId 无效 | groupId 为空或格式错误 | 确保传入有效的 Guid |
| deviceId 不能为空 | deviceId 参数缺失 | 检查请求参数 |
| 时间范围无效 | startTime >= endTime | 调整时间范围 |
| yearMonth 格式无效 | 非 yyyy-MM 格式 | 使用正确格式，如 "2026-06" |

---

## 运维建议

### 1. API Key 管理

- 生产环境必须配置强随机 API Key
- 定期轮换 API Key
- 不要在前端代码中暴露 API Key

### 2. 接口监控

- 监控 `/voiceprint/result` 调用频率
- 记录清理任务执行结果
- 追踪音频修复操作

### 3. 数据备份

- 执行删除操作前先备份数据
- 定期备份 `vp_device_audio_record` 和 `vp_voiceprint_alarm` 表
- 保留至少 30 天的历史数据

### 4. 定期维护

- 每月检查音频文件完整性
- 定期清理过期的测试数据
- 验证备份恢复流程

---

## 相关文档

- [前端接口文档](../../../../frontend-api-interfaces.md) - 23 个前端专用接口
- [VoiceprintAPI.md](../Voiceprint/VoiceprintAPI.md) - 声纹模块前端 API
- [VoiceprintCaptureJob.md](../../../Hangfire/VoiceprintCaptureJob.md) - 声纹采集任务
- [VoiceprintProcessedCleanupJob.md](../../../Hangfire/System/VoiceprintProcessedCleanupJob.md) - 音频清理任务
- [CLAUDE.md](../../../../CLAUDE.md) - 项目架构说明

---

**最后更新**：2026-06-03  
**版本**：v1.0.0  
**维护者**：Backend Team
