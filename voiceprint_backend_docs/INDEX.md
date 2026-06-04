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

## 📊 文档覆盖率（2026-06-04 更新）

| 维度 | 估计覆盖 | 说明 |
|------|---------|------|
| **Domain 实体** | ~75% ⬆️ | 45+ 篇实体文档（从62%提升） |
| **Application 服务** | ~92% | 69+ 篇服务文档 |
| **Hangfire Jobs** | 100% | 11 个 Job 全部文档化 |
| **EventHandler** | ~95% ⬆️ | 15 篇事件处理器文档（从60%提升） |
| **框架/宿主** | 100% | 12 篇框架组件文档 |
| **管理端 API** | 100% | 4 篇管理端 API 汇总 |
| **全项目覆盖** | ~85%+ ⬆️ | 核心功能完整，可选扩展待补 |

> **详细补齐计划**：参见 [[文档补齐计划2026.md]]

---

### 📋 学习路线

- [[Architecture/整体架构设计]] — 整体架构设计
- [[Architecture/模块依赖关系]] — 模块依赖关系
- [[Modules/ast-intellisub/变电站监视范围概览]] — 模块划分、数据驱动方式与监视内容总览
- [[Architecture/数据库设计]] — 数据库结构说明

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
Classes/            核心实体文档（45+ 篇，覆盖率 ~75%）
  ├─ Alarm/         告警实体
  │   ├─ AlarmRecordAggregateRoot.md      ✅ 告警记录聚合根
  │   ├─ ProcessingRecordEntity.md        ✅ 告警处理记录
  │   ├─ AlarmCategoryAggregateRoot.md   ✅ 告警分类聚合根
  │   └─ AlarmRecordItemEntity.md        ✅ 告警记录项
  ├─ Monitoring/    监测实体
  │   ├─ MonitoredPointEntity.md         ✅ 监测点位
  │   ├─ MonitoredObjectAggregateRoot.md  ✅ 监测对象聚合根
  │   ├─ MonitoredObjectTypeEntity.md    ✅ 监测对象类型
  │   ├─ MonitoredObjectItemRelEntity.md ✅ 对象-监测项关联
  │   ├─ MonitoredObjectAttrGroupEntity.md ✅ 属性分组
  │   ├─ MonitoredObjectPresetRelEntity.md ✅ 对象-预置位关联
  │   └─ MonitoredPointAlarmCategoryRelEntity.md ✅ 点位-告警分类关联
  ├─ Gateway/       网关实体
  │   ├─ GatewayAggregateRoot.md         ✅ 网关聚合根
  │   ├─ AstPointEntity.md               ✅ 传感器点位实体
  │   ├─ SensorEntity.md                 ✅ 传感器实体
  │   ├─ DeviceEntity.md                 ✅ 设备实体
  │   └─ NvrEntity.md                    ✅ NVR实体
  ├─ DataBinding/   数据绑定实体
  │   ├─ DataBindingItemEntity.md        ✅ 绑定项实体
  │   ├─ DataStrategyEntity.md           ✅ 数据策略
  │   ├─ StrategyStateEntity.md          ✅ 策略状态
  │   ├─ DisplayComponentEntity.md       ✅ 展示组件
  │   ├─ BindingItemStrategyRelEntity.md ✅ 绑定项-策略关联
  │   └─ PointBindingRelEntity.md        ✅ 点位绑定关联
  ├─ Patrol/        巡检实体
  │   ├─ PatrolTaskEntity.md            ✅ 巡检任务
  │   ├─ PatrolRecordEntity.md          ✅ 巡检记录
  │   └─ PatrolTaskMonitoredPointRelEntity.md ✅ 任务-点位关联
  ├─ Video/         视频实体
  │   ├─ VideoLayoutDetailEntity.md     ✅ 视频布局详情
  │   └─ VideoLayoutFavoriteEntity.md    ✅ 视频布局收藏
  ├─ Voiceprint/    声纹实体
  │   ├─ VoiceprintAlarmRecordEntity.md  ✅ 声纹告警记录
  │   ├─ VoiceprintDeviceAudioRecordEntity.md ✅ 设备音频记录
  │   ├─ VoiceprintStandardAudioEntity.md ✅ 标准音频
  │   └─ VoiceprintCollectorLogEntity.md ✅ 采集通信日志
  ├─ Substation/    变电站实体
  │   ├─ SubstationAggregateRoot.md      ✅ 变电站聚合根
  │   ├─ SubstationTypeEntity.md         ✅ 变电站类型
  │   └─ SubstationUserEntity.md         ✅ 变电站用户
  ├─ Camera/        摄像机实体
  │   └─ DeviceEntity.md                 ✅ 设备实体
  └─ Rbac/          RBAC实体
      ├─ UserEntity.md                   ✅ 用户实体
      ├─ RoleEntity.md                   ✅ 角色实体
      ├─ MenuEntity.md                   ✅ 菜单实体
      ├─ DeptEntity.md                   ✅ 部门实体
      ├─ PostEntity.md                   ✅ 岗位实体
      ├─ DictionaryTypeEntity.md         ✅ 字典类型
      ├─ DictionaryDataEntity.md         ✅ 字典数据
      ├─ UserRoleEntity.md               ✅ 用户-角色关联
      ├─ UserPostEntity.md               ✅ 用户-岗位关联
      ├─ RoleMenuEntity.md               ✅ 角色-菜单关联
      └─ RoleDeptEntity.md               ✅ 角色-部门关联
Modules/            模块文档（按业务模块组织）
  ├─ rbac/          认证授权模块
  │   ├─ 认证系统.md        ✅
  │   ├─ 用户管理API.md     ✅
  │   ├─ 权限定义服务.md    ✅
  │   └─ RbacOverview.md    ✅
  ├─ ast-intellisub/ 变电站监控模块
  │   ├─ 变电站监视范围概览.md  ✅ 模块/数据流/监视内容总览
  │   ├─ 设备管理服务.md         ✅
  │   ├─ 设备告警流程.md         ✅
  │   ├─ Patrol/      巡检系统
  │   │   ├─ PatrolTaskService.md           ✅
  │   │   ├─ PatrolExecutionService.md      ✅
  │   │   ├─ PatrolRecordService.md         ✅
  │   │   └─ PatrolSystemRecoveryService.md ⬜ 规划中
  │   ├─ Monitoring/   监测点位
  │   │   ├─ MonitoredPointService.md        ✅
  │   │   ├─ MonitoredObjectService.md      ✅
  │   │   ├─ MonitoredItemService.md        ✅
  │   │   ├─ MonitoredObjectAttrService.md   ✅
  │   │   ├─ MonitoredObjectTypeService.md   ✅
  │   │   ├─ RealtimeMonitoringPointService.md ✅
  │   │   ├─ SensorService.md               ✅
  │   │   └─ MonitoredPointAlarmCategoryRelService.md ✅
  │   ├─ Gateway/      网关/MQTT
  │   │   ├─ GatewayService.md              ✅
  │   │   ├─ MqttService.md                 ✅
  │   │   ├─ StreamingGatewayService.md     ✅
  │   │   └─ MqttMessageBackgroundService.md ✅
  │   ├─ DataBinding/  数据绑定
  │   │   ├─ DataBindingItemService.md     ✅
  │   │   ├─ DataBindingTypeService.md      ✅
  │   │   ├─ DisplayComponentService.md     ✅
  │   │   ├─ DataStrategyService.md         ✅
  │   │   ├─ StrategyStateService.md        ✅
  │   │   ├─ BindingItemStrategyRelService.md ✅
  │   │   ├─ PointValueProcessingService.md ✅
  │   │   ├─ PointValueCacheService.md      ✅
  │   │   └─ PointBindingRelService.md      ✅
  │   ├─ Alarm/        告警系统
  │   │   ├─ AlarmNotificationHub.md       ✅
  │   │   ├─ AlarmNotificationService.md   ✅
  │   │   ├─ AlarmCategoryService.md       ✅
  │   │   ├─ AlarmRecordService.md          ✅
  │   │   ├─ AlarmProcessingService.md     ✅
  │   │   ├─ CollectorService.md           ✅
  │   │   └─ AlarmStrategyOverview.md      ✅
  │   ├─ Camera/      摄像机/流媒体
  │   │   ├─ CameraService.md               ✅
  │   │   ├─ MediaService.md                ✅
  │   │   ├─ StreamingApiService.md         ✅
  │   │   ├─ PresetService.md               ✅
  │   │   ├─ NvrService.md                  ⬜ 规划中
  │   │   └─ CameraResourceManager.md       ✅
  │   ├─ AI/          AI 识别
  │   │   ├─ AIRecognitionService.md        ✅
  │   │   ├─ AlgorithmService.md            ✅
  │   │   └─ RecognitionApiService.md       ✅
  │   ├─ Substation/   变电站管理
  │   │   ├─ SubstationService.md           ✅
  │   │   ├─ SubstationUserService.md       ✅
  │   │   └─ SubstationTypeService.md       ✅
  │   ├─ Streaming/   流媒体
  │   │   ├─ StreamingGatewayService.md     ✅
  │   │   ├─ NvrService.md                  ✅
  │   │   └─ StreamingTransferService.md    ✅
  │   ├─ DataCollection/ 数据采集
  │   │   └─ DataCollectionService.md       ✅
  │   ├─ Report/      报告服务
  │   │   └─ ReportService.md               ✅ 综合报表服务
  │   └─ Analysis/     系统分析
  │       ├─ SystemAnalysisService.md      ✅
  │       └─ SystemStatisticsService.md     ✅
  ├─ ast-intellisubdata/ 传感器数据模块
  │   ├─ README.md                        ✅
  │   ├─ AstPointDataService.md           ✅
  │   ├─ ChildTableNameManager.md         ✅
  │   ├─ TimeSeriesDbHelper.md            ✅
  │   ├─ LargeTextStorageManager.md       ✅
  │   ├─ PointData.md                     ✅
  │   ├─ 传感器数据管理服务.md            ✅
  │   ├─ 数据清理任务.md                  ✅
  │   ├─ TDengine集成.md                  ✅
  │   └─ 数据上报流程.md                  ✅
  ├─ ast-voiceprint/ 声纹分析模块
  │   ├─ README.md                        ✅
  │   ├─ VoiceprintCaptureRuntimeStateService.md ✅
  │   ├─ 声纹采集服务.md                  ✅
  │   └─ 声纹分析流程.md                  ✅
  ├─ isapi/         工业协议集成模块
  │   ├─ ISAPIService.md                  ✅
  │   ├─ Iec61850ApiService.md            ✅
  │   ├─ IEC61850数据上报服务.md          ✅
  │   └─ 数据上报流程.md                  ✅
  ├─ audit-logging/  审计日志模块
  │   └─ AuditLoggingOverview.md          ✅
  ├─ tenant-management/ 多租户模块
  │   └─ TenantManagementOverview.md      ✅
  ├─ setting-management/ 系统设置模块
  │   └─ SettingManagementOverview.md     ✅
  └─ System/        系统基础服务
      ├─ EnumService.md                   ✅
      ├─ FileService.md                   ✅
      ├─ DataIntegrityCheckService.md     ✅
      └─ ApplicationStartupService.md     ✅
Services/           应用服务实现级文档
  ├─ ReportService.md                   ✅ 综合报表服务（3000+行核心服务）
  └─ PointDataService.md                ✅ 点位数据API服务
Managers/          领域管理器文档（Domain Manager层）
  ├─ rbac/          RBAC管理器
  │   ├─ AccountManager.md              ✅ 账户管理器（JWT令牌、认证）
  │   ├─ UserManager.md                  ✅ 用户管理器（创建、角色分配）
  │   ├─ RoleManager.md                  ✅ 角色管理器（菜单分配）
  │   ├─ FileManager.md                  ✅ 文件管理器（上传、缩略图）
  │   └─ TencentCloudManager.md           ✅ 腾讯云服务管理器（短信）
  └─ ast-intellisub/ 变电站监控管理器
      ├─ AlarmConditionEvaluationManager.md ✅ 告警条件评估管理器
      ├─ Iec61850MappingManager.md       ✅ IEC61850映射管理器
      └─ StreamingUrlManager.md          ✅ 流媒体URL管理器
  └─ SettingManagement/ 系统设置管理器
      └─ SettingManager.md               ✅ 设置管理器（继承、加密）
API/                API 接口文档
  ├─ Account/        账户 API
  │   └─ AccountAPI.md                   ✅
  ├─ Alarm/          告警 API
  │   └─ AlarmAPI.md                     ✅
  ├─ Dashboard/      仪表板 API
  │   └─ DashboardAPI.md                 ✅
  ├─ Assets/         设备台账 API
  │   └─ AssetsAPI.md                    ✅
  ├─ Voiceprint/     声纹 API
  │   └─ VoiceprintAPI.md                ✅
  ├─ VoiceprintAudio/ 声纹音频内部 API
  │   ├─ VoiceprintAudioInternalAPI.md   ✅
  │   └─ VoiceprintAudioUploadAPI.md     ✅
  ├─ Report/         报告 API
  │   └─ ReportAPI.md                    ✅
  ├─ PointData/      传感器数据 API
  │   └─ PointDataAPI.md                 ✅
  ├─ Admin/         管理端 API（RBAC）
  │   ├─ AdminAPIIndex.md                ✅ RBAC管理端API汇总
  │   ├─ UserRoleMenuAPI.md              ✅ 用户角色菜单API
  │   ├─ SystemConfigAPI.md              ✅ 系统配置API
  │   └─ MonitorAPI.md                   ✅ 系统监控API
  └─ IntelliSub/     智能变电站 API
      └─ IntelliSubAPI.md                ✅
Pipelines/          数据流程文档
  ├─ Audio/         音频处理流程
  │   ├─ 音频处理管道.md                  ✅
  │   └─ 音频采集流程.md                  ✅
  ├─ DataReport/    数据上报流程
  │   └─ IEC61850上报管道.md              ✅
  └─ Events/        事件驱动
      ├─ EventDrivenPipeline.md          ✅
      └─ Handlers/     事件处理器
          ├─ EventHandlerOverview.md    ✅ 事件处理器总览
          ├─ AlarmProcessingService.md       ✅ 告警处理服务
          ├─ AlarmRecordItemCreatedHandler.md ✅ 告警记录创建事件
          ├─ PointValueEventHandler.md    ✅ 点位值事件
          ├─ ProcessedPointValueEventHandler.md ✅ 处理后点位值事件
          ├─ NormalDataProcessedHandler.md ✅ 正常数据处理事件
          ├─ PatrolResultUpdateHandler.md   ✅ 巡检结果更新处理器
          └─ Iec61850DataReportHandler.md ✅ IEC61850数据上报事件
          ├─ GatewayStatusChangedHandler.md ✅ 网关状态变更事件
          ├─ DeviceStatusEventHandler.md ✅ 设备状态事件
          ├─ DeviceMetadataEventHandler.md ✅ 设备元数据事件
          ├─ Iec61850AlarmReportHandler.md ✅ IEC61850告警上报事件
          ├─ LoginEventHandler.md ✅ 登录事件（RBAC）
          └─ UserInfoHandler.md ✅ 用户信息查询事件（RBAC）
Hangfire/           后台任务文档
  ├─ Voiceprint/    声纹相关 Jobs
  │   └─ VoiceprintCaptureJob.md          ✅
  ├─ System/        系统相关 Jobs
  │   ├─ PatrolJobManager.md             ✅
  │   ├─ VoiceprintProcessedCleanupJob.md ✅
  │   └─ PatrolSystemCleanupJob.md       ✅
  │   └─ VisualGatewayHealthCheckJob.md  ✅
  ├─ Data/          数据相关 Jobs
  │   ├─ PointDataCleanupJob.md          ✅
  │   ├─ PointValueCacheCleanupJob.md    ✅
  │   ├─ InfraredTemperatureCollectionJob.md ✅
  │   └─ EnvironmentDetectionCollectionJob.md ✅
  ├─ Gateway/       网关相关 Jobs
  │   └─ GatewaySyncJob.md               ✅
  └─ Camera/        摄像机相关 Jobs
      ├─ CameraResourceCleanupJob.md     ✅
      └─ ISAPIResourceCleanupJob.md     ✅
  └─ Rbac/          RBAC相关 Jobs
      └─ BackupDataBaseJob.md            ✅ 数据库备份任务
Concepts/           架构概念与设计模式
  ├─ ABP仓储模式.md                       ✅
  └─ DDD分层架构.md                       ✅
Architecture/       架构文档
  ├─ 整体架构设计.md                      ✅
  ├─ 模块依赖关系.md                      ✅
  └─ 数据库设计.md                        ✅
Diagrams/           架构图和流程图
  ├─ 01-用户登录认证流程.md              ✅
  ├─ 02-告警管理流程.md                  ✅
  ├─ 03-仪表板与巡视流程.md              ✅
  ├─ 04-设备台账查询流程.md              ✅
  ├─ 05-声纹分析流程.md                  ✅
  └─ 图表索引.md                          ✅
Issues/             Bug 案例与调试
  ├─ ARM32堆栈溢出问题.md                ✅
  ├─ 连续告警误报问题.md                  ✅
  ├─ TDengine连接超时问题.md              ✅
  └─ 调试指南.md                          ✅
Configuration/      配置文档
  ├─ AppSettingsIndex.md                 ✅
  └─ 配置快速参考.md                      ⬜ 规划中
Resources/          外部资源索引
  └─ ExternalDocsIndex.md                ✅
    链接到外部实现方案：
    - [[../../../voiceprint/00-系统架构与功能实现.md|系统架构实现]]
    - [[../../../voiceprint/02-python-voiceprint-service方案.md|Python 声纹服务]]
    - [[../../../voiceprint/03-raspberry-pi-agent方案.md|树莓派边缘采集]]
    - [[../../../data-flow-diagrams/|前端 API 时序图（6 个 HTML）]]
    - [[../../../frontend-api-interfaces.md|前端 23 个接口]]
Templates/          笔记模板
  ├─ 组件笔记模板.md                     ✅
  ├─ 流程文档模板.md                     ✅
  ├─ 概念文档模板.md                     ✅
  ├─ API文档模板.md                      ✅
  └─ Hangfire任务模板.md                 ✅
DocumentationPlan.md  文档补充计划         ✅
快速导航.md          快速导航             ✅
README.md           项目说明             ✅
INDEX.md            本文件（知识图谱索引）✅
```

---

## API 文档索引

### 账户与认证
[[API/Account/AccountAPI]] — 用户登录、登出、密码管理 ✅

### 告警管理
[[API/Alarm/AlarmAPI]] — 告警列表、月度统计、告警处理 ✅

### 仪表板
[[API/Dashboard/DashboardAPI]] — 巡视统计、系统状态、首页告警 ✅

### 设备台账
[[API/Assets/AssetsAPI]] — 设备台账树、设备信息、运行趋势 ✅

### 声纹分析
[[API/Voiceprint/VoiceprintAPI]] — 标准音频库、测试音频、报告生成 ✅
[[API/VoiceprintAudio/VoiceprintAudioInternalAPI]] — 音频记录、处理、下载 ✅
[[API/VoiceprintAudio/VoiceprintAudioUploadAPI]] — 音频上传接口 ✅

### 传感器数据
[[API/PointData/PointDataAPI]] — 最新数据、历史数据、批量查询 ✅

### 报告管理
[[API/Report/ReportAPI]] — 报告导出、模板管理 ✅

### 智能变电站
[[API/IntelliSub/IntelliSubAPI]] — 变电站综合接口 ✅

---

## 后台任务索引 (Hangfire)

### 系统任务
[[Hangfire/System/PatrolJobManager]] — 巡检任务调度 ✅
[[Hangfire/System/VoiceprintProcessedCleanupJob]] — 音频清理 ✅
[[Hangfire/System/PatrolSystemCleanupJob]] — 巡检清理 ✅
[[Hangfire/System/VisualGatewayHealthCheckJob]] — 网关健康检查 ✅

### 数据任务
[[Hangfire/Data/PointDataCleanupJob]] — 点位数据清理 ✅
[[Hangfire/Data/PointValueCacheCleanupJob]] — 缓存清理 ✅
[[Hangfire/Data/InfraredTemperatureCollectionJob]] — 红外温度采集 ✅
[[Hangfire/Data/EnvironmentDetectionCollectionJob]] — 环境监测采集 ✅

### 网关任务
[[Hangfire/Gateway/GatewaySyncJob]] — 网关同步 ✅

### 摄像机任务
[[Hangfire/Camera/CameraResourceCleanupJob]] — 摄像机资源清理 ✅
[[Hangfire/Camera/ISAPIResourceCleanupJob]] — ISAPI 资源清理 ✅

### RBAC任务
[[Hangfire/Rbac/BackupDataBaseJob]] — 数据库备份任务 ✅

### 声纹任务
[[Hangfire/Voiceprint/VoiceprintCaptureJob]] — 声纹采集 ✅

---

## 数据流程管道

### 音频处理
[[Pipelines/Audio/音频处理管道]] — 音频处理完整流程 ✅
[[Pipelines/Audio/音频采集流程]] — 音频采集详细步骤 ✅

### 数据上报
[[Pipelines/DataReport/IEC61850上报管道]] — IEC61850 数据上报流程 ✅

### 事件驱动
[[Pipelines/Events/EventDrivenPipeline]] — 事件驱动架构管道 ✅

---

## 模块详细分类

### ✅ RBAC 模块（认证授权）

#### 概览
[[Modules/rbac/RbacOverview]] — RBAC 模块概述 ✅

#### 核心
[[Modules/rbac/认证系统]] · [[Modules/rbac/权限定义服务]] · [[Modules/rbac/用户管理API]] ✅

---

### ✅ ast-intellisub 模块（变电站监控）

#### 概览
[[Modules/ast-intellisub/变电站监视范围概览]] — 主要模块、数据驱动方式、监视对象与物理量 ✅

#### 设备管理
[[Modules/ast-intellisub/设备管理服务]] · [[Classes/Device]] ✅

#### 巡检系统
[[Modules/ast-intellisub/Patrol/PatrolTaskService]] · [[Modules/ast-intellisub/Patrol/PatrolExecutionService]] · [[Modules/ast-intellisub/Patrol/PatrolRecordService]] · [[Modules/ast-intellisub/Patrol/PatrolSystemRecoveryService]] ⬜

#### 监测点位
[[Modules/ast-intellisub/Monitoring/MonitoredPointService]] · [[Modules/ast-intellisub/Monitoring/MonitoredObjectService]] · [[Modules/ast-intellisub/Monitoring/SensorService]] ✅

#### 网关/MQTT
[[Modules/ast-intellisub/Gateway/GatewayService]] · [[Modules/ast-intellisub/Gateway/MqttService]] · [[Modules/ast-intellisub/Gateway/StreamingGatewayService]] ✅

#### 数据绑定
[[Modules/ast-intellisub/DataBinding/DataBindingItemService]] · [[Modules/ast-intellisub/DataBinding/DataStrategyService]] ✅

#### 告警系统
[[Modules/ast-intellisub/设备告警流程]] · [[Modules/ast-intellisub/Alarm/AlarmNotificationHub]] · [[Modules/ast-intellisub/Alarm/AlarmNotificationService]] · [[Modules/ast-intellisub/Alarm/AlarmCategoryService]] · [[Classes/Alarm]] ✅

#### 摄像机集成
[[Modules/ast-intellisub/Camera/CameraService]] · [[Modules/ast-intellisub/Camera/MediaService]] · [[Modules/ast-intellisub/Camera/StreamingApiService]] · [[Modules/ast-intellisub/Camera/PresetService]] · [[Modules/ast-intellisub/Camera/CameraResourceManager]] ✅

#### AI 识别
[[Modules/ast-intellisub/AI/AIRecognitionService]] · [[Modules/ast-intellisub/AI/AlgorithmService]] · [[Modules/ast-intellisub/AI/RecognitionApiService]] ✅

#### 系统分析
[[Modules/ast-intellisub/Analysis/SystemAnalysisService]] · [[Modules/ast-intellisub/Analysis/SystemStatisticsService]] ✅

#### 变电站管理
[[Modules/ast-intellisub/Substation/SubstationService]] · [[Modules/ast-intellisub/Substation/SubstationUserService]] · [[Modules/ast-intellisub/Substation/SubstationTypeService]] ✅

---

### ✅ ast-intellisubdata 模块（时序数据）

#### 概览
[[Modules/ast-intellisubdata/传感器数据管理服务]] · [[Classes/PointData]] ✅

#### 数据采集
[[Modules/ast-intellisubdata/数据上报流程]] ✅

#### 数据存储
[[Modules/ast-intellisubdata/TDengine集成]] ✅

#### 数据清理
[[Modules/ast-intellisubdata/数据清理任务]] ✅

---

### ✅ ast-voiceprint 模块（声纹分析）

#### 音频处理
[[Modules/ast-voiceprint/声纹采集服务]] · [[Modules/ast-voiceprint/声纹分析流程]] · [[Classes/Voiceprint]] · [[Modules/ast-voiceprint/VoiceprintCaptureRuntimeStateService]] ✅

#### 流程文档
[[Pipelines/Audio/音频处理管道]] · [[Pipelines/Audio/音频采集流程]] ✅

---

### ✅ isapi 模块（工业协议）

#### IEC61850
[[Modules/isapi/IEC61850数据上报服务]] · [[Modules/isapi/数据上报流程]] · [[Pipelines/DataReport/IEC61850上报管道]] · [[Modules/isapi/Iec61850ApiService]] ✅

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
[[Modules/audit-logging/AuditLoggingOverview]] ✅

### 多租户
[[Modules/tenant-management/TenantManagementOverview]] ✅

### 系统设置
[[Modules/setting-management/SettingManagementOverview]] ✅

### 系统服务
[[Modules/System/EnumService]] · [[Modules/System/FileService]] · [[Modules/System/DataIntegrityCheckService]] · [[Modules/System/ApplicationStartupService]] ✅

---

## 架构概念

### 架构文档
[[Architecture/整体架构设计]] · [[Architecture/模块依赖关系]] · [[Architecture/数据库设计]] ✅

### 设计模式
[[Concepts/ABP仓储模式]] · [[Concepts/DDD分层架构]] ✅

---

## 配置说明

### 部署模式
- **LowResource** (默认) — SQLite + Memory 存储，ARM32/边缘设备优化
- **HighPerformance** — 完整数据库 + Redis/TDengine，数据中心部署

### 配置文件
[[Configuration/AppSettingsIndex]] ✅

---

## 学习资源

- [[DocumentationPlan]] — 文档补充计划 ✅
- [[待补文档清单]] — 待补文档清单与建议目录结构 ✅
- [[快速导航]] — 快速导航索引 ✅
- [[Diagrams/图表索引]] — 流程图索引 ✅

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
