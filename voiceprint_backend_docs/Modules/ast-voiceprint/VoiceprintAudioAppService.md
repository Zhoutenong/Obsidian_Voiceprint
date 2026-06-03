# VoiceprintAudioAppService - 音频处理服务

## 概述

**VoiceprintAudioAppService** 是 ast-voiceprint 模块的核心服务，负责处理树莓派边缘采集端的音频上传和 Python 算法服务的识别结果回写。该服务是音频数据流转的中心枢纽，连接边缘采集、算法识别和数据持久化。

**源码位置：** `module/ast-voiceprint/Ast.Voiceprint.Application/Services/VoiceprintAudioAppService.cs`

**API 基础路径：** `/api/app/voiceprint-audio/`

---

## 核心功能

### 1. 音频上传 (`UploadAsync`)

**端点：** `POST /api/app/voiceprint-audio/upload`

**职责：** 接收树莓派采集端上传的原始音频文件

**请求参数：**
```csharp
public class VoiceprintAudioUploadInput
{
    public Guid GroupId { get; set; }          // 采集批次 ID
    public string DeviceId { get; set; }       // 设备标识（如 "pi-01"）
    public IFormFile File { get; set; }         // 音频文件（WAV 格式）
    public DateTime DateTime { get; set; }      // 采集时间（可选）
}
```

**返回结果：**
```csharp
public class VoiceprintAudioUploadResultDto
{
    public string FileName { get; set; }       // 保存的文件名
    public long FileSize { get; set; }          // 文件大小（字节）
    public string FilePath { get; set; }        // 公共访问路径
    public DateTime UploadTime { get; set; }   // 上传时间
    public string ContentType { get; set; }     // 内容类型
    public Guid GroupId { get; set; }          // 批次 ID
    public string DeviceId { get; set; }        // 设备 ID
}
```

**业务流程：**
1. **API 密钥验证** - 通过 `X-Api-Key` 请求头验证调用权限
2. **输入验证** - 检查文件非空、GroupId 有效、DeviceId 非空
3. **超时恢复** - 检查手动采集是否超时，自动恢复定时任务
4. **文件保存** - 保存到 `wwwroot/audio/unprocessed/{groupId}/` 目录
5. **状态更新** - 通过 `VoiceprintCaptureRuntimeStateService` 标记设备上传完成
6. **批次检测** - 判断是否所有设备都已上传完成

**配置项：**
```json
{
  "VoiceprintStorage": {
    "Root": "wwwroot/audio",
    "PendingFolderName": "unprocessed",
    "ApiKey": "your-secret-api-key"
  }
}
```

**相关文档：**
- [[../../Pipelines/Audio/音频采集与处理流程.md|音频采集与处理流程]]
- [[../../Hangfire/Voiceprint/VoiceprintCaptureJob.md|VoiceprintCaptureJob - 采集任务调度]]

---

### 2. 算法结果回写 (`SubmitResultAsync`)

**端点：** `POST /api/app/voiceprint/result`

**职责：** 接收 Python 算法服务的声纹识别结果并持久化

**请求参数：**
```csharp
public class VoiceprintResultUploadInput
{
    public Guid GroupId { get; set; }              // 采集批次 ID
    public Guid MonitoredObjectId { get; set; }    // 监控对象 ID（设备）
    public string DeviceId { get; set; }           // 设备标识
    public DateTime CollectedAt { get; set; }      // 采集时间
    public string DurationSeconds { get; set; }     // 音频时长（秒）
    public string ProcessedFilePath { get; set; }   // 处理后音频路径
    public string resultLabel { get; set; }         // 识别结果标签
    public double Score { get; set; }               // 置信度（0-1）
}
```

**返回结果：**
```csharp
public class VoiceprintResultUploadResponseDto
{
    public bool Success { get; set; }     // 处理成功标志
    public string Message { get; set; }   // 返回消息
}
```

**业务流程：**
1. **查找或创建记录** - 根据 `GroupId` 和 `MonitoredObjectId` 查询或创建音频记录
2. **标签规范化** - 将算法返回的标签规范化为标准异常类型
3. **连续异常抑制** - 检查最近 5 条记录，只有全部异常才判定为异常（减少误报）
4. **更新记录字段**：
   - `CollectedAt` - 采集时间
   - `DurationSeconds` - 音频时长
   - `AnomalyType` - 规范化后的异常类型
   - `IsNormal` - 是否正常（`AnomalyType == "运行正常"`）
   - `Score` - 置信度
   - `ProcessedFilePath` - 处理后音频路径
5. **设备状态更新** - 根据识别结果更新设备状态（正常/异常）
6. **告警创建** - 如果识别为异常，创建告警记录

**异常类型映射：**
```csharp
private static readonly string[] _voiceprintCategories =
{
    "运行正常",
    "夹件螺栓松动",
    "风扇轴承摩擦",
    "局部放电",
    "励磁",
    "连接螺栓松动",
    "焊渣异物",
    "铁芯松动50%励磁",
    "线圈完全松动励磁"
};
```

**连续异常抑制逻辑：**
```csharp
// 只有最近 5 条全部异常时才判定为异常
var prevRecords = await _deviceAudioRecordRepository._DbQueryable
    .Where(x => x.MonitoredObjectId == input.MonitoredObjectId && x.Id != record.Id)
    .OrderBy(x => x.CollectedAt, OrderByType.Desc)
    .Select(x => new { x.RealAnomalyType })
    .Take(5)
    .ToListAsync();

var hasTwoPreviousAbnormal = prevRecords.Count == 5 &&
    prevRecords.All(x => !string.Equals(
        NormalizeLabel(x.RealAnomalyType),
        "运行正常",
        StringComparison.OrdinalIgnoreCase));

if (!hasTwoPreviousAbnormal)
{
    normalizedLabel = "运行正常";
}
```

**配置项：**
```json
{
  "VoiceprintRecognition": {
    "ContinuousAnomalyThreshold": 5,
    "EnableContinuousAnomalySuppression": true
  }
}
```

**相关文档：**
- [[../../Pipelines/Audio/声纹识别与告警流程.md|声纹识别与告警流程]]
- [[../../../voiceprint/02-python-voiceprint-service方案.md|Python 声纹识别服务方案]]

---

### 3. 单音频报告生成 (`GenerateReportFromAudioAsync`)

**端点：** `POST /api/app/voiceprint/reports/generate-from-audio`

**职责：** 根据上传的音频文件和识别类型生成单次分析报表

**请求参数：**
```csharp
public class VoiceprintAudioReportGenerateInput
{
    public IFormFile AudioFile { get; set; }        // 音频文件
    public string AudioFileName { get; set; }       // 原始文件名（可选）
    public string DiagnosisType { get; set; }       // 诊断类型
    public string ResultFileName { get; set; }      // 报告文件名（可选）
}
```

**返回结果：**
```csharp
public class VoiceprintAudioReportGenerateResultDto
{
    public string DownloadPath { get; set; }         // 下载路径
    public string DiagnosisType { get; set; }        // 最终诊断类型
    public string AudioMd5 { get; set; }             // 音频 MD5
    public string TemplateFileName { get; set; }     // 使用的模板文件名
    public string TargetAudioPath { get; set; }      // 目标音频路径
}
```

**业务流程：**
1. **标准库匹配** - 计算音频 MD5，查询标准库是否已有相同音频
2. **诊断类型确定** - 如果命中标准库，使用标准库的类型；否则使用输入的类型
3. **标准音频查询** - 查询同类型的 Top 3 标准音频
4. **频域图生成**：
   - 生成目标音频的频域图（宽图版本）
   - 生成目标音频的频域图（普通版本）
   - 生成 Top 3 标准音频的频域图
   - 生成目标音频与 Top 1 标准音频的对比图
5. **模板选择** - 根据诊断类型选择对应的报告模板
6. **报告填充** - 调用报告服务填充模板并生成 DOCX 文件
7. **临时文件清理** - 删除上传的临时音频文件

**模板映射规则：**
```csharp
private static readonly Dictionary<string, string> _reportTemplateMap = new(StringComparer.OrdinalIgnoreCase)
{
    ["运行正常"] = "运行正常.docx",
    ["风扇轴承摩擦"] = "风扇轴承摩擦.docx",
    ["局部放电"] = "局部放电.docx",
    ["铁芯松动50%励磁"] = "铁芯松动50%励磁.docx",
    ["连接螺栓松动"] = "连接螺栓松动.docx",
    ["线圈完全松动励磁"] = "线圈完全松动励磁.docx",
    ["焊渣异物"] = "焊渣异物.docx",
    ["夹件螺栓松动"] = "夹件螺栓松动.docx",
    ["励磁"] = "励磁.docx"
};
```

**频域图生成技术：**
- 使用 **NAudio** 读取音频文件
- 应用 **汉宁窗（Hanning Window）** 减少频谱泄漏
- 执行 **快速傅里叶变换（FFT）** 计算频域幅度
- 归一化到 0-255 范围便于图表显示
- 调用独立的频域图生成服务（HTTP API）

**配置项：**
```json
{
  "VoiceprintReport": {
    "TemplateBasePath": "wwwroot/report_templates",
    "OutputBasePath": "wwwroot/reports",
    "ImageWidth": 800,
    "ApiUrl": "http://localhost:5000/api/report/generate",
    "ChartApiUrl": "http://localhost:3000"
  }
}
```

**相关文档：**
- [[../../Pipelines/Audio/报告生成流程.md|报告生成流程]]
- [[../../../voiceprint/分析报表需求文档.md|分析报表需求文档]]

---

### 4. 算法切换 (`SwitchAlgorithmAsync`)

**端点：** `POST /api/app/voiceprint/algorithm/switch`

**职责：** 切换算法模式（原始/增强）并触发相应的采集流程

**请求参数：**
```csharp
public class VoiceprintAlgorithmSwitchInput
{
    public int TargetMode { get; set; }  // 目标模式：0=原始, 1=增强
}
```

**返回结果：**
```csharp
public class VoiceprintAlgorithmSwitchResultDto
{
    public bool Success { get; set; }              // 操作成功标志
    public string ActiveMode { get; set; }         // 当前激活模式
    public string ActiveAlgorithmPath { get; set; } // 当前算法路径
    public int CaptureDurationSeconds { get; set; } // 采集时长（秒）
    public Guid? ManualGroupId { get; set; }       // 手动采集批次 ID
    public string Message { get; set; }            // 返回消息
}
```

**业务流程：**

#### 切换到原始模式 (`TargetMode = 0`)
1. **调用算法服务** - 通知 Python 服务切换到原始算法
2. **取消手动采集** - 终止当前运行的手动采集批次
3. **恢复定时任务** - Hangfire 定时任务恢复正常调度

#### 切换到增强模式 (`TargetMode = 1`)
1. **调用算法服务** - 通知 Python 服务切换到增强算法
2. **取消现有批次** - 如果已有手动采集在运行，先取消
3. **启动手动采集**：
   - 生成新的采集批次 `groupId`
   - 设置超时时间（默认 10 分钟）
   - 通过 MQTT 下发采集指令（60 秒时长）
4. **返回批次信息** - 返回手动采集批次 ID 供前端跟踪

**算法服务接口：**
```http
POST {AlgorithmSwitchApiUrl}
Content-Type: application/json

{
  "targetMode": "original" | "enhanced"
}
```

**MQTT 采集命令格式：**
```json
{
  "command": "voiceprint-capture",
  "data": {
    "groupId": "5b7e3d6f-...",
    "durationSeconds": 60,
    "dateTime": "2025-05-22T03:30:00+00:00",
    "retryLimit": 3
  }
}
```

**配置项：**
```json
{
  "VoiceprintRecognition": {
    "AlgorithmSwitchApiUrl": "http://localhost:8000/api/algorithm/switch",
    "AlgorithmSwitchTimeoutSeconds": 10
  },
  "VoiceprintCaptureJob": {
    "DurationSeconds": 10,
    "ManualCaptureTimeoutMinutes": 10,
    "Agents": [
      {
        "GatewayId": "gateway-guid-here",
        "DeviceIds": ["pi-01", "pi-02"],
        "RetryLimit": 3
      }
    ]
  }
}
```

**相关文档：**
- [[../../../voiceprint/需求变更-算法切换与1分钟分段识别.md|需求变更-算法切换与1分钟分段识别]]
- [[../../Hangfire/Voiceprint/VoiceprintCaptureRuntimeStateService.md|运行时状态管理]]

---

### 5. 取消手动采集 (`CancelManualCaptureAsync`)

**端点：** `POST /api/app/voiceprint/capture/manual-cancel`

**职责：** 手动终止当前手动采集批次，恢复定时流程

**返回结果：**
```csharp
public class VoiceprintManualCancelResultDto
{
    public bool Success { get; set; }         // 操作成功标志
    public bool Restored { get; set; }        // 是否已恢复
    public Guid? CancelledGroupId { get; set; } // 被取消的批次 ID
    public string Message { get; set; }       // 返回消息
}
```

**业务流程：**
1. **检查运行状态** - 判断当前是否有手动采集在运行
2. **取消手动批次** - 清除运行时状态中的批次信息
3. **恢复定时任务** - Hangfire 定时任务恢复自动调度
4. **自动切回算法** - 如果当前是增强模式，自动切回原始算法

**相关文档：**
- [[../../Hangfire/Voiceprint/VoiceprintCaptureRuntimeStateService.md|运行时状态管理]]

---

## 维护与调试接口

### 手动触发清理任务 (`TriggerProcessedCleanupAsync`)

**端点：** `POST /api/app/voiceprint/cleanup/trigger`

**职责：** 手动触发 Hangfire 已处理音频清理任务（调试用）

### 补齐缺失音频 (`RepairMissingProcessedAudiosAsync`)

**端点：** `POST /api/app/voiceprint/processed-audios/repair-by-group`

**职责：** 补齐指定设备在指定年月缺失的处理后音频

**请求参数：**
```csharp
public class RepairMissingAudiosInput
{
    public string DeviceId { get; set; }       // 目标设备 ID
    public string SourceDeviceId { get; set; } // 源设备 ID
    public string YearMonth { get; set; }      // 年月（格式：yyyy-MM）
}
```

### 删除告警数据 (`DeleteAlarmsByTimeRangeAsync`)

**端点：** `DELETE /api/app/voiceprint/alarms/delete-by-time`

**职责：** 删除指定时间段内的声纹告警数据

### 修复测试音频类型 (`FixTestAudioAnomalyTypeAsync`)

**端点：** `POST /api/app/voiceprint/device-audios/fix-test-audio-anomaly-type`

**职责：** 批量修复 anomaly_type 字段（"测试音频" → "运行正常"）

### 模拟告警 (`SimulateLatestVoiceprintAlarmAsync`)

**端点：** `POST /api/app/voiceprint/alarms/simulate`

**职责：** 模拟一条最新声纹数据告警（测试用）

---

## 安全机制

### API 密钥验证

所有接口均通过 `X-Api-Key` 请求头进行密钥验证：

```csharp
private void ValidateApiKey()
{
    if (string.IsNullOrWhiteSpace(_storageOptions.ApiKey))
    {
        return;
    }

    var ctx = _httpContextAccessor.HttpContext;
    var keyFromHeader = ctx?.Request?.Headers["X-Api-Key"].ToString();
    if (!string.Equals(keyFromHeader, _storageOptions.ApiKey, StringComparison.Ordinal))
    {
        throw new AbpAuthorizationException("Invalid api key");
    }
}
```

### 文件大小限制

```csharp
[RequestSizeLimit(104_857_600)] // 100MB
[RequestFormLimits(MultipartBodyLengthLimit = 104_857_600)] // 100MB
```

---

## 外部依赖

### Python 声纹识别服务

**详细方案：** [[../../../voiceprint/02-python-voiceprint-service方案.md|Python 声纹识别服务方案]]

**算法切换接口：**
```
POST {VoiceprintRecognition:AlgorithmSwitchApiUrl}
Content-Type: application/json

{
  "targetMode": "original" | "enhanced"
}
```

### 报告生成服务

**配置节：** `VoiceprintReport:ApiUrl`

**接口约定：**
```http
POST {ApiUrl}
Content-Type: application/json

{
  "templateFileName": "运行正常.docx",
  "resultFileName": "声纹分析报表_20250622.docx",
  "imageWidth": 800,
  "imageWidths": {
    "__default__": 800,
    "target_frequency_image": 1600
  },
  "context": {
    "analysis_time": "2025-06-22 10:30:00",
    "target_file_name": "pi-01_20250622103000.wav",
    "diagnosis_type": "运行正常",
    ...
  }
}
```

### 频域图生成服务

**配置节：** `VoiceprintReport:ChartApiUrl`

**接口约定：**
```http
POST {ChartApiUrl}/waveform/frequency-domain-graph
Content-Type: application/json

{
  "width": 800,
  "height": 240,
  "xAxisData": [0, 10.5, 21.0, ...],
  "seriesData": [{
    "name": "现场",
    "value": [120.5, 135.2, ...]
  }],
  "xAxisName": "Hz",
  "yAxisName": "幅度"
}
```

---

## 数据库实体关联

### 主要操作实体

| 实体 | 表名 | 用途 |
|------|------|------|
| `VoiceprintDeviceAudioRecordEntity` | `vp_device_audio_record` | 设备声纹采集与识别记录 |
| `VoiceprintAlarmRecordEntity` | `vp_voiceprint_alarm` | 声纹告警记录 |
| `VoiceprintStandardAudioEntity` | `vp_standard_audio_library` | 标准音频库 |
| `MonitoredObjectAggregateRoot` | `monitored_object` | 监控对象（设备） |

### 实体关系

```
VoiceprintDeviceAudioRecordEntity (音频记录)
├── MonitoredObjectId → MonitoredObjectAggregateRoot (设备)
├── GroupId (采集批次)
└── 触发创建 → VoiceprintAlarmRecordEntity (告警)
    ├── MonitoredObjectId → MonitoredObjectAggregateRoot
    └── DeviceAudioRecordId → VoiceprintDeviceAudioRecordEntity
```

**详细实体文档：** [[../../../voiceprint/声纹数据库实体介绍文档.md|声纹数据库实体介绍文档]]

---

## 配置总览

### appsettings.json 完整配置

```json
{
  "VoiceprintStorage": {
    "Root": "wwwroot/audio",
    "PendingFolderName": "unprocessed",
    "ProcessedFolderName": "processed",
    "LogsFolderName": "logs",
    "StandardLibraryFolderName": "standard_library",
    "ApiKey": "your-secret-api-key"
  },
  "VoiceprintRecognition": {
    "AlgorithmSwitchApiUrl": "http://localhost:8000/api/algorithm/switch",
    "AlgorithmSwitchTimeoutSeconds": 10,
    "ContinuousAnomalyThreshold": 5,
    "EnableContinuousAnomalySuppression": true,
    "TestAudioApiUrl": "http://localhost:8000/api/test-recognize",
    "TestAudioDirectory": "tests/audio",
    "RequestTimeoutSeconds": 30
  },
  "VoiceprintReport": {
    "TemplateBasePath": "wwwroot/report_templates",
    "OutputBasePath": "wwwroot/reports",
    "TemplateFileName": "report_template.docx",
    "ImageWidth": 800,
    "ApiUrl": "http://localhost:5000/api/report/generate",
    "ChartApiUrl": "http://localhost:3000",
    "AudioBaseUrl": ""
  },
  "VoiceprintCaptureJob": {
    "Enabled": true,
    "CronExpression": "0 */5 * * * *",
    "DurationSeconds": 10,
    "ManualCaptureTimeoutMinutes": 10,
    "Agents": [
      {
        "GatewayId": "gateway-guid-here",
        "DeviceIds": ["pi-01", "pi-02"],
        "RetryLimit": 3
      }
    ]
  }
}
```

---

## 常见问题

### Q: 音频上传失败，返回 401 Unauthorized

**原因：** API 密钥验证失败

**解决方案：**
1. 检查请求头 `X-Api-Key` 是否正确
2. 确认配置节 `VoiceprintStorage:ApiKey` 与采集端配置一致
3. 查看服务日志中的验证错误信息

### Q: 算法切换后采集时长没有变化

**原因：** 切换到增强模式后，需要等待手动采集完成或超时

**解决方案：**
1. 检查 `VoiceprintCaptureRuntimeStateService` 状态
2. 手动调用 `/api/app/voiceprint/capture/manual-cancel` 取消当前批次
3. 确认 MQTT 采集命令中的 `durationSeconds` 参数

### Q: 报告生成失败，提示"音频文件不存在"

**原因：** 音频文件的公共路径与物理路径映射问题

**解决方案：**
1. 确认音频文件实际存在于 `wwwroot/audio/` 目录
2. 检查 `VoiceprintStorage:Root` 配置是否正确
3. 查看服务日志中的文件解析信息

### Q: 连续异常抑制导致真实异常被忽略

**原因：** 默认配置需要最近 5 条全部异常才判定为异常

**解决方案：**
1. 调整 `VoiceprintRecognition:ContinuousAnomalyThreshold` 配置
2. 禁用连续异常抑制：`EnableContinuousAnomalySuppression: false`
3. 根据实际场景调整阈值（推荐值：3-5）

---

## 相关文档

### 模块总览
- [[README.md|ast-voiceprint 模块总览]]

### 相关服务
- [[VoiceprintPortalAppService.md|VoiceprintPortalAppService - 前端门户服务]]
- [[../../Hangfire/Voiceprint/VoiceprintCaptureRuntimeStateService.md|运行时状态管理]]

### 流程文档
- [[../../Pipelines/Audio/音频采集与处理流程.md|音频采集与处理流程]]
- [[../../Pipelines/Audio/声纹识别与告警流程.md|声纹识别与告警流程]]
- [[../../Pipelines/Audio/报告生成流程.md|报告生成流程]]

### 外部集成
- [[../../../voiceprint/02-python-voiceprint-service方案.md|Python 声纹识别服务方案]]
- [[../../../voiceprint/03-raspberry-pi-agent方案.md|树莓派采集程序方案]]

---

**文档版本：** v1.0.0  
**最后更新：** 2025-06-18  
**维护者：** ast-voiceprint 模块团队
