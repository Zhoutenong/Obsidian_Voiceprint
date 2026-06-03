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
- [[数据库设计]] — 数据库结构说明

### 🔧 核心模块

| 模块 | 说明 | 状态 |
|------|------|------|
| `rbac` | 基于角色的访问控制 | 🟡 学习中 |
| `ast-intellisub` | 智能巡检/变电站监控 | 🟡 学习中 |
| `ast-intellisubdata` | 传感器时序数据 | 🟡 学习中 |
| `ast-voiceprint` | 声纹分析 | 🟡 学习中 |
| `isapi` | IEC61850/IEC104 集成 | 🟡 学习中 |

---

## 目录结构

```
Classes/            类文档（核心服务、实体、DTO）
Modules/            模块文档（按业务模块组织）
  ├─ rbac/          认证授权模块
  ├─ ast-intellisub/ 变电站监控模块
  ├─ ast-intellisubdata/ 传感器数据模块
  ├─ ast-voiceprint/ 声纹分析模块
  └─ isapi/         工业协议集成模块
Pipelines/          数据流程文档
  ├─ Audio/         音频处理流程
  └─ DataReport/    数据上报流程
Concepts/           架构概念与设计模式
Architecture/       架构文档
Hangfire/           后台任务文档
Diagrams/           架构图和流程图
Issues/             Bug 案例与调试
Templates/          笔记模板
Resources/          图片等资源
```

---

## 模块详细分类

### ✅ RBAC 模块（认证授权）

#### 核心
[[认证系统]] · [[授权系统]] · [[JWT令牌]] · [[OAuth集成]]

#### 用户管理
[[用户管理]] · [[角色管理]] · [[权限管理]]

---

### ✅ ast-intellisub 模块（变电站监控）

#### 设备管理
[[设备管理]] · [[设备类型]] · [[设备状态]]

#### 巡检系统
[[巡检路线]] · [[巡检任务]] · [[巡检记录]]

#### 告警系统
[[实时告警]] · [[告警规则]] · [[告警历史]]

#### 摄像机集成
[[摄像机管理]] · [[视频流]] · [[抓拍功能]]

---

### ✅ ast-intellisubdata 模块（时序数据）

#### 数据采集
[[传感器数据]] · [[数据采集]] · [[点位管理]]

#### 数据存储
[[TDengine集成]] · [[数据保留策略]] · [[数据清理]]

---

### ✅ ast-voiceprint 模块（声纹分析）

#### 音频处理
[[音频采集]] · [[声纹识别]] · [[标准音频库]]

#### 报告生成
[[报告生成]] · [[DOCX导出]] · [[数据分析]]

---

### ✅ isapi 模块（工业协议）

#### IEC61850
[[IEC61850协议]] · [[数据上报]] · [[模型配置]]

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

## 后台任务 (Hangfire)

### 定时任务
| 任务 | Cron表达式 | 说明 | 状态 |
|------|-----------|------|------|
| VoiceprintCaptureJob | `0 */10 * * * *` | 音频采集 | 🟡 |
| VoiceprintProcessedCleanupJob | 手动 | 清理已处理音频 | 🟡 |
| PointDataCleanupJob | 可配置 | 数据清理 | 🟡 |

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

- [[ABP框架文档]] — ABP Framework 官方文档
- [[项目CLAUDE.md]] — 项目开发指南
- [[Obsidian使用指南]] — Obsidian 高级用法
- [[学习路线图]] — 系统学习路径

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
