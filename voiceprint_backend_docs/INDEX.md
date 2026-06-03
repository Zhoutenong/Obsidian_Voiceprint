# IntelliSubstation Voiceprint Backend 知识图谱

基于双向链接的 .NET 8 ABP 框架智能变电站监控系统知识网络。

---

## 项目概述

**IntelliSubstation Voiceprint Monitoring Backend** — 基于 .NET 8 ABP 框架的智能变电站监控系统，集成了声纹分析、IEC61850 数据上报和实时设备健康监测。

**技术栈：**
- .NET 8 / ABP Framework
- SqlSugar ORM (多数据库支持)
- Hangfire (后台任务)
- Autofac (DI 容器)
- JWT + OAuth 认证
- SignalR (实时告警中心)

**源码位置**：`e:\Code\myCode\VoicePrint\intelli-substation-voiceprint-backend\`

---

## 快速导航

### 📋 学习路线

- [[ABP框架入门]] — ABP Framework 基础知识
- [[项目架构概览]] — 整体架构设计
- [[Modules/ast-intellisub/变电站监视范围概览]] — 模块划分、数据驱动方式与监视内容总览
- [[数据库设计]] — 数据库结构说明

### 🔧 核心模块

| 模块 | 说明 | 状态 |
|------|------|------|
| `rbac` | 基于角色的访问控制 | 🟢 已完成 |
| `ast-intellisub` | 智能巡检/变电站监控 | 🟢 已完成 |
| `ast-intellisubdata` | 传感器时序数据 | 🟢 已完成 |
| `ast-voiceprint` | 声纹分析 | 🟢 已完成 |
| `isapi` | IEC61850/IEC104 集成 | 🟢 已完成 |
| `audit-logging` | 审计日志 | 🟡 学习中 |
| `tenant-management` | 多租户管理 | 🟡 学习中 |
| `setting-management` | 系统设置 | 🟡 学习中 |

---

## 目录结构

```
Classes/            核心实体文档
  ├─ Device.md      设备实体
  ├─ Alarm.md       告警实体
  ├─ Voiceprint.md  声纹实体
  └─ PointData.md   点位数据实体
Modules/            模块文档（按业务模块组织）
  ├─ rbac/          认证授权模块
  │   ├─ 认证系统.md
  │   ├─ 用户管理API.md
  │   ├─ 权限定义服务.md
  │   └─ RbacOverview.md
  ├─ ast-intellisub/ 变电站监控模块
  │   ├─ 变电站监视范围概览.md  模块/数据流/监视内容总览
  │   ├─ 设备管理服务.md
  │   ├─ 设备告警流程.md
  │   ├─ Patrol/      巡检系统
  │   │   ├─ PatrolTaskService.md
  │   │   ├─ PatrolExecutionService.md
  │   │   └─ PatrolRecordService.md
  │   ├─ Monitoring/   监测点位
  │   │   ├─ MonitoredPointService.md
  │   │   ├─ MonitoredObjectService.md
  │   │   ├─ MonitoredItemService.md
  │   │   ├─ MonitoredObjectAttrService.md
  │   │   ├─ MonitoredObjectTypeService.md
  │   │   ├─ RealtimeMonitoringPointService.md
  │   │   ├─ SensorService.md
  │   │   └─ MonitoredPointAlarmCategoryRelService.md
  │   ├─ Gateway/      网关/MQTT
  │   │   ├─ GatewayService.md
  │   │   ├─ MqttService.md
  │   │   └─ StreamingGatewayService.md
  │   ├─ DataBinding/  数据绑定
  │   │   ├─ DataBindingItemService.md
  │   │   ├─ DataBindingTypeService.md
  │   │   ├─ DisplayComponentService.md
  │   │   ├─ DataStrategyService.md
  │   │   ├─ StrategyStateService.md
  │   │   ├─ BindingItemStrategyRelService.md
  │   │   ├─ PointValueProcessingService.md
  │   │   ├─ PointValueCacheService.md
  │   │   └─ PointBindingRelService.md
  │   ├─ Alarm/        告警系统
  │   │   ├─ AlarmNotificationHub.md
  │   │   ├─ AlarmNotificationService.md
  │   │   ├─ AlarmCategoryService.md
  │   │   ├─ AlarmRecordService.md
  │   │   ├─ AlarmProcessingService.md
  │   │   ├─ CollectorService.md
  │   │   └─ AlarmStrategyOverview.md
  │   ├─ Camera/      摄像机/流媒体
  │   │   ├─ CameraService.md
  │   │   ├─ MediaService.md
  │   │   ├─ StreamingApiService.md
  │   │   ├─ PresetService.md
  │   │   ├─ NvrService.md
  │   │   └─ StreamingTransferService.md
  │   ├─ AI/          AI 识别
  │   │   ├─ AIRecognitionService.md
  │   │   └─ AlgorithmService.md
  │   ├─ Camera/      摄像机/流媒体
  │   │   ├─ CameraService.md
  │   │   ├─ MediaService.md
  │   │   ├─ StreamingApiService.md
  │   │   ├─ PresetService.md
  │   │   ├─ NvrService.md
  │   │   ├── StreamingTransferService.md
  │   │   └─ CameraResourceManager.md
  │   ├─ Substation/   变电站管理
  │   │   ├─ SubstationService.md
  │   │   ├─ SubstationUserService.md
  │   │   └─ SubstationTypeService.md
  │   ├─ Streaming/   流媒体
  │   │   ├─ StreamingGatewayService.md
  │   │   ├─ NvrService.md
  │   │   └─ StreamingTransferService.md
  │   ├─ DataCollection/ 数据采集
  │   │   └─ DataCollectionService.md
  │   ├─ Report/      报告服务
  │   │   └─ ReportService.md
  │   └─ Analysis/     系统分析
  │       ├─ SystemAnalysisService.md
  │       └─ SystemStatisticsService.md
  ├─ ast-intellisubdata/ 传感器数据模块
  │   ├─ README.md
  │   ├─ AstPointDataService.md
  │   ├─ ChildTableNameManager.md
  │   ├─ TimeSeriesDbHelper.md
  │   ├─ LargeTextStorageManager.md
  │   ├─ PointData.md
  │   ├─ 传感器数据管理服务.md
  │   ├─ 数据清理任务.md
  │   ├─ TDengine集成.md
  │   └─ 数据上报流程.md
  ├─ ast-voiceprint/ 声纹分析模块
  │   ├─ README.md
  │   ├─ VoiceprintCaptureRuntimeStateService.md
  │   ├─ 声纹采集服务.md
  │   └─ 声纹分析流程.md
  ├─ isapi/         工业协议集成模块
  │   ├─ ISAPIService.md
  │   ├─ Iec61850ApiService.md
  │   ├─ IEC61850数据上报服务.md
  │   └─ 数据上报流程.md
  ├─ audit-logging/  审计日志模块
  │   └─ AuditLoggingOverview.md
  ├─ tenant-management/ 多租户模块
  │   └─ TenantManagementOverview.md
  ├─ setting-management/ 系统设置模块
  │   └─ SettingManagementOverview.md
  └─ System/        系统基础服务
      ├─ EnumService.md
      ├─ FileService.md
      └─ DataIntegrityCheckService.md
API/                API 接口文档
  ├─ Account/        账户 API
  │   └─ AccountAPI.md
  ├─ Alarm/          告警 API
  │   └─ AlarmAPI.md
  ├─ Dashboard/      仪表板 API
  │   └─ DashboardAPI.md
  ├─ Assets/         设备台账 API
  │   └─ AssetsAPI.md
  ├─ Voiceprint/     声纹 API
  │   └─ VoiceprintAPI.md
  ├─ VoiceprintAudio/ 声纹音频内部 API
  │   ├─ VoiceprintAudioInternalAPI.md
  │   └─ VoiceprintAudioUploadAPI.md
  ├─ Report/         报告 API
  │   └─ ReportAPI.md
  ├─ PointData/      传感器数据 API
  │   └─ PointDataAPI.md
  └─ IntelliSub/     智能变电站 API
      └─ IntelliSubAPI.md
Pipelines/          数据流程文档
  ├─ Audio/         音频处理流程
  │   ├─ 音频处理管道.md
  │   └─ 音频采集流程.md
  ├─ DataReport/    数据上报流程
  │   └─ IEC61850上报管道.md
  └─ Events/        事件驱动
      └─ EventDrivenPipeline.md
Hangfire/           后台任务文档
  ├─ Voiceprint/    声纹相关 Jobs
  │   └─ VoiceprintCaptureJob.md
  ├─ System/        系统相关 Jobs
  │   ├─ PatrolJobManager.md
  │   ├─ VoiceprintProcessedCleanupJob.md
  │   └─ PatrolSystemCleanupJob.md
  ├─ Data/          数据相关 Jobs
  │   ├─ PointDataCleanupJob.md
  │   ├─ PointValueCacheCleanupJob.md
  │   ├─ InfraredTemperatureCollectionJob.md
  │   └─ EnvironmentDetectionCollectionJob.md
  ├─ Gateway/       网关相关 Jobs
  │   ├─ GatewaySyncJob.md
  │   └─ VisualGatewayHealthCheckJob.md
  └─ Camera/        摄像机相关 Jobs
      ├─ CameraResourceCleanupJob.md
      └─ ISAPIResourceCleanupJob.md
Concepts/           架构概念与设计模式
  ├─ ABP仓储模式.md
  └─ DDD分层架构.md
Architecture/       架构文档
  ├─ 整体架构设计.md
  ├─ 模块依赖关系.md
  └─ 数据库设计.md
Diagrams/           架构图和流程图
  ├─ 01-用户登录认证流程.md
  ├─ 02-告警管理流程.md
  ├─ 03-仪表板与巡视流程.md
  ├─ 04-设备台账查询流程.md
  ├─ 05-声纹分析流程.md
  └─ 图表索引.md
Issues/             Bug 案例与调试
  ├─ ARM32堆栈溢出问题.md
  ├─ 连续告警误报问题.md
  ├─ TDengine连接超时问题.md
  └─ 调试指南.md
Configuration/      配置文档
  ├─ AppSettingsIndex.md
  └─ 配置快速参考.md
Resources/          外部资源索引
  └─ ExternalDocsIndex.md
    链接到外部实现方案：
    - [[../../../voiceprint/00-系统架构与功能实现.md|系统架构实现]]
    - [[../../../voiceprint/02-python-voiceprint-service方案.md|Python 声纹服务]]
    - [[../../../voiceprint/03-raspberry-pi-agent方案.md|树莓派边缘采集]]
    - [[../../../data-flow-diagrams/|前端 API 时序图（6 个 HTML）]]
    - [[../../../frontend-api-interfaces.md|前端 23 个接口]]
Templates/          笔记模板
  ├─ 组件笔记模板.md
  ├─ 流程文档模板.md
  ├─ 概念文档模板.md
  ├─ API文档模板.md
  └─ Hangfire任务模板.md
DocumentationPlan.md  文档补充计划
快速导航.md          快速导航
README.md           项目说明
INDEX.md            本文件（知识图谱索引）
```

---

## API 文档索引

### 账户与认证
[[API/Account/AccountAPI]] — 用户登录、登出、密码管理

### 告警管理
[[API/Alarm/AlarmAPI]] — 告警列表、月度统计、告警处理

### 仪表板
[[API/Dashboard/DashboardAPI]] — 巡视统计、系统状态、首页告警

### 设备台账
[[API/Assets/AssetsAPI]] — 设备台账树、设备信息、运行趋势

### 声纹分析
[[API/Voiceprint/VoiceprintAPI]] — 标准音频库、测试音频、报告生成
[[API/VoiceprintAudio/VoiceprintAudioInternalAPI]] — 音频记录、处理、下载

### 传感器数据
[[API/PointData/PointDataAPI]] — 最新数据、历史数据、批量查询

### 报告管理
[[API/Report/ReportAPI]] — 报告导出、模板管理

### 智能变电站
[[API/IntelliSub/IntelliSubAPI]] — 变电站综合接口

---

## 后台任务索引 (Hangfire)

### 系统任务
[[Hangfire/System/PatrolJobManager]] — 巡检任务调度
[[Hangfire/System/VoiceprintProcessedCleanupJob]] — 音频清理
[[Hangfire/System/PatrolSystemCleanupJob]] — 巡检清理
[[Hangfire/System/VisualGatewayHealthCheckJob]] — 网关健康检查

### 数据任务
[[Hangfire/Data/PointDataCleanupJob]] — 点位数据清理
[[Hangfire/Data/PointValueCacheCleanupJob]] — 缓存清理
[[Hangfire/Data/InfraredTemperatureCollectionJob]] — 红外温度采集
[[Hangfire/Data/EnvironmentDetectionCollectionJob]] — 环境监测采集

### 网关任务
[[Hangfire/Gateway/GatewaySyncJob]] — 网关同步

### 摄像机任务
[[Hangfire/Camera/CameraResourceCleanupJob]] — 摄像机资源清理
[[Hangfire/Camera/ISAPIResourceCleanupJob]] — ISAPI 资源清理

### 声纹任务
[[Hangfire/Voiceprint/VoiceprintCaptureJob]] — 声纹采集

---

## 数据流程管道

### 音频处理
[[Pipelines/Audio/音频处理管道]] — 音频处理完整流程
[[Pipelines/Audio/音频采集流程]] — 音频采集详细步骤

### 数据上报
[[Pipelines/DataReport/IEC61850上报管道]] — IEC61850 数据上报流程

### 事件驱动
[[Pipelines/Events/EventDrivenPipeline]] — 事件驱动架构管道

---

## 模块详细分类

### ✅ RBAC 模块（认证授权）

#### 概览
[[Modules/rbac/RbacOverview]] — RBAC 模块概述

#### 核心
[[Modules/rbac/认证系统]] · [[Modules/rbac/权限定义服务]] · [[Modules/rbac/用户管理API]]

#### 用户管理
[[用户管理]] · [[角色管理]] · [[权限管理]]

---

### ✅ ast-intellisub 模块（变电站监控）

#### 概览
[[Modules/ast-intellisub/变电站监视范围概览]] — 主要模块、数据驱动方式、监视对象与物理量

#### 设备管理
[[Modules/ast-intellisub/设备管理服务]] · [[Classes/Device]]

#### 巡检系统
[[Modules/ast-intellisub/Patrol/PatrolTaskService]] · [[Modules/ast-intellisub/Patrol/PatrolExecutionService]] · [[Modules/ast-intellisub/Patrol/PatrolRecordService]]

#### 监测点位
[[Modules/ast-intellisub/Monitoring/MonitoredPointService]] · [[Modules/ast-intellisub/Monitoring/MonitoredObjectService]] · [[Modules/ast-intellisub/Monitoring/SensorService]]

#### 网关/MQTT
[[Modules/ast-intellisub/Gateway/GatewayService]] · [[Modules/ast-intellisub/Gateway/MqttService]]

#### 数据绑定
[[Modules/ast-intellisub/DataBinding/DataBindingItemService]] · [[Modules/ast-intellisub/DataBinding/DataStrategyService]]

#### 告警系统
[[Modules/ast-intellisub/设备告警流程]] · [[Modules/ast-intellisub/Alarm/AlarmNotificationHub]] · [[Modules/ast-intellisub/Alarm/AlarmNotificationService]] · [[Modules/ast-intellisub/Alarm/AlarmCategoryService]] · [[Classes/Alarm]]

#### 摄像机集成
[[Modules/ast-intellisub/Camera/CameraService]] · [[Modules/ast-intellisub/Camera/MediaService]] · [[Modules/ast-intellisub/Camera/StreamingApiService]] · [[Modules/ast-intellisub/Camera/PresetService]]

#### AI 识别
[[Modules/ast-intellisub/AI/AIRecognitionService]]

#### 系统分析
[[Modules/ast-intellisub/Analysis/SystemAnalysisService]] · [[Modules/ast-intellisub/Analysis/SystemStatisticsService]]

---

### ✅ ast-intellisubdata 模块（时序数据）

#### 概览
[[Modules/ast-intellisubdata/传感器数据管理服务]] · [[Classes/PointData]]

#### 数据采集
[[Modules/ast-intellisubdata/数据上报流程]]

#### 数据存储
[[Modules/ast-intellisubdata/TDengine集成]]

#### 数据清理
[[Modules/ast-intellisubdata/数据清理任务]]

---

### ✅ ast-voiceprint 模块（声纹分析）

#### 音频处理
[[Modules/ast-voiceprint/声纹采集服务]] · [[Modules/ast-voiceprint/声纹分析流程]] · [[Classes/Voiceprint]]

#### 流程文档
[[Pipelines/Audio/音频处理管道]] · [[Pipelines/Audio/音频采集流程]]

---

### ✅ isapi 模块（工业协议）

#### IEC61850
[[Modules/isapi/IEC61850数据上报服务]] · [[Modules/isapi/数据上报流程]] · [[Pipelines/DataReport/IEC61850上报管道]]

#### IEC104
[[IEC104协议]] · [[数据解析]]

---

## 按类型查看

### 应用服务 (AppService)
```dataview
TABLE without id
  file.link as "服务",
  module as "模块",
  status as "状态"
FROM #appservice
WHERE type = "component"
SORT file.name ASC
```

### 领域实体 (Entity)
```dataview
TABLE without id
  file.link as "实体",
  module as "模块",
  status as "状态"
FROM #entity
WHERE type = "component"
SORT file.name ASC
```

### API 端点
```dataview
TABLE without id
  file.link as "API",
  method as "方法",
  path as "路径"
FROM #api
SORT file.name ASC
```

### 流程文档
```dataview
TABLE without id
  file.link as "流程",
  module as "模块",
  status as "状态"
FROM #flow
SORT file.name ASC
```

---

## 基础设施模块

### 审计日志
[[Modules/audit-logging/AuditLoggingOverview]]

### 多租户
[[Modules/tenant-management/TenantManagementOverview]]

### 系统设置
[[Modules/setting-management/SettingManagementOverview]]

---

## 架构概念

### 架构文档
[[Architecture/整体架构设计]] · [[Architecture/模块依赖关系]] · [[Architecture/数据库设计]]

### 设计模式
[[Concepts/ABP仓储模式]] · [[Concepts/DDD分层架构]]

---

## 后台任务 (Hangfire)

### 定时任务
| 任务 | Cron表达式 | 说明 | 状态 |
|------|-----------|------|------|
| VoiceprintCaptureJob | `0 */10 * * * *` | 音频采集 | 🟢 |
| VoiceprintProcessedCleanupJob | 手动 | 清理已处理音频 | 🟢 |
| PointDataCleanupJob | 可配置 | 数据清理 | 🟢 |
| PatrolJobManager | 可配置 | 巡检任务调度 | 🟢 |

```dataview
LIST
FROM #hangfire
SORT file.name ASC
```

---

## 架构概念

### DDD 分层
[[应用层]] · [[领域层]] · [[基础设施层]] · [[表示层]]

### ABP 特性
[[模块系统]] · [[依赖注入]] · [[仓储模式]] · [[工作单元]]

### 数据库
[[CodeFirst迁移]] · [[多数据库支持]] · [[SqlSugar配置]]

### 认证授权
[[JWT认证]] · [[OAuth集成]] · [[权限定义]]

---

## 配置说明

### 部署模式
- **LowResource** (默认) — SQLite + Memory 存储，ARM32/边缘设备优化
- **HighPerformance** — 完整数据库 + Redis/TDengine，数据中心部署

### 配置文件结构
```json
{
  "DbConnOptions": {
    "DeploymentMode": "LowResource",
    "Url": "Data Source=db/ast_intellisub.db",
    "DbType": "Sqlite"
  },
  "Hangfire": {
    "StorageMode": "Memory"
  },
  "VoiceprintCaptureJob": {
    "Enabled": true,
    "CronExpression": "0 */10 * * * *"
  },
  "Iec61850": {
    "EnableDataReport": false,
    "ModelFiles": [...]
  }
}
```

---

## 学习资源

- [[DocumentationPlan]] — 文档补充计划
- [[项目CLAUDE.md]] — 项目开发指南
- [[快速导航]] — 快速导航索引
- [[Diagrams/图表索引]] — 流程图索引

---

## 最近更新

```dataview
LIST
FROM -Templates AND -Resources
SORT mtime DESC
LIMIT 20
```

---

## 标签索引

```dataview
TABLE without id
  tag as "标签",
  count(rows) as "文档数"
FROM "Modules"
FLATTEN tags
WHERE tag != ""
GROUP BY tag
SORT tag ASC
```

---

## 状态统计

```dataview
TABLE without ID
  status as "状态",
  count(rows) as "数量"
FROM -Templates AND -Resources
GROUP BY status
```

---

> 创建时间：2026-06-02
> 源码：`e:\Code\myCode\VoicePrint\intelli-substation-voiceprint-backend\`
> 框架：.NET 8 + ABP Framework
