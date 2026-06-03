# VoiceprintPortalAppService - 前端门户服务

## 概述

**VoiceprintPortalAppService** 是 ast-voiceprint 模块的前端门户 API 服务，为 Web 管理界面提供全方位的数据查询、统计分析和设备管理功能。该服务封装了复杂的数据库查询和业务逻辑，提供简洁的 RESTful 接口供前端调用。

**源码位置：** `module/ast-voiceprint/Ast.Voiceprint.Application/Services/VoiceprintPortalAppService.cs`

**API 基础路径：** `/api/app/voiceprint/`

**认证方式：** JWT Bearer Token（除特别标注 `[AllowAnonymous]` 的接口）

---

## 功能模块

### Dashboard - 首页总览

#### 1. 总览统计 (`GetOverviewAsync`)

**端点：** `GET /api/app/voiceprint/dashboard/overview`

**职责：** 提供系统运行总览数据

**返回结果：**
```csharp
public class VoiceprintDashboardOverviewDto
{
    public int TotalDays { get; set; }         // 总运行天数
    public int TotalCaptureCount { get; set; }  // 总采集批次次数
}
```

**统计逻辑：**
```sql
-- 最早记录时间
SELECT MIN(CollectedAt) FROM vp_device_audio_record

-- 总采集批次（去重 GroupId）
SELECT COUNT(DISTINCT GroupId) FROM vp_device_audio_record
```

---

#### 2. 指定日期巡视记录 (`GetDailyRecordsAsync`)

**端点：** `GET /api/app/voiceprint/dashboard/records`

**职责：** 获取指定日期的所有巡视记录（按批次分组）

**请求参数：**
```csharp
public DateTime date { get; set; }  // 目标日期（默认：今天）
```

**返回结果：**
```csharp
public class VoiceprintDashboardRecordDto
{
    public Guid GroupId { get; set; }       // 批次 ID
    public string Status { get; set; }      // 批次状态："normal" | "abnormal"
    public DateTime OccurredAt { get; set; } // 发生时间
}
```

**查询逻辑：**
```sql
SELECT GroupId, 
       SUM(CASE WHEN IsNormal = 0 THEN 1 ELSE 0 END) > 0 AS HasAbnormal,
       MIN(CollectedAt) AS OccurredAt
FROM vp_device_audio_record
WHERE CollectedAt >= @Start AND CollectedAt < @End
GROUP BY GroupId
```

**状态判定：**
- `normal` - 批次内所有设备均正常
- `abnormal` - 批次内至少有一台设备异常

---

#### 3. 巡视记录详情 (`GetRecordDetailsAsync`)

**端点：** `GET /api/app/voiceprint/dashboard/records/{groupId}`

**职责：** 获取指定批次的详细巡视记录

**返回结果：**
```csharp
public class VoiceprintRecordDetailDto
{
    public Guid RecordId { get; set; }          // 记录 ID
    public Guid GroupId { get; set; }           // 批次 ID
    public Guid MonitoredObjectId { get; set; } // 监控对象 ID
    public string DeviceName { get; set; }      // 设备名称
    public string DeviceTypeName { get; set; }   // 设备类型名称
    public string AudioPath { get; set; }       // 音频路径
    public string AnomalyType { get; set; }     // 异常类型
    public DateTime CollectedAt { get; set; }   // 采集时间
    public bool IsNormal { get; set; }          // 是否正常
}
```

**排序规则：**
1. 按 `CollectedAt` 降序（最新优先）
2. 按 `MonitoredObject.OrderNum` 升序（设备排序号）

---

#### 4. 系统总览状态 (`GetSystemStatusAsync`)

**端点：** `GET /api/app/voiceprint/dashboard/system-status`

**职责：** 提供电子沙盘/系统总览数据

**返回结果：**
```csharp
public class VoiceprintSystemStatusDto
{
    public int TotalDevices { get; set; }                 // 总设备数
    public int AbnormalDevices { get; set; }              // 异常设备数
    public IReadOnlyList<VoiceprintDeviceTypeStatDto> TypeStats { get; set; }  // 按类型统计
    public IReadOnlyList<VoiceprintDeviceSnapshotDto> Devices { get; set; }   // 设备快照列表
}

public class VoiceprintDeviceTypeStatDto
{
    public Guid? TypeId { get; set; }      // 类型 ID
    public string TypeName { get; set; }   // 类型名称
    public int Count { get; set; }         // 该类型设备数量
}

public class VoiceprintDeviceSnapshotDto
{
    public Guid Id { get; set; }            // 设备 ID
    public Guid? ParentId { get; set; }    // 父设备 ID
    public string Name { get; set; }        // 设备名称
    public string TypeName { get; set; }    // 类型名称
    public bool IsNormal { get; set; }      // 是否正常（最新记录）
    public string AnomalyType { get; set; } // 异常类型（最新记录）
    public string AudioPath { get; set; }   // 音频路径（最新记录）
    public DateTime? CollectedAt { get; set; } // 采集时间（最新记录）
}
```

**业务逻辑：**
1. **设备统计** - 仅统计有父节点的设备（`ParentId IS NOT NULL`）
2. **状态关联** - 从 `vp_device_audio_record` 获取每台设备的最新识别结果
3. **类型分组** - 按 `MonitoredObjectTypeId` 分组统计

---

#### 5. 首页告警信息 (`GetDashboardAlarmsAsync`)

**端点：** `GET /api/app/voiceprint/dashboard/alarms`

**职责：** 获取告警统计和最新告警列表

**返回结果：**
```csharp
public class VoiceprintDashboardAlarmDto
{
    public int ProcessingCount { get; set; }  // 处理中告警数
    public int PendingCount { get; set; }     // 待处理告警数
    public IReadOnlyList<VoiceprintDashboardAlarmItemDto> Items { get; set; }  // 告警列表
}

public class VoiceprintDashboardAlarmItemDto
{
    public Guid Id { get; set; }              // 告警 ID
    public string AlarmMessage { get; set; }   // 告警消息
    public DateTime AlarmTime { get; set; }    // 告警时间
    public string AudioPath { get; set; }     // 音频路径
    public string DevicePath { get; set; }    // 设备路径（如"变压器>风扇"）
    public string ProcessingStatus { get; set; } // 处理状态字符串
}
```

**状态统计：**
```sql
-- 处理中
SELECT COUNT(*) FROM vp_voiceprint_alarm 
WHERE ProcessingStatus = 1  -- Processing

-- 待处理
SELECT COUNT(*) FROM vp_voiceprint_alarm 
WHERE ProcessingStatus = 0  -- Pending
```

**排序规则：** 按 `AlarmTime` 降序（最新告警优先）

---

#### 6. 告警报告生成 (`GenerateAlarmReportAsync`)

**端点：** `POST /api/app/voiceprint/dashboard/generate-alarm-report`

**职责：** 根据多台异常设备生成声纹告警报表

**请求参数：**
```csharp
public class VoiceprintAlarmReportGenerateInput
{
    public IReadOnlyList<VoiceprintAlarmReportDeviceInput> Devices { get; set; }
}

public class VoiceprintAlarmReportDeviceInput
{
    public Guid MonitoredObjectId { get; set; }   // 设备 ID
    public string DeviceName { get; set; }        // 设备名称
    public string AudioPath { get; set; }         // 音频路径
    public string AnomalyType { get; set; }       // 异常类型
    public DateTime CollectedAt { get; set; }     // 采集时间
}
```

**返回结果：**
```csharp
public IReadOnlyList<VoiceprintAudioReportGenerateResultDto>
```

**业务流程：**
1. **参数验证** - 检查设备列表和音频路径
2. **逐设备生成** - 为每台设备生成独立报告
3. **标准库匹配** - 查询同类标准音频 Top 3
4. **频域图生成** - 生成目标音频、标准音频和对比图
5. **模板填充** - 根据异常类型选择模板并填充
6. **返回结果列表** - 每台设备返回一个报告下载路径

---

### Assets - 设备资产管理

#### 1. 设备台账树 (`GetAssetTreeAsync`)

**端点：** `GET /api/app/voiceprint/assets/tree`

**职责：** 获取完整的设备树形结构

**返回结果：**
```csharp
public class VoiceprintAssetTreeNodeDto
{
    public Guid Id { get; set; }                    // 节点 ID
    public string Name { get; set; }                // 节点名称
    public string Status { get; set; }             // 状态字符串
    public Guid? ParentId { get; set; }            // 父节点 ID
    public IReadOnlyList<VoiceprintAssetTreeNodeDto> Children { get; set; }  // 子节点
}
```

**数据结构：**
```
根节点（变电站）
├── 1号主变
│   ├── 铁芯
│   ├── 夹件
│   └── 风扇
└── 2号主变
    ├── 铁芯
    └── 线圈
```

---

#### 2. 设备基础信息 (`GetDeviceInfoAsync`)

**端点：** `GET /api/app/voiceprint/assets/{monitoredObjectId}`

**职责：** 获取设备的详细信息和属性组

**返回结果：**
```csharp
public class VoiceprintDeviceInfoDto
{
    public Guid Id { get; set; }                            // 设备 ID
    public string Name { get; set; }                        // 设备名称
    public string Status { get; set; }                     // 状态字符串
    public SimpleDeviceReferenceDto Parent { get; set; }   // 父设备信息
    public IReadOnlyList<VoiceprintDeviceAttrGroupDto> Groups { get; set; }  // 属性组列表
}

public class VoiceprintDeviceAttrGroupDto
{
    public Guid Id { get; set; }                            // 属性组 ID
    public string Name { get; set; }                        // 属性组名称
    public IReadOnlyList<VoiceprintDeviceAttrDto> Attributes { get; set; }  // 属性列表
}

public class VoiceprintDeviceAttrDto
{
    public Guid Id { get; set; }            // 属性 ID
    public string Name { get; set; }        // 属性名称
    public string Description { get; set; } // 属性描述
    public string Value { get; set; }       // 属性值
}
```

**查询逻辑：**
1. 主设备信息：`monitored_object` 表
2. 父设备信息：通过 `ParentId` 关联
3. 属性组：`monitored_object_attr_group` 表
4. 属性明细：`monitored_object_attr` 表（支持 `StrVal` 和 `Val` 两种值类型）

---

#### 3. 设备运行趋势 (`GetDeviceTrendAsync`)

**端点：** `GET /api/app/voiceprint/assets/{monitoredObjectId}/trend`

**职责：** 获取指定时间段的设备运行趋势

**请求参数：**
```csharp
public DateTime startTime { get; set; }  // 开始时间（默认：7天前）
```

**返回结果：**
```csharp
public IReadOnlyList<VoiceprintTrendItemDto>
{
    public DateTime CollectedAt { get; set; }  // 采集时间
    public bool IsNormal { get; set; }         // 是否正常
}
```

**查询逻辑：**
```sql
SELECT CollectedAt, IsNormal
FROM vp_device_audio_record
WHERE MonitoredObjectId = @Id
  AND CollectedAt >= @StartTime
ORDER BY CollectedAt ASC
```

---

#### 4. 设备异常分布 (`GetDeviceAnomalyMixAsync`)

**端点：** `GET /api/app/voiceprint/assets/{monitoredObjectId}/anomaly-mix`

**职责：** 统计设备各种异常类型的分布

**返回结果：**
```csharp
public IReadOnlyList<VoiceprintAnomalyMixItemDto>
{
    public string AnomalyType { get; set; }  // 异常类型
    public int Count { get; set; }           // 出现次数
    public double Ratio { get; set; }         // 占比（0-1）
}
```

**统计逻辑：**
1. 查询所有非正常记录并按 `AnomalyType` 分组
2. 计算每种异常的数量和占比
3. 如果无异常记录，返回全正常（占比 100%）

---

#### 5. 设备巡检日志 (`GetDeviceLogsAsync`)

**端点：** `GET /api/app/voiceprint/assets/{monitoredObjectId}/logs`

**职责：** 获取设备的巡检日志（分页）

**请求参数：**
```csharp
public DateTime? startTime { get; set; }  // 开始时间（可选）
public DateTime? endTime { get; set; }    // 结束时间（可选）
public int page { get; set; }             // 页码（默认：1）
public int pageSize { get; set; }         // 页大小（默认：50，最大：200）
```

**返回结果：**
```csharp
public class VoiceprintDeviceLogPagedResultDto
{
    public int TotalCount { get; set; }   // 总记录数
    public IReadOnlyList<VoiceprintDeviceLogDto> Items { get; set; }  // 当前页数据
}

public class VoiceprintDeviceLogDto
{
    public Guid RecordId { get; set; }          // 记录 ID
    public string AnomalyType { get; set; }     // 异常类型
    public string AudioPath { get; set; }       // 音频路径
    public DateTime CollectedAt { get; set; }   // 采集时间
    public bool IsNormal { get; set; }          // 是否正常
    public string StandardNormalAudioPath { get; set; }   // 标准正常音频路径（填充）
    public string StandardAnomalyAudioPath { get; set; }  // 标准异常音频路径（填充）
}
```

**业务逻辑：**
1. **基础查询** - 按时间范围和设备 ID 查询
2. **标准音频填充** - 为每条记录填充对应的标准音频路径：
   - 正常记录：随机填充一条"运行正常"标准音频
   - 异常记录：根据 `AnomalyType` 随机填充一条同类型标准音频
3. **排序规则** - 按 `CollectedAt` 降序

---

#### 6. 批量下载处理后音频 (`DownloadProcessedAudiosAsync`)

**端点：** `GET /api/app/voiceprint/assets/{monitoredObjectId}/processed-audios/download`

**职责：** 打包下载指定时间范围的处理后音频（ZIP 格式）

**请求参数：**
```csharp
public DateTime startTime { get; set; }  // 开始时间（默认：7天前）
public DateTime endTime { get; set; }    // 结束时间（默认：现在）
```

**返回类型：** `FileStreamResult` (application/zip)

**文件命名：** `{设备名称}_{开始时间}_{结束时间}.zip`

**业务流程：**
1. **时间范围验证** - 确保开始时间 < 结束时间
2. **查询音频记录** - 查询指定设备在时间范围内的处理后音频
3. **打包 ZIP**：
   - 使用文件名格式：`{设备名称}_{时间戳}.wav`
   - 处理文件名冲突（自动添加后缀 `_1`, `_2`）
   - 使用无压缩模式（`CompressionLevel.NoCompression`）避免二次压缩
   - 1MB 缓冲区复制文件
4. **流式返回** - 使用 `FileOptions.DeleteOnClose` 自动删除临时文件

**性能优化：**
```csharp
const int CopyBufferSize = 1024 * 1024; // 1MB 缓冲区
await fileStream.CopyToAsync(entryStream, CopyBufferSize);
```

**日志记录：**
```
开始导出: monitoredObjectId=xxx, startTime=xxx, endTime=xxx, recordCount=10
导出完成: requested=10, added=9, skippedMissing=1, elapsedMs=1234, zipSizeBytes=9876543
```

---

### Standard Audio - 标准音频库

#### 1. 标准音频列表 (`GetStandardAudioListAsync`)

**端点：** `GET /api/app/voiceprint/standard-audios`

**职责：** 获取按异常类型分组的标准音频列表

**返回结果：**
```csharp
public IReadOnlyList<VoiceprintStandardAudioGroupDto>
{
    public string AudioType { get; set; }    // 音频类型（如"运行正常"）
    public IReadOnlyList<VoiceprintStandardAudioDto> Audios { get; set; }  // 该类型的音频列表
}

public class VoiceprintStandardAudioDto
{
    public Guid Id { get; set; }             // 音频 ID
    public string AudioName { get; set; }    // 音频名称
    public string AudioType { get; set; }    // 音频类型
    public string AudioFilePath { get; set; } // 音频路径
    public string AudioMd5 { get; set; }     // 音频 MD5
    public string Description { get; set; }  // 描述
}
```

**分组规则：**
1. 按音频类型（`AudioType`）分组
2. 过滤"测试音频"类型
3. 排序：`运行正常` 优先，其他按类型名称升序

---

#### 2. 上传标准音频 (`CreateStandardAudioAsync`)

**端点：** `POST /api/app/voiceprint/standard-audios`

**职责：** 上传音频到标准库

**请求参数：**
```csharp
[FromForm] string anomalyType    // 异常类型
[FromForm] IFormFile file        // 音频文件
[FromForm] string audioName      // 音频名称（可选）
```

**业务流程：**
1. **参数验证** - 检查异常类型和文件非空
2. **文件保存** - 保存到 `wwwroot/audio/standard_library/` 目录
3. **MD5 计算** - 计算音频文件 MD5 哈希
4. **数据库记录** - 创建 `VoiceprintStandardAudioEntity` 记录

**字段映射：**
- `AudioName` - 如果未提供，使用文件名（不含扩展名）
- `AudioType` - 使用输入的 `anomalyType`
- `AudioFilePath` - 公共访问路径（`/audio/standard_library/{文件名}`）
- `AudioMd5` - 计算得到的 MD5 值

---

#### 3. 随机获取标准音频 (`GetRandomStandardAudioAsync`)

**端点：** `GET /api/app/voiceprint/standard-audios/random`

**职责：** 随机获取指定类型的一条标准音频

**请求参数：**
```csharp
public string anomalyType { get; set; }  // 异常类型
```

**返回结果：** `VoiceprintStandardAudioDto`

**SQL 查询：**
```sql
SELECT * FROM vp_standard_audio_library
WHERE AudioType = @anomalyType
ORDER BY RAND()
LIMIT 1
```

---

#### 4. 导入测试音频 (`ImportTestAudioAsync`)

**端点：** `POST /api/app/voiceprint/test-audios/import`

**职责：** 上传测试音频并触发识别

**请求参数：**
```csharp
public IFormFile file { get; set; }  // 音频文件
```

**返回结果：**
```csharp
public class VoiceprintStandardAudioImportResultDto
{
    public string FileName { get; set; }       // 文件名
    public long FileSize { get; set; }         // 文件大小
    public string AudioPath { get; set; }      // 音频路径
    public string ResultMessage { get; set; }  // 结果消息
    public string Label { get; set; }          // 识别标签
    public double? Score { get; set; }         // 置信度
    public double? DurationSeconds { get; set; } // 音频时长
    public string StandardAudioPath { get; set; } // 匹配的标准音频路径
}
```

**业务流程：**
1. **文件保存** - 保存到测试音频目录（`tests/audio/`）
2. **MD5 匹配** - 查询标准库是否有相同音频
3. **识别调用** - 如果未命中标准库，调用 Python 识别服务
4. **标准音频查询** - 根据识别结果查询对应的标准音频

**识别服务接口：**
```http
POST {VoiceprintRecognition:TestAudioApiUrl}
Content-Type: application/json

{
  "audioPath": "test_audio_20250622.wav"
}
```

---

### Alarm Management - 告警管理

#### 1. 近30天告警统计 (`GetMonthlyStatsAsync`)

**端点：** `GET /api/app/voiceprint/alarms/monthly-stat`

**职责：** 提供近30天的告警统计数据

**返回结果：**
```csharp
public class VoiceprintMonthlyStatDto
{
    public IReadOnlyList<VoiceprintKeyValueStatDto> AnomalyStats { get; set; }    // 按异常类型统计
    public IReadOnlyList<VoiceprintKeyValueStatDto> ProcessingStats { get; set; }  // 按处理状态统计
    public IReadOnlyList<VoiceprintDeviceAlarmStatDto> DeviceStats { get; set; }   // 按设备统计（Top 20）
}

public class VoiceprintKeyValueStatDto
{
    public string Key { get; set; }    // 键（类型或状态）
    public int Count { get; set; }     // 数量
}

public class VoiceprintDeviceAlarmStatDto
{
    public Guid MonitoredObjectId { get; set; }  // 设备 ID
    public string DeviceName { get; set; }      // 设备名称
    public int Count { get; set; }               // 告警数量
}
```

**统计维度：**
1. **异常类型分布** - 从 `vp_device_audio_record` 统计异常类型
2. **处理状态分布** - 从 `vp_voiceprint_alarm` 统计处理状态
3. **设备告警排名** - Top 20 告警最多的设备

---

#### 2. 报警筛选设备下拉 (`GetDeviceOptionsAsync`)

**端点：** `GET /api/app/voiceprint/alarms/device-options`

**职责：** 获取可用于筛选的设备列表

**返回结果：**
```csharp
public IReadOnlyList<SimpleDeviceReferenceDto>
{
    public Guid Id { get; set; }     // 设备 ID
    public string Name { get; set; } // 设备名称
}
```

**筛选规则：** 仅返回有父节点的设备（`ParentId IS NOT NULL`）

**排序规则：** 按设备名称升序

---

#### 3. 报警记录列表 (`GetAlarmListAsync`)

**端点：** `GET /api/app/voiceprint/alarms`

**职责：** 分页查询报警记录（支持多维度筛选）

**请求参数：**
```csharp
public DateTime? startTime { get; set; }        // 开始时间（可选）
public DateTime? endTime { get; set; }          // 结束时间（可选）
public Guid? monitoredObjectId { get; set; }    // 设备 ID（可选）
public int page { get; set; }                   // 页码（默认：1）
public int pageSize { get; set; }               // 页大小（默认：50，最大：200）
```

**返回结果：**
```csharp
public class VoiceprintAlarmPagedResultDto
{
    public int TotalCount { get; set; }   // 总记录数
    public IReadOnlyList<VoiceprintAlarmListItemDto> Items { get; set; }  // 当前页数据
}

public class VoiceprintAlarmListItemDto
{
    public Guid Id { get; set; }              // 告警 ID
    public DateTime AlarmTime { get; set; }    // 告警时间
    public string AlarmMessage { get; set; }   // 告警消息
    public string AudioPath { get; set; }     // 音频路径
    public string DevicePath { get; set; }    // 设备路径（如"变压器>风扇"）
    public int ProcessingStatus { get; set; }  // 处理状态（0=待处理, 1=处理中, 2=已完成, 3=已忽略）
    public string Remark { get; set; }         // 备注信息
}
```

**筛选逻辑：**
- **时间范围** - `AlarmTime >= startTime AND AlarmTime < endTime`
- **设备筛选** - `MonitoredObjectId == monitoredObjectId OR ParentId == monitoredObjectId`

**排序规则：** 按 `AlarmTime` 降序

---

#### 4. 处理报警 (`ProcessAlarmAsync`)

**端点：** `PUT /api/app/voiceprint/alarms/{alarmId}`

**职责：** 更新告警的处理状态和备注

**请求参数：**
```csharp
public class ProcessVoiceprintAlarmInput
{
    public string ProcessingStatus { get; set; }  // 处理状态字符串
    public string Remark { get; set; }             // 备注信息
}
```

**处理状态枚举：**
```csharp
public enum VoiceprintAlarmProcessingStatusEnum
{
    Pending = 0,      // 待处理
    Processing = 1,   // 处理中
    Completed = 2,    // 已完成
    Ignored = 3      // 已忽略
}
```

**业务流程：**
1. **参数验证** - 检查告警存在性和状态有效性
2. **更新告警** - 更新处理状态和备注
3. **设备状态恢复** - 如果状态为 `Completed`，检查是否需要恢复设备为正常：
   - 查询该设备是否有更新的告警
   - 如果没有，将设备状态恢复为 `Normal`

**状态恢复逻辑：**
```csharp
private async Task TryRestoreMonitoredObjectStatusAsync(VoiceprintAlarmRecordEntity alarm)
{
    if (alarm.ProcessingStatus != VoiceprintAlarmProcessingStatusEnum.Completed)
        return;

    // 检查是否有更新的告警
    var hasNewerAlarm = await _db.Queryable<VoiceprintAlarmRecordEntity>()
        .Where(x => x.MonitoredObjectId == alarm.MonitoredObjectId
                    && x.Id != alarm.Id
                    && x.AlarmTime > alarm.AlarmTime)
        .AnyAsync();

    if (hasNewerAlarm)
        return;

    // 恢复设备为正常状态
    await _db.Updateable<MonitoredObjectAggregateRoot>()
        .Where(x => x.Id == alarm.MonitoredObjectId)
        .SetColumns(x => new MonitoredObjectAggregateRoot
        {
            Status = CommonStatusEnum.Normal,
            LastModificationTime = Clock.Now
        })
        .ExecuteCommandAsync();
}
```

---

### Reports - 报告导出

#### 综合报告导出 (`ExportReportAsync`)

**端点：** `POST /api/app/voiceprint/reports/export`

**职责：** 生成指定时间范围的综合声纹巡视报告

**请求参数：**
```csharp
public class VoiceprintReportExportInput
{
    public DateTime StartTime { get; set; }     // 开始时间
    public DateTime EndTime { get; set; }       // 结束时间
    public string ResultFileName { get; set; }  // 报告文件名（可选）
}
```

**返回结果：**
```csharp
public class VoiceprintReportExportResultDto
{
    public string DownloadPath { get; set; }  // 下载路径
}
```

**报告内容结构：**

1. **总体统计（截止时间）**
   ```json
   {
     "last_patrol_time": "2025-06-22 10:00:00",
     "total_patrol_days": 180,
     "total_patrol_count": 8640,
     "total_devices": 45,
     "traction_transformer_count": 12,
     "power_transformer_count": 18,
     "active_alarm_device_count": 3
   }
   ```

2. **期段统计（时间范围内）**
   ```json
   {
     "start_time": "2025-06-01 00:00:00",
     "end_time": "2025-06-22 23:59:59",
     "patrol_days": 22,
     "patrol_count": 1056,
     "alarm_device_count": 5,
     "alarm_count": 12
   }
   ```

3. **设备频域图列表（截止时间最近批次）**
   - 取最近一批次的 5 台设备
   - 每台设备包含：
     - 现场音频频域图
     - 标准音频频域图（随机匹配）
     - 对比图

4. **告警记录列表（时间范围内）**
   ```json
   [{
     "seq": 1,
     "alarm_event": "风扇轴承摩擦",
     "device_path": "1号主变>风扇",
     "alarm_message": "检测到风扇轴承摩擦异常",
     "processing_status": "已处理",
     "processing_log": "已安排检修",
     "alarm_time": "2025-06-15 14:30:00"
   }]
   ```

**业务流程：**
1. **时间验证** - 确保开始时间 < 结束时间
2. **总体统计** - 查询截止时间的累计数据
3. **期段统计** - 查询时间范围内的增量数据
4. **设备查询** - 获取最近一批次的设备及其最新音频记录
5. **标准库匹配** - 为每台设备随机匹配同类型标准音频
6. **频域图生成** - 批量生成现场、标准和对比频域图
7. **告警查询** - 查询时间范围内的所有告警并按时间排序
8. **模板填充** - 调用报告服务填充模板
9. **返回路径** - 返回报告下载路径

**模板配置：**
```json
{
  "VoiceprintReport": {
    "TemplateFileName": "report_template.docx",
    "ImageWidth": 800
  }
}
```

---

### Collector Logs - 采集器通信日志

#### 采集器日志查询 (`GetCollectorLogsAsync`)

**端点：** `GET /api/app/voiceprint/collector/logs`

**职责：** 查询采集器通信日志（分页，支持筛选）

**请求参数：**
```csharp
public string collectorDeviceId { get; set; }  // 采集器设备 ID（可选）
public Guid? groupId { get; set; }             // 批次 ID（可选）
public string status { get; set; }             // 状态（可选）
public int page { get; set; }                  // 页码（默认：1）
public int pageSize { get; set; }              // 页大小（默认：50，最大：200）
```

**返回结果：**
```csharp
public class VoiceprintCollectorLogPagedResultDto
{
    public int TotalCount { get; set; }   // 总记录数
    public IReadOnlyList<VoiceprintCollectorLogItemDto> Items { get; set; }  // 当前页数据
}

public class VoiceprintCollectorLogItemDto
{
    public Guid Id { get; set; }                    // 日志 ID
    public string CollectorDeviceId { get; set; }  // 采集器设备 ID
    public Guid? MonitoredObjectId { get; set; }    // 监控对象 ID
    public Guid? GroupId { get; set; }             // 批次 ID
    public string Status { get; set; }             // 状态（如"success", "failed"）
    public string Message { get; set; }             // 日志消息
    public DateTime OccurredAt { get; set; }        // 发生时间
}
```

**排序规则：** 按 `OccurredAt` 降序

---

## 数据库查询优化

### 分页查询模式

所有列表接口均采用 SqlSugar 分页模式：

```csharp
RefAsync<int> total = 0;
var items = await query
    .OrderBy(x => x.CreatedAt, OrderByType.Desc)
    .Select(x => new Dto { ... })
    .ToPageListAsync(page, pageSize, total);

return new PagedResultDto
{
    TotalCount = total,
    Items = items
};
```

### 联表查询优化

典型三表联查（告警记录详情）：

```csharp
var query = _db.Queryable<VoiceprintAlarmRecordEntity>()
    .LeftJoin<VoiceprintDeviceAudioRecordEntity>((alarm, record) => alarm.DeviceAudioRecordId == record.Id)
    .LeftJoin<MonitoredObjectAggregateRoot>((alarm, record, obj) => alarm.MonitoredObjectId == obj.Id)
    .LeftJoin<MonitoredObjectAggregateRoot>((alarm, record, obj, parent) => obj.ParentId == parent.Id);

var list = await query
    .OrderBy((alarm, record, obj, parent) => alarm.AlarmTime, OrderByType.Desc)
    .Select((alarm, record, obj, parent) => new VoiceprintAlarmListItemDto
    {
        Id = alarm.Id,
        AlarmMessage = alarm.AlarmMessage,
        DevicePath = parent.Name == null ? obj.Name : $"{parent.Name}>{obj.Name}",
        ...
    })
    .ToPageListAsync(page, pageSize, total);
```

---

## 频域图生成技术

### FFT 频谱分析

**技术栈：** NAudio + 自实现 FFT

**处理流程：**
1. **音频读取** - 使用 `AudioFileReader` 读取 WAV 文件
2. **采样窗口** - 取音频中间 4096 个采样点
3. **汉宁窗** - 应用汉宁窗减少频谱泄漏
4. **FFT 计算** - 执行快速傅里叶变换
5. **幅度转换** - 转换为对数刻度（dB）
6. **归一化** - 归一化到 0-255 范围

**汉宁窗公式：**
```csharp
var windowValue = 0.5 * (1 - Math.Cos(2 * Math.PI * i / (fftSize - 1)));
var sampleValue = idx < channelData.Length ? channelData[idx] * windowValue : 0;
samples[i] = new Complex(sampleValue, 0);
```

**FFT 幅度计算：**
```csharp
var magnitude = samples[i].Magnitude;
magnitudes[i] = 20 * Math.Log10(magnitude + 1e-10);  // 转换为 dB
```

**频率分辨率计算：**
```csharp
var frequencyStep = sampleRate / (double)fftSize;
// 44100 Hz / 4096 ≈ 10.77 Hz 频率分辨率
```

### 图表服务调用

**频域图服务接口：**
```http
POST {VoiceprintReport:ChartApiUrl}/frequency-domain-graph
Content-Type: application/json

{
  "width": 800,
  "height": 240,
  "xAxisData": [0, 10.77, 21.54, ...],
  "seriesData": [{
    "name": "现场",
    "value": [120.5, 135.2, ...]
  }],
  "xAxisName": "Hz",
  "yAxisName": "幅度"
}
```

**对比图服务接口：**
```http
POST {VoiceprintReport:ChartApiUrl}/frequency-domain-graph
Content-Type: application/json

{
  "width": 800,
  "height": 240,
  "xAxisData": [0, 10.77, 21.54, ...],
  "seriesData": [
    { "name": "现场", "value": [120.5, 135.2, ...] },
    { "name": "标准", "value": [118.3, 132.1, ...] }
  ],
  "xAxisName": "Hz",
  "yAxisName": "幅度"
}
```

---

## 外部服务依赖

### 报告生成服务

**配置项：** `VoiceprintReport:ApiUrl`

**接口约定：**
```http
POST {ApiUrl}
Content-Type: application/json

{
  "templateFileName": "report_template.docx",
  "resultFileName": "变压器声纹声音智能巡视报表_20250622.docx",
  "imageWidth": 800,
  "context": {
    "time": "2025-06-22 10:30:00",
    "summary": { ... },
    "period": { ... },
    "device_list": [ ... ],
    "alarm_list": [ ... ]
  }
}
```

### 频域图生成服务

**配置项：** `VoiceprintReport:ChartApiUrl`

**接口约定：** 详见"频域图生成技术"章节

### 测试音频识别服务

**配置项：** `VoiceprintRecognition:TestAudioApiUrl`

**接口约定：**
```http
POST {TestAudioApiUrl}
Content-Type: application/json

{
  "audioPath": "test_audio_20250622.wav"
}

Response:
{
  "label": "运行正常",
  "score": 0.95,
  "durationSeconds": 10.5
}
```

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
  }
}
```

---

## 常见问题

### Q: 设备树结构不正确，缺少层级关系

**原因：** 设备的 `ParentId` 未正确设置

**解决方案：**
1. 检查 `monitored_object` 表的 `ParentId` 字段
2. 确认根节点（变电站）的 `ParentId` 为 NULL
3. 确认子设备正确指向父设备的 ID

### Q: 告警列表显示的设备路径不正确

**原因：** 设备路径由父设备名称和子设备名称拼接，如果父设备为空则只显示子设备名称

**解决方案：**
```csharp
DevicePath = parent.Name == null ? obj.Name : $"{parent.Name}>{obj.Name}"
```

### Q: 下载的音频 ZIP 文件过大

**原因：** 使用了无压缩模式（`CompressionLevel.NoCompression`）避免二次压缩

**优化建议：**
1. 限制时间范围（默认 7 天）
2. 减少页大小（默认 50 条）
3. 考虑使用 `CompressionLevel.Fast` 压缩

### Q: 报告生成超时

**原因：** 频域图生成耗时较长

**优化建议：**
1. 调整 `VoiceprintReport:ImageWidth` 降低分辨率
2. 减少报告中的设备数量（默认 5 台）
3. 增加 HTTP 客户端超时时间

### Q: 标准音频随机获取总是返回相同音频

**原因：** SqlSugar 的 `SqlFunc.GetRandom()` 在某些数据库上可能不生效

**解决方案：**
```sql
-- 手动在应用层随机排序
var audios = await query.ToListAsync();
var randomAudio = audios[random.Next(audios.Count)];
```

---

## 相关文档

### 模块总览
- [[README.md|ast-voiceprint 模块总览]]

### 相关服务
- [[VoiceprintAudioAppService.md|VoiceprintAudioAppService - 音频处理服务]]

### 功能文档
- [[../../../voiceprint/声纹数据库实体介绍文档.md|声纹数据库实体介绍文档]]
- [[../../../voiceprint/分析报表需求文档.md|分析报表需求文档]]

### 运维文档
- [[../../../voiceprint/已处理音频下载接口优化方案.md|已处理音频下载接口优化方案]]

---

**文档版本：** v1.0.0  
**最后更新：** 2025-06-18  
**维护者：** ast-voiceprint 模块团队
