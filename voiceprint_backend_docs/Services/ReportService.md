# ReportService — 综合报表服务

## 基本信息

- **服务名称**：`ReportService`
- **模块位置**：`module/ast-intellisub/Ast.IntelliSub.Application/Services/ReportService.cs`
- **接口实现**：`IReportService`
- **认证要求**：`[Authorize]` - 需要认证
- **操作日志**：所有报表生成方法均记录操作日志

## 服务概述

ReportService 是变电站监控系统中负责综合报表生成的核心服务，负责从多个数据源聚合数据，生成各类运行报告，包括综合运行报告、设备运行报告、巡检报告等。

### 核心职责

1. **数据聚合** - 从多个数据源（设备、告警、巡检、监测数据）聚合信息
2. **报表生成** - 按业务规则生成多种类型的运行报告
3. **外部API集成** - 将生成的报表数据推送到外部系统
4. **状态分析** - 分析设备运行状态，生成运行汇总

## 依赖注入

### 仓储依赖（14个）

| 仓储 | 用途 |
|------|------|
| `SubstationAggregateRoot` | 变电站基础信息查询 |
| `MonitoredObjectAggregateRoot` | 监测对象（设备）查询 |
| `MonitoredObjectItemRelEntity` | 监测对象-监测项关联 |
| `PointBindingRelEntity` | 点位绑定关系 |
| `AstPointEntity` | 数据点定义 |
| `AlarmRecordAggregateRoot` | 告警记录查询 |
| `AlarmRecordItemEntity` | 告警记录项 |
| `PatrolRecordEntity` | 巡检记录 |
| `PatrolRecordItemEntity` | 巡检记录项 |
| `PatrolTaskAggregateRoot` | 巡检任务 |
| `PatrolTaskMonitoredPointRelEntity` | 巡检任务-点位关联 |
| `MonitoredPointEntity` | 监测点位 |
| `MonitoredItemEntity` | 监测项模板 |
| `DeviceEntity` | 设备信息 |

### 服务依赖

| 服务 | 用途 |
|------|------|
| `IHttpClientFactory` | HTTP客户端工厂，用于外部API调用 |
| `IConfiguration` | 配置读取 |
| `ILogger<ReportService>` | 日志记录 |
| `IPointDataRepository` | 时序数据查询 |
| `IDataFilter` | 数据过滤（多租户等） |
| `IStreamingTransferService` | 流式传输服务 |

## 核心方法

### 1. ComprehensiveOperationReport - 综合运行报告

**签名**：
```csharp
[OperLog("生成综合运行报告", OperEnum.Export)]
public async Task<ReportResponse> ComprehensiveOperationReport(RequestReportDataDto req)
```

**功能**：
生成指定时间段内变电站的综合运行报告，包含设备统计、告警信息、正常设备列表等。

**执行流程**：

```
1. 查询变电站基础信息和设备列表
   │
   ├─ 查询变电站及其监测对象
   ├─ 关联监测对象类型
   ├─ 过滤有效设备（有ParentId的）
   │
2. 设备分组统计
   │
   ├─ 按设备ID和名称去重
   ├─ 按设备种类分组统计
   │  ├─ GIS设备数量
   │  ├─ 接地变压器数量
   │  └─ 油浸式变压器数量
   │
3. 查询告警数据
   │
   ├─ 按时间段过滤告警记录
   ├─ 关联监测对象和父级设备
   ├─ 关联监测点位
   │
4. 查询监测数据
   │
   ├─ 查询设备的监测点位
   ├─ 获取时序数据（PointData）
   ├─ 处理波形文件数据（可选）
   │
5. 生成告警设备列表
   │
   ├─ 按设备名称和监测点位分组
   ├─ 取最早的告警记录
   ├─ 统计告警数量
   │
6. 生成正常设备列表
   │
   ├─ 过滤出无告警的设备
   ├─ 为每个设备生成运行状态汇总
   │  ├─ GIS设备：温度、局放、操作时间
   │  ├─ 接地变压器：特高频局放、声纹
   │  └─ 油浸式变压器：高频电流局放、铁芯接地电流、声纹
   │
7. 生成运行状态汇总
   │
   ├─ 计算告警设备数量
   ├─ 计算正常设备数量
   └─ 生成汇总文本
   │
8. 构建报表数据结构
   │
9. 数据验证（无数据直接返回）
   │
10. 发送到外部API
    │
    ├─ 优先使用HTTP Chunked Transfer-Encoding
    ├─ 失败时回退到普通HTTP POST
    └─ 返回API响应结果
```

**返回数据结构**：
```csharp
public class ComprehensiveOperationReportDto
{
    public string ReportTitle { get; set; }              // 报表标题
    public string ReportPeriodStart { get; set; }       // 报告开始时间
    public string ReportPeriodEnd { get; set; }          // 报告结束时间
    public int DeviceTotal { get; set; }                 // 设备总数
    public int GisCount { get; set; }                    // GIS设备数量
    public int GroundTransformerCount { get; set; }     // 接地变压器数量
    public int OilTransformerCount { get; set; }        // 油浸式变压器数量
    public int SummaryTotal { get; set; }               // 汇总总数
    public int AlarmCount { get; set; }                 // 告警设备数量
    public int NormalCount { get; set; }                // 正常设备数量
    public List<AlarmDeviceDto> AlarmDevices { get; set; }    // 告警设备列表
    public List<NormalDeviceDto> NormalDevices { get; set; }   // 正常设备列表
    public string Summary { get; set; }                  // 运行状态汇总
}
```

### 2. DeviceOperationReport - 设备运行报告

**签名**：
```csharp
[OperLog("生成设备运行报告", OperEnum.Export)]
public async Task<ReportResponse> DeviceOperationReport(RequestReportDataDto req)
```

**功能**：
生成单个设备的详细运行报告，包含设备的各类监测数据。

### 3. GetMonitoringDataForDevices - 获取设备监测数据

**签名**：
```csharp
private async Task<List<MonitoringDataInfo>> GetMonitoringDataForDevices(
    List<Guid> deviceIds, DateTime startTime, DateTime endTime)
```

**功能**：
查询指定设备列表在时间范围内的所有监测数据。

**执行流程**：
1. 查询设备的监测点位配置
2. 关联监测对象-监测项关系
3. 关联点位绑定关系
4. 从时序数据库获取实际数据
5. 处理波形文件数据
6. 返回合并后的监测数据

### 4. GenerateAlarmDeviceList - 生成告警设备列表

**签名**：
```csharp
private List<AlarmDeviceDto> GenerateAlarmDeviceList(
    List<ComprehensiveAlarmInfo> alarmData)
```

**功能**：
从告警数据中生成告警设备列表，按设备和监测点位分组。

**分组规则**：
- 按 `DeviceName + MonitoredPointName + DeviceId + MonitoredPointId + AlarmContent` 分组
- 每个组合取最早的告警记录
- 统计每个监测点位的告警数量（超过5显示"5+"）

### 5. GenerateNormalDeviceList - 生成正常设备列表

**签名**：
```csharp
private async Task<List<NormalDeviceDto>> GenerateNormalDeviceList(
    List<DeviceInfo> allDevices,
    List<ComprehensiveAlarmInfo> alarmData,
    List<MonitoringDataInfo> monitoringData)
```

**功能**：
生成无告警的设备列表，为每个设备生成运行状态汇总。

**处理逻辑**：
1. 从所有设备中排除有告警的设备
2. 按创建时间排序
3. 查询父级设备名称作为Location
4. 根据设备类型生成不同的状态汇总

### 6. GenerateOperationStatusSummary - 生成设备运行状态汇总

**签名**：
```csharp
private List<string> GenerateOperationStatusSummary(
    DeviceInfo device, 
    List<MonitoringDataInfo> deviceData)
```

**功能**：
根据设备类型和监测数据生成运行状态汇总信息。

**设备类型处理**：

| 设备类型 | 状态汇总内容 |
|---------|-------------|
| GIS中压交流柜 | 动触头三相温度、二合一局放、分闸合闸时间 |
| 接地变压器 | 特高频局放、声纹声音 |
| 油浸式变压器 | 高频电流局放、铁芯接地电流、声纹声音 |
| 其他设备 | 尝试生成温度汇总和声纹汇总 |

### 7. SendComprehensiveReportDataAsync - 发送报表数据

**签名**：
```csharp
private async Task<ReportResponse> SendComprehensiveReportDataAsync(
    ComprehensiveOperationReportDto reportData)
```

**功能**：
将生成的报表数据发送到外部API。

**传输策略**：
1. **优先使用**：HTTP Chunked Transfer-Encoding 流式传输（通过 `IStreamingTransferService`）
2. **回退方案**：普通 HTTP POST 请求（使用 `IHttpClientFactory`）

**配置项**：
- `ExternalApi:ComprehensiveReportUrl` - 综合报告API地址
- `ExternalApi:ReportDataUrl` - 默认报表API地址（备用）

## 辅助方法

### 数据处理方法

| 方法 | 功能 |
|------|------|
| `ParseDoubleValues` | 解析字符串值为双精度数值列表 |
| `GetNumericValues` | 根据Property名称获取数值型数据 |
| `GetNumericValuesByMonitoredPointName` | 根据Property和监测点位名称获取数值 |
| `GetStringValues` | 获取字符串型数据 |
| `ProcessWaveformFileDataForMonitoringInfo` | 处理波形文件数据 |

### 状态汇总生成方法

| 方法 | 功能 |
|------|------|
| `GenerateSwitchCabinetStatusSummary` | 生成开关柜状态汇总 |
| `GenerateGroundTransformerStatusSummary` | 生成接地变压器状态汇总 |
| `GenerateOilTransformerStatusSummary` | 生成油浸式变压器状态汇总 |
| `GenerateTemperatureSummary` | 生成温度汇总信息 |
| `GenerateDischargeSummary` | 生成局放汇总信息 |
| `GenerateOperationTimeSummary` | 生成操作时间汇总 |
| `GenerateUhfDischargeSummary` | 生成特高频局放汇总 |
| `GenerateSoundSummary` | 生成声纹汇总 |
| `GenerateHighFreqCurrentSummary` | 生成高频电流局放汇总 |
| `GenerateCoreGroundCurrentSummary` | 生成铁芯接地电流汇总 |

### 属性处理方法

| 方法 | 功能 |
|------|------|
| `ProcessRealDataByProperty` | 根据Property字段处理真实数据 |
| `ProcessTemperatureRange` | 处理温度范围数据 |
| `ProcessMovingContactTemperature` | 处理动触头温度数据 |
| `ProcessUltrasonicPeak` | 处理超声波峰值数据 |
| `ProcessTransientGroundPeak` | 处理暂态接地峰值数据 |
| `ProcessUhfPeak` | 处理特高频峰值数据 |
| `ProcessHighFreqCurrentPeak` | 处理高频电流峰值数据 |
| `ProcessOpenTime` | 处理分闸时间数据 |
| `ProcessCloseTime` | 处理合闸时间数据 |
| `ProcessSoundFingerprint` | 处理声纹数据 |
| `ProcessCoreGroundCurrent` | 处理铁芯接地电流数据 |

### 枚举处理方法

| 方法 | 功能 |
|------|------|
| `GetAlarmLevelText` | 获取告警级别文本 |
| `GetProcessingStatusText` | 获取处理状态文本 |
| `ParseEnumDescription` | 解析枚举描述（格式："0:分闸,1:合闸"） |
| `FormatInspectionResult` | 格式化巡视结果 |
| `ProcessInspectionResult` | 处理巡视记录检查结果 |

### 其他辅助方法

| 方法 | 功能 |
|------|------|
| `GetDeviceCategory` | 根据设备类型名称和设备名称确定设备种类 |
| `GenerateOperationSummary` | 生成运行状态汇总字符串 |

## 数据传输对象

### RequestReportDataDto - 报表请求参数

```csharp
public class RequestReportDataDto
{
    public string SubstationName { get; set; }   // 变电站名称
    public DateTime? StartTime { get; set; }      // 开始时间
    public DateTime? EndTime { get; set; }        // 结束时间
    public string Title { get; set; }            // 报表标题
}
```

### ReportResponse - 报表响应

```csharp
public class ReportResponse
{
    public bool success { get; set; }           // 是否成功
    public int code { get; set; }                // 响应码
    public string message { get; set; }          // 响应消息
    public object data { get; set; }             // 响应数据
}
```

### AlarmDeviceDto - 告警设备信息

```csharp
public class AlarmDeviceDto
{
    public int Index { get; set; }              // 序号
    public string AlarmTime { get; set; }        // 告警时间
    public string Location { get; set; }         // 位置（父级设备名称）
    public string Device { get; set; }           // 设备名称
    public string MonitorPointName { get; set; } // 监测点位名称
    public string Reason { get; set; }           // 告警原因
    public string Level { get; set; }            // 告警级别
    public string Count { get; set; }            // 告警总数
    public string MonitorPointAlarmCount { get; set; } // 监测点位告警数
    public string Status { get; set; }           // 处理状态
}
```

### NormalDeviceDto - 正常设备信息

```csharp
public class NormalDeviceDto
{
    public int Index { get; set; }                // 序号
    public string Location { get; set; }          // 位置
    public string Device { get; set; }            // 设备名称
    public List<string> OperationStatusSummary { get; set; } // 运行状态汇总
}
```

## 配置说明

### appsettings.json 配置项

```json
{
  "ExternalApi": {
    "ComprehensiveReportUrl": "https://external-api.com/reports/comprehensive",
    "ReportDataUrl": "https://external-api.com/reports/data"
  }
}
```

## 业务规则

### 1. 设备种类分类规则

- **GIS设备**：设备类型名称或设备名称包含"GIS中压交流柜"或"GIS"
- **接地变压器**：设备类型名称或设备名称包含"接地变压器"或"接地"
- **油浸式变压器**：设备类型名称或设备名称包含"油浸式变压器"或"油浸"
- **其他**：不匹配以上规则的设备

### 2. 告警级别映射

| 枚举值 | 描述 |
|-------|------|
| AlarmLevelEnum.Low | 一般 |
| AlarmLevelEnum.Medium | 重要 |
| AlarmLevelEnum.High | 紧急 |
| AlarmLevelEnum.Critical | 危急 |

### 3. 告警状态映射

| 枚举值 | 描述 |
|-------|------|
| AlarmStatusEnum.Pending | 待处理 |
| AlarmStatusEnum.Processing | 处理中 |
| AlarmStatusEnum.Resolved | 已解决 |
| AlarmStatusEnum.Closed | 已关闭 |

### 4. 数据类型处理规则

- **枚举类型**：Unit字段包含冒号（":"），格式如"0:分闸,1:合闸"
- **数值类型**：Unit字段为测量单位（如"℃"、"A"、"dB"），返回"值+单位"
- **字符串类型**：直接返回StrVal字段
- **执行失败**：Val为空且StrVal等于"执行失败"

### 5. 温度数据命名规则

- **进线温度**：Property为"JXA_Temperature"、"JXB_Temperature"、"JXC_Temperature"或"进线温度"
- **出线温度**：Property为"CXA_Temperature"、"CXB_Temperature"、"CXC_Temperature"或"出线温度"

### 6. 局放数据命名规则

- **超声波局放峰值**：Property为"ultrasonic_peak"、"ultrasonic_discharge_peak"或"超声波局放峰值"
- **暂态接地局放峰值**：Property为"transient_ground_peak"、"transient_discharge_peak"或"暂态接地局放峰值"
- **特高频局放峰值**：Property为"uhf_peak"、"uhf_discharge_peak"或"特高频局放峰值"
- **高频电流局放峰值**：Property为"high_freq_current_peak"或"高频电流局放峰值"

## 错误处理

### 1. 无数据情况

当查询结果为空时，返回：
```csharp
new ReportResponse
{
    success = true,
    code = 200,
    message = "该时间段内暂无数据下载到本地",
    data = null
};
```

### 2. 外部API调用失败

- 记录Warning日志，不抛出异常
- 让主流程继续执行
- 返回null或失败的ReportResponse

### 3. 数据解析异常

- 记录Warning日志
- 返回安全的默认值（"无数据"或原始值）

## 性能优化

### 1. 流式传输优先

- 优先使用HTTP Chunked Transfer-Encoding
- 减少内存占用
- 提高大报表传输效率

### 2. 数据分页查询

- 时序数据查询支持时间范围分页
- 避免一次性加载大量数据

### 3. 分组去重

- 设备列表按设备ID分组去重
- 告警数据按设备和监测点位分组
- 减少重复数据处理

## 日志记录

### 1. 操作日志

所有报表生成方法使用 `[OperLog]` 特性记录：
- 操作类型：`OperEnum.Export`
- 操作描述：如"生成综合运行报告"

### 2. 运行日志

```csharp
// 信息日志
_logger.LogInformation("准备发送综合运行报告数据到: {Url}", targetUrl);
_logger.LogInformation("HTTP Chunked Transfer-Encoding流式传输成功");

// 警告日志
_logger.LogWarning("未配置外部API地址，跳过数据发送");
_logger.LogWarning("HTTP Chunked Transfer-Encoding流式传输失败: {Message}", message);

// 错误日志
_logger.LogError(ex, "发送综合运行报告数据失败: {Message}", ex.Message);
```

## 相关服务

- `IStreamingTransferService` - 流式传输服务
- `IPointDataRepository` - 时序数据仓储
- `AlarmService` - 告警服务
- `PatrolService` - 巡检服务

## 相关实体

- `SubstationAggregateRoot` - 变电站聚合根
- `MonitoredObjectAggregateRoot` - 监测对象聚合根
- `AlarmRecordAggregateRoot` - 告警记录聚合根
- `PatrolRecordEntity` - 巡检记录实体
- `PointDataEntity` - 时序数据点实体

## API端点

### 生成综合运行报告

```
POST /api/app/ast-intellisub/report/comprehensive-operation-report
```

**请求体**：
```json
{
  "substationName": "小东庄主变电所",
  "startTime": "2026-06-01T00:00:00",
  "endTime": "2026-06-04T23:59:59",
  "title": "综合运行报告"
}
```

**响应**：
```json
{
  "success": true,
  "code": 200,
  "message": "报表生成成功",
  "data": {
    "reportTitle": "小东庄主变电所综合运行报告",
    "reportPeriodStart": "2026-06-01 00:00:00",
    "reportPeriodEnd": "2026-06-04 23:59:59",
    "deviceTotal": 50,
    "gisCount": 20,
    "groundTransformerCount": 15,
    "oilTransformerCount": 15,
    "alarmCount": 2,
    "normalCount": 48,
    "summary": "2台设备告警、48台设备正常",
    "alarmDevices": [...],
    "normalDevices": [...]
  }
}
```

## 业务价值

1. **综合监控** - 提供变电站全方位的运行状态视图
2. **决策支持** - 为运维决策提供数据支撑
3. **数据集成** - 整合多个数据源，提供统一报表
4. **外部协作** - 支持与外部系统的数据对接
5. **合规审计** - 记录完整的运行历史，支持审计要求

---

> **最后更新**：2026-06-04
> **源码位置**：`module/ast-intellisub/Ast.IntelliSub.Application/Services/ReportService.cs`
> **接口定义**：`module/ast-intellisub/Ast.IntelliSub.Application.Contracts/IServices/IReportService.cs`
