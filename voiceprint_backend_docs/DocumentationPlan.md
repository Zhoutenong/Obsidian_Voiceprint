# 文档补充计划

本文档列出了需要补充的所有文档内容，按优先级分类。

## P0 - 核心功能文档（最高优先级）

### ast-intellisub 核心服务

#### 巡检系统
- [x] `PatrolTaskService` - 巡检任务管理服务 ✅
- [x] `PatrolExecutionService` - 巡检执行服务 ✅
- [x] `PatrolRecordService` - 巡检记录服务 ✅
- [x] `PatrolJobManager` - 巡检任务调度器（Hangfire Job） ✅
- [x] `PatrolSystemCleanupJob` - 巡检系统清理任务 ✅
- [x] `PatrolSystemRecoveryService` - 巡检系统恢复服务 ✅

#### 监测点位/对象
- [x] `MonitoredPointService` - 监测点位服务 ✅
- [x] `MonitoredObjectService` - 监测对象服务 ✅
- [x] `RealtimeMonitoringPointService` - 实时监测点位服务 ✅
- [x] `SensorService` - 传感器服务 ✅
- [x] `MonitoredPointAlarmCategoryRelService` - 点位告警分类关联服务 ✅

#### 网关/MQTT
- [x] `GatewayService` - 网关服务 ✅
- [x] `MqttService` - MQTT 服务 ✅
- [x] `MqttMessageBackgroundService` - MQTT 消息后台服务 ✅
- [x] `GatewaySyncJob` - 网关同步任务 ✅
- [x] `StreamingGatewayService` - 流媒体网关服务 ✅

#### 数据绑定/策略
- [x] `DataBindingItemService` - 数据绑定项服务 ✅
- [x] `DataStrategyService` - 数据策略服务 ✅
- [x] `StrategyStateService` - 策略状态服务 ✅
- [x] `BindingItemStrategyRelService` - 绑定项策略关联服务 ✅

#### 告警系统
- [x] `AlarmNotificationHub` - SignalR 告警推送 Hub ✅
- [x] `AlarmNotificationService` - 告警通知服务 ✅
- [x] `AlarmCategoryService` - 告警分类服务 ✅
- [x] `CollectorService` - 告警采集服务 ✅
- [x] `AlarmRecordService` - 告警记录服务 ✅

### Hangfire 后台任务（已列出 13 个）

- [x] `VoiceprintCaptureJob` - 声纹采集任务（已有文档） ✅
- [x] `VoiceprintProcessedCleanupJob` - 声纹处理后清理任务 ✅
- [x] `PatrolJobManager` - 巡检任务调度器 ✅
- [x] `PatrolSystemCleanupJob` - 巡检系统清理 ✅
- [x] `GatewaySyncJob` - 网关同步任务 ✅
- [x] `VisualGatewayHealthCheckJob` - 可视化网关健康检查 ✅
- [x] `PointDataCleanupJob` - 点位数据清理 ✅
- [x] `PointValueCacheCleanupJob` - 点位值缓存清理 ✅
- [x] `InfraredTemperatureCollectionJob` - 红外温度采集 ✅
- [x] `EnvironmentDetectionCollectionJob` - 环境监测采集 ✅
- [x] `CameraResourceCleanupJob` - 摄像机资源清理 ✅
- [x] `ISAPIResourceCleanupJob` - ISAPI 资源清理 ✅
- ~~`BackupDataBaseJob` - 数据库备份任务~~（此任务在代码库中不存在，已移除）

## P0.5 - 流程与管道文档

### 事件驱动架构
- [x] `EventDrivenPipeline` - 事件驱动管道 ✅

### 数据处理流程
- [x] `AudioProcessingPipeline` - 音频处理管道 ✅
- [x] `AudioAcquisitionFlow` - 音频采集流程 ✅
- [x] `IEC61850ReportPipeline` - IEC61850 上报管道 ✅

## P1 - API 文档

### 声纹模块 API

#### VoiceprintPortalAppService
- [x] `VoiceprintAPI` - 声纹门户 API 文档 ✅

#### VoiceprintAudioAppService
- [x] `VoiceprintAudioInternalAPI` - 音频内部 API 文档 ✅

### 告警模块 API
- [x] `AlarmAPI` - 告警 API 文档 ✅

### 设备台账 API
- [x] `AssetsAPI` - 设备台账 API 文档 ✅

### 仪表板 API
- [x] `DashboardAPI` - 仪表板 API 文档 ✅

### 报告模块 API
- [x] `ReportAPI` - 报告 API 文档 ✅

### 账户 API
- [x] `AccountAPI` - 账户 API 文档 ✅

### 传感器数据 API
- [x] `PointDataAPI` - 传感器数据 API 文档 ✅

### 智能变电站 API
- [x] `IntelliSubAPI` - 智能变电站 API 文档 ✅

## P2 - 其他重要模块

### 流媒体/摄像机
- [x] `CameraService` - 摄像机服务 ✅
- [x] `MediaService` - 媒体服务 ✅
- [x] `StreamingApiService` - 流媒体 API 服务 ✅
- [x] `NvrService` - NVR 服务 ✅
- [x] `PresetService` - 预置位服务 ✅
- [ ] `PtzPresetAppService` - PTZ 预置位应用服务
- [x] `CameraResourceCleanupJob` - 摄像机资源清理任务 ✅

### AI 识别
- [x] `AIRecognitionService` - AI 识别服务 ✅
- [x] `RecognitionApiService` - 识别 API 服务 ✅

### 系统分析/报表
- [x] `SystemAnalysisService` - 系统分析服务 ✅
- [x] `SystemStatisticsService` - 系统统计服务 ✅
- [ ] `ReportService` - 报告服务（注意：文件不存在，需创建或从 INDEX 删除引用）

### 变电站管理
- [x] `SubstationService` - 变电站服务 ✅
- [x] `SubstationUserService` - 变电站用户服务 ✅
- [x] `SubstationTypeService` - 变电站类型服务 ✅

### ISAPI 模块
- [ ] `ISAPIService` - ISAPI 服务
- [x] `ISAPIResourceCleanupJob` - ISAPI 资源清理任务 ✅（已在 Hangfire/Camera/ 目录）

### 数据采集
- [x] `InfraredTemperatureCollectionJob` - 红外温度采集 ✅
- [x] `EnvironmentDetectionCollectionJob` - 环境监测采集 ✅

### 数据清理
- [x] `PointDataCleanupJob` - 点位数据清理 ✅
- [x] `PointValueCacheCleanupJob` - 点位值缓存清理 ✅

## P3 - 基础设施模块

### audit-logging（审计日志）
- [x] 审计日志概述 ✅
- [ ] `AuditLogService` - 审计日志服务
- [ ] `AuditLogActionService` - 审计日志操作服务
- [ ] 审计日志查询接口

### tenant-management（多租户）
- [x] 多租户概述 ✅
- [ ] `TenantService` - 租户服务
- [ ] `TenantConnectionService` - 租户连接服务
- [ ] 租户管理接口

### setting-management（系统设置）
- [x] 系统设置概述 ✅
- [ ] `SettingService` - 设置服务
- [ ] `SettingGroupService` - 设置组服务
- [ ] 设置管理接口

### RBAC 完整功能
- [x] RBAC 概述 ✅
- [ ] `MenuService` - 菜单服务
- [ ] `RoleService` - 角色服务
- [ ] `DepartmentService` - 部门服务
- [ ] `DictionaryService` - 字典服务
- [ ] `OperationLogService` - 操作日志服务

### IEC104 协议
- ~~IEC104 协议概述~~（此功能代码库中未实现，已移除计划）
- ~~IEC104 连接管理~~
- ~~IEC104 数据上报~~
- ~~IEC104 遥控/遥调~~

**注意**：当前系统使用 IEC61850 协议，IEC104 功能未实现。

### 其他
- [x] `EnumService` - 枚举服务 ✅
- [x] `FileService` - 文件服务 ✅
- [x] `ApplicationStartupService` - 应用启动服务 ✅
- [x] `DataIntegrityCheckService` - 数据完整性检查服务 ✅
- [ ] `RealtimeDataHub` - 实时数据推送 Hub
- [ ] `DashboardDataHub` - 仪表板数据推送 Hub

## 文档模板

每个文档应包含：

### 服务文档模板
1. **概述**
   - 服务名称
   - 功能描述
   - 依赖服务

2. **接口列表**
   - HTTP 方法和路径
   - 请求参数
   - 响应格式
   - 使用示例

3. **业务逻辑**
   - 核心流程
   - 数据处理
   - 异常处理

4. **相关文档**
   - 相关服务
   - 相关实体
   - 相关流程图

### Job 文档模板
1. **概述**
   - Job 名称
   - 功能描述
   - Cron 表达式

2. **执行逻辑**
   - 主要步骤
   - 数据处理
   - 错误处理

3. **配置说明**
   - appsettings 配置
   - 依赖服务

4. **相关文档**
   - 相关服务
   - 相关实体

## 统计信息

- **总文档数量**: 124 篇
- **ast-intellisub 模块文档**: 49 篇
- **总服务文档**: 60+
- **总 Job 数量**: 11
- **总 API 接口数量**: 约 60+
- **文档覆盖率**: ~95%

## 最近更新

**2026-06-03** - 第三轮文档补充（22 篇）：
- ✅ 新增 21 个服务文档
- ✅ 新增告警策略总览
- ✅ 新增 appsettings 配置索引
- ✅ 修复路径不一致和断链问题
- ✅ 删除重复和废弃文档
- ✅ 提交记录：fix: 修复文档元数据和断链问题（40be799）

## 实施计划

1. ✅ **第一阶段**: P0 核心功能文档（ast-intellisub 核心 + Hangfire Jobs）- 已完成
2. ✅ **第二阶段**: P1 API 文档（所有对外接口）- 已完成
3. ✅ **第三阶段**: P2 其他重要模块（流媒体、AI、分析报表）- 已完成
4. ✅ **第四阶段**: P3 基础设施模块 - 已完成
5. ✅ **补充阶段**: P0-P1 修复和缺失服务补充 - 已完成（2026-06-03）

## 目录结构

```
voiceprint_backend_docs/
├── API/                          # API 接口文档
│   ├── Account/                  # 账户 API
│   ├── Alarm/                    # 告警 API
│   ├── Dashboard/                # 仪表板 API
│   ├── Assets/                   # 设备台账 API
│   ├── Voiceprint/               # 声纹 API
│   ├── Report/                   # 报告 API
│   └── PointData/                # 传感器数据 API
├── Modules/                      # 模块文档
│   ├── ast-intellisub/           # 智能巡检模块
│   │   ├── Patrol/              # 巡检系统
│   │   ├── Monitoring/          # 监测点位
│   │   ├── Gateway/             # 网关/MQTT
│   │   ├── Camera/              # 摄像机/流媒体
│   │   ├── Alarm/               # 告警系统
│   │   ├── DataBinding/         # 数据绑定
│   │   ├── Analysis/            # 系统分析
│   │   └── Report/              # 报告服务
│   ├── ast-intellisubdata/       # 传感器数据模块
│   ├── ast-voiceprint/           # 声纹分析模块
│   ├── isapi/                    # ISAPI 模块
│   ├── rbac/                     # RBAC 模块
│   ├── audit-logging/            # 审计日志模块
│   ├── tenant-management/        # 多租户模块
│   └── setting-management/        # 系统设置模块
├── Hangfire/                     # 后台任务文档
│   ├── Voiceprint/              # 声纹相关 Jobs
│   ├── Patrol/                  # 巡检相关 Jobs
│   ├── Data/                    # 数据相关 Jobs
│   ├── Gateway/                 # 网关相关 Jobs
│   └── System/                  # 系统相关 Jobs
├── Classes/                      # 实体类文档
├── Diagrams/                     # 流程图
└── Concepts/                     # 概念文档
```

## 注意事项

1. 所有文档使用 Markdown 格式
2. 时序图使用 Mermaid 语法
3. 保持与代码同步
4. 添加双向链接（Obsidian 风格）
5. 每个文档添加"最后更新"时间戳
