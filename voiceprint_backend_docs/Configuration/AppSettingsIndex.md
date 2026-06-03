# AppSettings 配置索引

> **概述**：本文档索引了 IntelliSubstation Voiceprint Backend 系统的所有配置节，帮助开发者快速定位和理解各项配置的作用。

---

## 配置文件结构

系统支持多环境配置，通过 **appsettings.json** 和环境特定文件实现配置分层：

| 配置文件 | 用途 | 加载时机 |
|---------|------|---------|
| `appsettings.json` | 基础配置（默认） | 始终加载 |
| `appsettings.{Environment}.json` | 环境特定配置 | 根据 `ASPNETCORE_ENVIRONMENT` 加载 |
| `appsettings.LowResource.json` | 低资源部署模式（ARM32/边缘设备） | 显式指定 |
| `appsettings.HighPerformance.json` | 高性能部署模式（数据中心） | 显式指定 |

**配置优先级**：环境特定配置 > 基础配置

**设置环境变量**：
```bash
# Windows
set ASPNETCORE_ENVIRONMENT=Production

# Linux/Mac
export ASPNETCORE_ENVIRONMENT=Production
```

---

## 核心配置节

### 1. DbConnOptions（数据库连接）

数据库连接和部署模式配置。

```json
{
  "DbConnOptions": {
    "DeploymentMode": "LowResource",
    "Url": "Data Source=db/ast_intellisub.db",
    "DbType": "Sqlite",
    "TDengineUrl": "",
    "EnabledReadWrite": false,
    "EnabledCodeFirst": true,
    "EnabledSqlLog": false,
    "EnabledDbSeed": true,
    "EnableUnderLine": false,
    "EnabledSaasMultiTenancy": false
  }
}
```

**配置项说明**：

| 配置项 | 说明 | 可选值 |
|-------|------|--------|
| `DeploymentMode` | 部署模式 | `LowResource`（低资源）<br>`HighPerformance`（高性能） |
| `Url` | 主数据库连接字符串 | 依 `DbType` 而定 |
| `DbType` | 数据库类型 | `Sqlite`, `Mysql`, `Sqlserver`, `Oracle`, `PostgreSQL` |
| `TDengineUrl` | TDengine 时序数据库连接字符串 | 留空表示不使用 TDengine |
| `EnabledReadWrite` | 启用读写分离 | `true`, `false` |
| `EnabledCodeFirst` | 启用 CodeFirst（自动建表） | `true`, `false` |
| `EnabledSqlLog` | 启用 SQL 日志 | `true`, `false` |
| `EnabledDbSeed` | 启用种子数据初始化 | `true`, `false` |
| `EnableUnderLine` | 启用驼峰转下划线（数据库字段） | `true`, `false` |
| `EnabledSaasMultiTenancy` | 启用多租户 | `true`, `false` |

**部署模式差异**：

| 特性 | LowResource | HighPerformance |
|-----|-------------|----------------|
| 数据库 | SQLite（单文件） | PostgreSQL / MySQL / SQL Server |
| 时序数据 | SQLite 主库 | TDengine 独立存储 |
| Hangfire 存储 | 内存 | SQLite / Redis |
| 适用场景 | ARM32 边缘设备 | x64 数据中心 |

**连接字符串示例**：

```json
// SQLite
"Url": "Data Source=db/ast_intellisub.db"

// MySQL
"Url": "server=localhost;port=3306;database=ast_db;user id=root;password=123456"

// PostgreSQL
"Url": "Host=localhost;Port=5432;Database=ast_db;Username=ast;Password=ast123456"

// SQL Server
"Url": "Data Source=localhost;Initial Catalog=ast_db;User ID=sa;Password=YourPassword123"

// TDengine
"TDengineUrl": "Host=localhost;Port=6030;Database=ast;Username=root;Password=taosdata"
```

---

### 2. Hangfire（后台任务）

Hangfire 后台任务存储配置。

```json
{
  "Hangfire": {
    "StorageMode": "Memory",
    "SQLiteDbPath": "db/hangfire.db"
  }
}
```

**配置项说明**：

| 配置项 | 说明 | 可选值 |
|-------|------|--------|
| `StorageMode` | 存储模式 | `Memory`（内存，重启丢失）<br>`SQLite`（持久化）<br>`Redis`（高性能） |
| `SQLiteDbPath` | SQLite 数据库路径 | 相对或绝对路径，`StorageMode=SQLite` 时使用 |

**选择建议**：

- **LowResource 模式**：`Memory`（减少磁盘 I/O）
- **HighPerformance 模式**：`SQLite` 或 `Redis`（任务持久化）

**访问仪表板**：`http://localhost:19001/hangfire`

---

### 3. VoiceprintCaptureJob（声纹采集）

声纹音频采集下行任务配置。

```json
{
  "VoiceprintCaptureJob": {
    "Enabled": true,
    "CronExpression": "0 */10 * * * *",
    "DurationSeconds": 10,
    "ManualCaptureTimeoutMinutes": 10,
    "Agents": [
      {
        "GatewayId": "d2e89964-38e0-8649-3462-1b311d196a1c",
        "DeviceIds": ["device1", "device2"],
        "RetryLimit": 3
      }
    ]
  }
}
```

**配置项说明**：

| 配置项 | 说明 | 默认值 |
|-------|------|--------|
| `Enabled` | 是否启用任务 | `true` |
| `CronExpression` | Cron 表达式（秒级） | `"0 */10 * * * *"`（每 10 分钟） |
| `DurationSeconds` | 采集持续时长（秒） | `10` |
| `ManualCaptureTimeoutMinutes` | 手动采集超时（分钟） | `10` |
| `Agents` | 网关设备列表 | - |

**Cron 表达式格式**：`秒 分 时 日 月 周`

```bash
# 每 5 分钟
"0 */5 * * * *"

# 每小时（整点）
"0 0 * * * *"

# 每天凌晨 2 点
"0 0 2 * * *"
```

---

### 4. VoiceprintStorage（声纹存储）

声纹音频文件存储路径配置。

```json
{
  "VoiceprintStorage": {
    "Root": "wwwroot/audio",
    "PendingFolderName": "unprocessed",
    "ProcessedFolderName": "processed",
    "StandardLibraryFolderName": "standard_library"
  }
}
```

**配置项说明**：

| 配置项 | 说明 | 默认值 |
|-------|------|--------|
| `Root` | 音频存储根目录（相对或绝对路径） | `wwwroot/audio` |
| `PendingFolderName` | 待处理音频子目录 | `unprocessed` |
| `ProcessedFolderName` | 已处理音频子目录 | `processed` |
| `StandardLibraryFolderName` | 标准音频库子目录 | `standard_library` |

**目录结构**：

```
wwwroot/audio/
├── unprocessed/      # 待处理音频（采集后落盘）
├── processed/        # 已处理音频（识别后移动）
└── standard_library/ # 标准音频库
```

**HTTP 访问**：
- 待处理：`http://host/audio/unprocessed/{filename}`
- 已处理：`http://host/audio/processed/{filename}`

---

### 5. VoiceprintRecognition（声纹识别）

声纹识别服务配置。

```json
{
  "VoiceprintRecognition": {
    "TestAudioApiUrl": "http://127.0.0.1:9806/api/app/voiceprint/test-audio",
    "TestAudioDirectory": "tests/audio",
    "RequestTimeoutSeconds": 30,
    "AlgorithmSwitchApiUrl": "http://voiceprint_analysis:9806/api/app/voiceprint/algorithm/switch",
    "AlgorithmCurrentApiUrl": "http://voiceprint_analysis:9806/api/app/voiceprint/algorithm/current",
    "AlgorithmSwitchTimeoutSeconds": 30
  }
}
```

**配置项说明**：

| 配置项 | 说明 |
|-------|------|
| `TestAudioApiUrl` | 测试音频识别 API 地址 |
| `TestAudioDirectory` | 测试音频文件目录 |
| `RequestTimeoutSeconds` | 请求超时时间（秒） |
| `AlgorithmSwitchApiUrl` | 算法切换 API 地址 |
| `AlgorithmCurrentApiUrl` | 当前算法查询 API 地址 |
| `AlgorithmSwitchTimeoutSeconds` | 算法切换超时时间（秒） |

---

### 6. VoiceprintReport（声纹报表）

声纹报表生成服务配置。

```json
{
  "VoiceprintReport": {
    "ApiUrl": "http://localhost:9888/report",
    "TemplateFileName": "report_template.docx",
    "ResultFileNamePrefix": "voiceprint_report_",
    "ImageWidth": 800,
    "ChartApiUrl": "http://192.168.3.16:9889/waveform",
    "AudioBaseUrl": "http://192.168.3.16:9803"
  }
}
```

**配置项说明**：

| 配置项 | 说明 |
|-------|------|
| `ApiUrl` | 报表生成服务地址 |
| `TemplateFileName` | DOCX 模板文件名 |
| `ResultFileNamePrefix` | 结果文件名前缀 |
| `ImageWidth` | 报表中图片宽度（像素） |
| `ChartApiUrl` | 频域/波形图服务地址 |
| `AudioBaseUrl` | 音频文件访问基础 URL |

---

### 7. VoiceprintProcessedCleanupJob（音频清理）

已处理音频文件清理任务。

```json
{
  "VoiceprintProcessedCleanupJob": {
    "Enabled": true,
    "CronExpression": "0 40 17 * * *",
    "RetentionDays": 117
  }
}
```

**配置项说明**：

| 配置项 | 说明 | 默认值 |
|-------|------|--------|
| `Enabled` | 是否启用任务 | `true` |
| `CronExpression` | Cron 表达式 | `"0 40 17 * * *"`（每天 17:40） |
| `RetentionDays` | 保留天数（超过此天数的已处理音频将被删除） | `117` |

---

### 8. Iec61850（IEC61850 上报）

IEC61850 工业协议数据上报配置。

```json
{
  "Iec61850": {
    "ModelFileDirectory": "",
    "EnableDataReport": false,
    "EnableAlarmReport": false,
    "BatchSize": 100,
    "BatchIntervalMs": 1000,
    "ModelFiles": [
      {
        "ModelFile": "GISSF60_104_ast-temp.icd",
        "Processor": "GeneralDataReportProcessor",
        "Host": "192.168.130.50:8000",
        "Enabled": true
      }
    ]
  }
}
```

**配置项说明**：

| 配置项 | 说明 |
|-------|------|
| `ModelFileDirectory` | ICD 模型文件目录（留空使用 `./Iec61850`） |
| `EnableDataReport` | 启用数据上报 |
| `EnableAlarmReport` | 启用告警上报 |
| `BatchSize` | 批量上报数量 |
| `BatchIntervalMs` | 批量上报间隔（毫秒） |
| `ModelFiles` | 模型文件配置列表 |

**ModelFile 配置项**：

| 配置项 | 说明 |
|-------|------|
| `ModelFile` | ICD 文件名 |
| `Processor` | 数据处理器类名 |
| `Host` | 远程主机地址 |
| `Enabled` | 是否启用此模型 |

---

### 9. Redis（缓存配置）

Redis 缓存配置。

```json
{
  "Redis": {
    "IsEnabled": false,
    "Configuration": "localhost:6379,password=,defaultDatabase=13"
  }
}
```

**配置项说明**：

| 配置项 | 说明 |
|-------|------|
| `IsEnabled` | 是否启用 Redis |
| `Configuration` | Redis 连接字符串 |

**连接字符串格式**：
```
{host}:{port},password={password},defaultDatabase={db}
```

---

### 10. SignalR（实时通信）

SignalR 实时告警中心配置。

```json
{
  "AlarmHub": {
    "RequireAuthentication": false
  }
}
```

**配置项说明**：

| 配置项 | 说明 |
|-------|------|
| `RequireAuthentication` | 是否要求身份认证（`false` 允许匿名连接） |

---

### 11. App（应用程序）

应用程序基础配置。

```json
{
  "App": {
    "SelfUrl": "http://*:19001",
    "CorsOrigins": "*",
    "EnableSwagger": true
  }
}
```

**配置项说明**：

| 配置项 | 说明 | 默认值 |
|-------|------|--------|
| `SelfUrl` | 应用监听地址 | `http://*:19001` |
| `CorsOrigins` | CORS 允许来源（逗号分隔） | `*`（允许所有） |
| `EnableSwagger` | 启用 Swagger UI（ARM32 建议设为 `false`） | `true` |

**注意**：ARM32 平台上启用 Swagger 可能导致堆栈溢出，建议设为 `false`。

---

### 12. FrontendApps（前端应用）

前端 SPA 应用路由配置。

```json
{
  "FrontendApps": {
    "App": {
      "PhysicalPath": "wwwroot/app",
      "RequestPath": "",
      "DefaultFile": "index.html"
    },
    "Admin": {
      "PhysicalPath": "wwwroot/admin",
      "RequestPath": "/admin",
      "DefaultFile": "index.html"
    },
    "Web": {
      "PhysicalPath": "wwwroot/web",
      "RequestPath": "/web",
      "DefaultFile": "index.html"
    }
  }
}
```

**配置项说明**：

| 应用 | 访问路径 | 说明 |
|-----|---------|------|
| `App` | `/` | 主应用 |
| `Admin` | `/admin` | 管理界面 |
| `Web` | `/web` | Web 应用 |

每个应用都配置了 SPA 回退，未匹配的路由将返回 `index.html`。

---

### 13. JwtOptions（JWT 认证）

JWT 访问令牌配置。

```json
{
  "JwtOptions": {
    "Issuer": "https://www.astraeus.cn",
    "Audience": "https://www.astraeus.cn",
    "SecurityKey": "tT9fj%2K{,pPdR8!K1X^AST*B&Hv+bMF4z$v#4xVnW]2gU~8m?J7kZ3|L5h1f-G+6S0Tyz",
    "ExpiresMinuteTime": 86400
  }
}
```

**配置项说明**：

| 配置项 | 说明 | 默认值 |
|-------|------|--------|
| `Issuer` | 发行者标识 | `https://www.astraeus.cn` |
| `Audience` | 受众标识 | `https://www.astraeus.cn` |
| `SecurityKey` | 签名密钥（生产环境必须更换） | - |
| `ExpiresMinuteTime` | 过期时间（分钟） | `86400`（60 天） |

---

### 14. RefreshJwtOptions（刷新令牌）

JWT 刷新令牌配置。

```json
{
  "RefreshJwtOptions": {
    "Issuer": "https://www.astraeus.cn",
    "Audience": "https://www.astraeus.cn",
    "SecurityKey": "aE7h3!xJfA1^G-8mPkU*D5(H2bR&K+F6Q[3qW$Mp9]VjY0z5LN@AST1tC|g4O8wRsX~72",
    "ExpiresMinuteTime": 172800
  }
}
```

**配置项说明**：与 `JwtOptions` 相同，`ExpiresMinuteTime` 默认 `172800`（120 天）。

---

### 15. RbacOptions（RBAC 模块）

基于角色的访问控制模块配置。

```json
{
  "RbacOptions": {
    "AdminPassword": "123456",
    "EnableCaptcha": false,
    "EnableRegister": false,
    "EnableDataBaseBackup": false
  }
}
```

**配置项说明**：

| 配置项 | 说明 |
|-------|------|
| `AdminPassword` | 超级管理员默认密码 |
| `EnableCaptcha` | 启用验证码 |
| `EnableRegister` | 启用用户注册 |
| `EnableDataBaseBackup` | 启用定时数据库备份 |

---

### 16. PointDataCleanup（点位数据清理）

传感器时序数据清理任务。

```json
{
  "PointDataCleanup": {
    "Enabled": true,
    "CronExpression": "0 0 2 * * *",
    "RetentionDays": 30,
    "BatchSize": 500,
    "EnableDetailedLogging": false,
    "ExcludedSensorKeys": ["jixie"]
  }
}
```

**配置项说明**：

| 配置项 | 说明 | 默认值 |
|-------|------|--------|
| `Enabled` | 是否启用任务 | `true` |
| `CronExpression` | Cron 表达式 | `"0 0 2 * * *"`（每天凌晨 2 点） |
| `RetentionDays` | 数据保留天数 | `30` |
| `BatchSize` | 批量删除大小 | `500` |
| `EnableDetailedLogging` | 启用详细日志 | `false` |
| `ExcludedSensorKeys` | 排除的传感器 Key（永久保留） | `["jixie"]` |

---

### 17. Serilog（日志配置）

Serilog 结构化日志配置。

```json
{
  "Serilog": {
    "MinimumLevel": {
      "Default": "Information",
      "Override": {
        "Microsoft": "Warning",
        "Hangfire": "Information"
      }
    },
    "WriteTo": [
      {
        "Name": "Async",
        "Args": {
          "configure": [
            {
              "Name": "File",
              "Args": {
                "path": "logs/all/log-.txt",
                "rollingInterval": "Day",
                "retainedFileCountLimit": 10,
                "fileSizeLimitBytes": 20971520,
                "rollOnFileSizeLimit": true
              }
            }
          ]
        }
      }
    ]
  }
}
```

**日志级别**：`Verbose`, `Debug`, `Information`, `Warning`, `Error`, `Fatal`

**日志输出位置**：

- 全部日志：`logs/all/log-.txt`
- 错误日志：`logs/error/errorlog-.txt`
- 控制台（可选）

---

## 后台任务配置

系统使用 Hangfire 管理后台任务，以下任务可通过配置启用/禁用：

| 任务名称 | 配置节 | 说明 |
|---------|-------|------|
| VoiceprintCaptureJob | `VoiceprintCaptureJob` | 声纹音频采集 |
| VoiceprintProcessedCleanupJob | `VoiceprintProcessedCleanupJob` | 已处理音频清理 |
| PointDataCleanup | `PointDataCleanup` | 传感器数据清理 |
| CameraResourceCleanup | `CameraResourceCleanup` | 摄像机资源清理 |
| ISAPIResourceCleanup | `ISAPIResourceCleanup` | ISAPI 资源清理 |
| PatrolSystemCleanup | `PatrolSystemCleanup` | 巡检系统清理 |
| VisualGatewayHealthCheck | `VisualGatewayHealthCheck` | 视觉网关健康检查 |
| PatrolJobManager | `PatrolJobManager` | 巡检任务管理 |
| EnvironmentDetectionCollection | `EnvironmentDetectionCollection` | 环境检测数据采集 |
| InfraredTemperatureCollection | `InfraredTemperatureCollection` | 红外温度采集 |

---

## 环境变量

系统支持通过环境变量覆盖配置：

```bash
# 数据库连接
export DbConnOptions__Url="Host=localhost;Database=ast_db"

# Redis 连接
export Redis__IsEnabled="true"
export Redis__Configuration="localhost:6379"

# 应用 URL
export App__SelfUrl="http://localhost:19001"

# JWT 密钥（生产环境）
export JwtOptions__SecurityKey="your-production-secret-key"
```

**环境变量命名约定**：`{Section}__{Key}`（双下划线）

---

## 相关文档

- [[部署指南]] — 部署模式和配置选择
- [[Hangfire 后台任务索引]] — Hangfire 任务详细说明
- [[IEC61850 集成指南]] — 工业协议配置
- [[声纹分析模块]] — 声纹采集和识别配置

---

**最后更新**：2026-06-03
**适用版本**：v2.0.0+
