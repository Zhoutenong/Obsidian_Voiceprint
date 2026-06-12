# 文档 Review 修复记录

> **修复日期**：2026-06-12
> **依据**：`文档Review报告.md` Phase 1 & Phase 2 修复计划

---

## Phase 1 — 紧急修复（P0）✅

### 1.1 INDEX.md 断链修复（8 处）

| 行号 | 原断链 | 修复后 |
|------|--------|--------|
| 35 | `[[文档补齐计划2026.md]]` | `[[文档补齐计划2026]]` |
| 445 | `[[Modules/ast-intellisub/设备管理服务]]` | `[[Modules/ast-intellisub/DeviceService]]` |
| 595 | `[[待补文档清单]]` | `[[DocumentationPlan]]` |
| 335 | `[[../../../voiceprint/00-系统架构与功能实现.md]]` | `[[../../voiceprint/00-系统架构与功能实现]]` |
| 336 | `[[../../../voiceprint/02-python-voiceprint-service方案.md]]` | `[[../../voiceprint/02-python-voiceprint-service方案]]` |
| 337 | `[[../../../voiceprint/03-raspberry-pi-agent方案.md]]` | `[[../../voiceprint/03-raspberry-pi-agent方案]]` |
| 338 | `[[../../../data-flow-diagrams/]]` | 已移除（不存在） |
| 339 | `[[../../../frontend-api-interfaces.md]]` | 已移除（不存在） |

### 1.2 快速导航.md 重写

- 原文件 97.7% 断链（125/128 个链接指向不存在的目录/文件）
- 已完全重写，所有链接指向实际存在的文档文件

### 1.3 VoiceprintAPI.md 修正

- 移除 4 个不存在的端点：`GET /audios`、`GET /audios/{id}`、`POST /audios/download`、`GET /test-audios/{audioId}/recognition-result`
- 修正 `POST /test-audios/import` 参数：仅接受 `IFormFile file`
- 修正 `GET /standard-audios` 返回值：`IReadOnlyList<VoiceprintStandardAudioGroupDto>`（无分页）
- 修正 `POST /standard-audios` 返回值：`Task`（void），参数为 `anomalyType`、`file`、`audioName`
- 补充 6 个缺失端点：`DELETE /alarms/delete-by-time`、`POST /device-audios/fix-test-audio-anomaly-type`、`POST /alarms/simulate`、`POST /capture/manual-cancel`、`POST /processed-audios/repair-by-group`、`POST /cleanup/trigger`
- 修正 `POST /algorithm/switch` 参数：`targetMode: int`

### 1.4 MQTT 方法名统一

- `PublishVoiceprintCaptureCommandAsync` → `PublishVoiceprintCommandAsync`
- 修复文件：`VoiceprintCaptureJob.md`（2 处）

---

## Phase 2 — 准确性修复（P1）✅

### 2.1 Agent 配置字段修正

- `DeviceId`（单数）→ `DeviceIds`（`List<string>`，复数）
- 补充 `RetryLimit` 字段（默认 3）
- 修复文件：`VoiceprintCaptureJob.md`

### 2.2 Cron 默认值统一

- JSON 示例 `*/10` → `*/5`（与源码默认值一致）
- 修复文件：`VoiceprintCaptureJob.md`

### 2.3 DeviceStatusEventHandler 并发描述修正

- "使用并发字典处理多个设备" → "顺序执行更新，保持在同一个 UOW 上下文中"
- 修复文件：`EventDrivenPipeline.md`

### 2.4 补充 NormalDataProcessedHandler 3 个重试方法

- 补充 `ProcessSingleContextWithRetryAsync`(100ms)、`UpdateMonitoredObjectItemRelStatusAsync`(50ms)、`UpdateMonitoredObjectStatusWithRetryAsync`(50ms)
- 修复文件：`EventDrivenPipeline.md`

### 2.5 补充 Iec61850 Float 数据类型

- 补充 `Iec61850DataType.Float => (float)alarmValue`
- 修复文件：`EventDrivenPipeline.md`

### 2.6 AlarmRecordItemCreatedHandler 级联更新补充

- 补充 "递归更新父设备状态" 步骤
- 修复文件：`EventDrivenPipeline.md`

### 2.7 内部 API 路径前缀修正

- `/api/app/voiceprint-audio/` → `/api/app/voiceprint/`
- 修复文件：`README.md`、`VoiceprintAudioAppService.md`、`VoiceprintAudioUploadAPI.md`

### 2.8 AlarmConditionEvaluationManager 验证步骤补充

- 补充 PointId、DeviceId、SensorKey、Property 四步验证
- 修复文件：`AlarmConditionEvaluationManager.md`

---

## 附加修复

### 断链修复

- `[[设备管理服务]]` → `[[Modules/ast-intellisub/DeviceService]]`（6 个文件）
- `[[待补文档清单]]` → 自引用（`DocumentationPlan.md`）
- `../../../voiceprint/` → `../../voiceprint/`（多个文件）
- `.md` 后缀从 wiki-style 链接中移除
- `VoiceprintCaptureRuntimeStateService.md` 路径修正
- `NvrService.md`、`PTZPresetService.md` 相关链接修正

### 格式修复

- 三重方括号 `[[[` → 双重方括号 `[[`
- `Related Documents` 链接修正为实际存在的文档

---

## 修复文件清单

| 文件 | 修复类型 |
|------|---------|
| `INDEX.md` | 断链修复 |
| `快速导航.md` | 完全重写 |
| `VoiceprintAPI.md` | 端点修正 + 补充 |
| `VoiceprintCaptureJob.md` | MQTT 方法名 + 配置字段 + Cron 默认值 |
| `EventDrivenPipeline.md` | 并发描述 + 重试方法 + Float 类型 + 级联更新 |
| `README.md` (ast-voiceprint) | API 路径 + 外部链接 + 日期 |
| `VoiceprintAudioAppService.md` | API 路径 + 链接修复 |
| `VoiceprintAudioUploadAPI.md` | API 路径 |
| `VoiceprintPortalAppService.md` | 链接修复 |
| `AlarmConditionEvaluationManager.md` | 验证步骤补充 |
| `ExternalDocsIndex.md` | 外部链接修复 |
| `DocumentationPlan.md` | 断链修复 |
| `MonitoredItemService.md` | 链接修复 |
| `MonitoredObjectService.md` | 链接修复 |
| `MonitoredObjectTypeService.md` | 链接修复 |
| `MonitoredObjectAttrService.md` | 链接修复 |
| `SensorService.md` | 链接修复 |
| `MqttMessageBackgroundService.md` | 链接修复 |
| `AssetsAPI.md` | 链接修复 |
| `NvrService.md` | 链接修复 |
| `PTZPresetService.md` | 链接修复 |
