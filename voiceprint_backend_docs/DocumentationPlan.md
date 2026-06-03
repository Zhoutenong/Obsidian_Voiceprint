# 文档补充计划

本文档列出了需要补充的所有文档内容，按优先级分类。

## P0 - 核心功能文档（最高优先级）

### ast-intellisub 核心服务

#### 巡检系统
- [ ] `PatrolTaskService` - 巡检任务管理服务
- [ ] `PatrolExecutionService` - 巡检执行服务
- [ ] `PatrolRecordService` - 巡检记录服务
- [ ] `PatrolJobManager` - 巡检任务调度器（Hangfire Job）
- [ ] `PatrolSystemCleanupJob` - 巡检系统清理任务
- [ ] `PatrolSystemRecoveryService` - 巡检系统恢复服务

#### 监测点位/对象
- [ ] `MonitoredPointService` - 监测点位服务
- [ ] `MonitoredObjectService` - 监测对象服务
- [ ] `RealtimeMonitoringPointService` - 实时监测点位服务
- [ ] `SensorService` - 传感器服务
- [ ] `MonitoredPointAlarmCategoryRelService` - 点位告警分类关联服务

#### 网关/MQTT
- [ ] `GatewayService` - 网关服务
- [ ] `MqttService` - MQTT 服务
- [ ] `MqttMessageBackgroundService` - MQTT 消息后台服务
- [ ] `GatewaySyncJob` - 网关同步任务
- [ ] `StreamingGatewayService` - 流媒体网关服务

#### 数据绑定/策略
- [ ] `DataBindingItemService` - 数据绑定项服务
- [ ] `DataStrategyService` - 数据策略服务
- [ ] `StrategyStateService` - 策略状态服务
- [ ] `BindingItemStrategyRelService` - 绑定项策略关联服务

#### 告警系统
- [ ] `AlarmNotificationHub` - SignalR 告警推送 Hub
- [ ] `AlarmNotificationService` - 告警通知服务
- [ ] `AlarmCategoryService` - 告警分类服务
- [ ] `CollectorService` - 告警采集服务

### Hangfire 后台任务（已列出 13 个）

- [x] `VoiceprintCaptureJob` - 声纹采集任务（已有文档）
- [ ] `VoiceprintProcessedCleanupJob` - 声纹处理后清理任务
- [ ] `PatrolJobManager` - 巡检任务调度器
- [ ] `PatrolSystemCleanupJob` - 巡检系统清理
- [ ] `GatewaySyncJob` - 网关同步任务
- [ ] `VisualGatewayHealthCheckJob` - 可视化网关健康检查
- [ ] `PointDataCleanupJob` - 点位数据清理
- [ ] `PointValueCacheCleanupJob` - 点位值缓存清理
- [ ] `InfraredTemperatureCollectionJob` - 红外温度采集
- [ ] `EnvironmentDetectionCollectionJob` - 环境监测采集
- [ ] `CameraResourceCleanupJob` - 摄像机资源清理
- [ ] `ISAPIResourceCleanupJob` - ISAPI 资源清理
- [ ] `BackupDataBaseJob` - 数据库备份任务

## P1 - API 文档

### 声纹模块 API

#### VoiceprintPortalAppService
- [ ] `GET /api/app/voiceprint/standard-audios` - 标准音频库列表
- [ ] `POST /api/app/voiceprint/standard-audios` - 添加标准音频
- [ ] `PUT /api/app/voiceprint/standard-audios/{id}` - 更新标准音频
- [ ] `DELETE /api/app/voiceprint/standard-audios/{id}` - 删除标准音频
- [ ] `GET /api/app/voiceprint/test-audios` - 测试音频列表
- [ ] `POST /api/app/voiceprint/test-audios/import` - 导入测试音频
- [ ] `POST /api/app/voiceprint/test-audios/{id}/recognize` - 触发识别
- [ ] `DELETE /api/app/voiceprint/test-audios/{id}` - 删除测试音频
- [ ] `POST /api/app/voiceprint/reports/generate-from-audio` - 从音频生成报告
- [ ] `GET /api/app/voiceprint/reports` - 报告列表
- [ ] `DELETE /api/app/voiceprint/reports/{id}` - 删除报告
- [ ] `POST /api/app/voiceprint/algorithm/switch` - 切换算法模式
- [ ] `GET /api/app/voiceprint/algorithm/status` - 获取算法状态

#### VoiceprintAudioAppService
- [ ] `GET /api/app/voiceprint/audios` - 音频记录列表
- [ ] `GET /api/app/voiceprint/audios/{id}` - 获取音频详情
- [ ] `DELETE /api/app/voiceprint/audios/{id}` - 删除音频记录
- [ ] `POST /api/app/voiceprint/audios/{id}/process` - 处理音频
- [ ] `POST /api/app/voiceprint/audios/download` - 批量下载音频
- [ ] `POST /api/app/voiceprint/audios/{id}/recognition-result` - 获取识别结果

### 告警模块 API

- [ ] `GET /api/app/voiceprint/alarms` - 告警列表
- [ ] `GET /api/app/voiceprint/alarms/monthly-stat` - 月度统计
- [ ] `GET /api/app/voiceprint/alarms/device-options` - 设备选项
- [ ] `PUT /api/app/voiceprint/alarms/{alarmId}` - 处理告警
- [ ] `GET /api/app/voiceprint/alarm-categories` - 告警分类列表

### 设备台账 API

- [ ] `GET /api/app/voiceprint/assets/tree` - 设备台账树
- [ ] `GET /api/app/voiceprint/assets/{monitoredObjectId}` - 设备基础信息
- [ ] `GET /api/app/voiceprint/assets/{monitoredObjectId}/trend` - 设备运行趋势
- [ ] `GET /api/app/voiceprint/assets/{monitoredObjectId}/anomaly-mix` - 设备异常分布
- [ ] `GET /api/app/voiceprint/assets/{monitoredObjectId}/logs` - 设备巡检日志
- [ ] `POST /api/app/voiceprint/assets/{monitoredObjectId}/processed-audios/download` - 批量下载音频

### 仪表板 API

- [ ] `GET /api/app/voiceprint/dashboard/overview` - 巡视统计
- [ ] `GET /api/app/voiceprint/dashboard/records` - 巡视记录列表
- [ ] `GET /api/app/voiceprint/dashboard/records/{groupId}` - 巡视记录详情
- [ ] `GET /api/app/voiceprint/dashboard/system-status` - 系统状态（电子沙盘）
- [ ] `GET /api/app/voiceprint/dashboard/alarms` - 首页告警
- [ ] `POST /api/app/voiceprint/dashboard/generate-alarm-report` - 生成告警报告

### 报告模块 API

- [ ] `POST /api/app/voiceprint/reports/export` - 导出智能巡视报表
- [ ] `GET /api/app/voiceprint/reports/templates` - 报告模板列表
- [ ] `POST /api/app/voiceprint/reports/templates` - 创建报告模板
- [ ] `PUT /api/app/voiceprint/reports/templates/{id}` - 更新报告模板
- [ ] `DELETE /api/app/voiceprint/reports/templates/{id}` - 删除报告模板

### 账户 API

- [ ] `POST /api/app/account/login` - 用户登录
- [ ] `GET /api/app/account` - 获取当前账户信息
- [ ] `POST /api/app/account/logout` - 用户登出
- [ ] `PUT /api/app/account/password` - 修改密码

### 传感器数据 API

- [ ] `GET /api/app/point-data/latest` - 最新点位数据
- [ ] `GET /api/app/point-data/history` - 历史点位数据
- [ ] `GET /api/app/point-data/batch` - 批量获取点位数据
- [ ] `POST /api/app/point-data/report` - 上报点位数据

## P2 - 其他重要模块

### 流媒体/摄像机
- [ ] `CameraService` - 摄像机服务
- [ ] `MediaService` - 媒体服务
- [ ] `StreamingApiService` - 流媒体 API 服务
- [ ] `NvrService` - NVR 服务
- [ ] `PresetService` - 预置位服务
- [ ] `PtzPresetAppService` - PTZ 预置位应用服务
- [ ] `CameraResourceCleanupJob` - 摄像机资源清理任务

### AI 识别
- [ ] `AIRecognitionService` - AI 识别服务
- [ ] `RecognitionApiService` - 识别 API 服务

### 系统分析/报表
- [ ] `SystemAnalysisService` - 系统分析服务
- [ ] `SystemStatisticsService` - 系统统计服务
- [ ] `ReportService` - 报告服务

### 变电站管理
- [ ] `SubstationService` - 变电站服务
- [ ] `SubstationUserService` - 变电站用户服务

### ISAPI 模块
- [ ] `ISAPIService` - ISAPI 服务
- [ ] `ISAPIResourceCleanupJob` - ISAPI 资源清理任务

### 数据采集
- [ ] `InfraredTemperatureCollectionJob` - 红外温度采集
- [ ] `EnvironmentDetectionCollectionJob` - 环境监测采集

### 数据清理
- [ ] `PointDataCleanupJob` - 点位数据清理
- [ ] `PointValueCacheCleanupJob` - 点位值缓存清理

## P3 - 基础设施模块

### audit-logging（审计日志）
- [ ] 审计日志概述
- [ ] `AuditLogService` - 审计日志服务
- [ ] `AuditLogActionService` - 审计日志操作服务
- [ ] 审计日志查询接口

### tenant-management（多租户）
- [ ] 多租户概述
- [ ] `TenantService` - 租户服务
- [ ] `TenantConnectionService` - 租户连接服务
- [ ] 租户管理接口

### setting-management（系统设置）
- [ ] 系统设置概述
- [ ] `SettingService` - 设置服务
- [ ] `SettingGroupService` - 设置组服务
- [ ] 设置管理接口

### RBAC 完整功能
- [ ] `MenuService` - 菜单服务
- [ ] `RoleService` - 角色服务
- [ ] `DepartmentService` - 部门服务
- [ ] `DictionaryService` - 字典服务
- [ ] `OperationLogService` - 操作日志服务

### IEC104 协议
- [ ] IEC104 协议概述
- [ ] IEC104 连接管理
- [ ] IEC104 数据上报
- [ ] IEC104 遥控/遥调

### 其他
- [ ] `EnumService` - 枚举服务
- [ ] `FileService` - 文件服务
- [ ] `ApplicationStartupService` - 应用启动服务
- [ ] `DataIntegrityCheckService` - 数据完整性检查服务
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

- **总服务数量**: 约 80+
- **总 Job 数量**: 13
- **总 API 接口数量**: 约 50+
- **预计文档数量**: 约 150+

## 实施计划

1. **第一阶段**: P0 核心功能文档（ast-intellisub 核心 + Hangfire Jobs）
2. **第二阶段**: P1 API 文档（所有对外接口）
3. **第三阶段**: P2 其他重要模块（流媒体、AI、分析报表）
4. **第四阶段**: P3 基础设施模块

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
