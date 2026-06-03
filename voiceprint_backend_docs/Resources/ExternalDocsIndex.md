# 外部文档索引

本文档提供项目外部资源的集中索引，包括前端流程图、实现方案文档和接口文档。这些文档与 Obsidian 知识库相互补充，形成完整的技术文档体系。

---

## 📊 前端 API 流程图

**位置**: `docs/data-flow-diagrams/`

包含 6 个交互式 HTML 流程图，展示前端 23 个 API 的调用流程和数据交互。

| 序号 | 流程图文件 | 涉及的 API | 描述 |
|------|-----------|-----------|------|
| 1 | [account-login.html](../../../data-flow-diagrams/account-login.html) | `/account/login` | 用户登录流程 |
| 2 | [account-get-info.html](../../../data-flow-diagrams/account-get-info.html) | `/account` | 获取账户信息流程 |
| 3 | [voiceprint-dashboard.html](../../../data-flow-diagrams/voiceprint-dashboard.html) | 仪表板相关 8 个 API | 首页仪表板数据加载流程 |
| 4 | [voiceprint-assets.html](../../../data-flow-diagrams/voiceprint-assets.html) | 设备台账相关 6 个 API | 设备台账查询和操作流程 |
| 5 | [voiceprint-alarms.html](../../../data-flow-diagrams/voiceprint-alarms.html) | 告警管理相关 4 个 API | 告警记录查询和处理流程 |
| 6 | [voiceprint-analysis.html](../../../data-flow-diagrams/voiceprint-analysis.html) | 声纹分析相关 3 个 API | 声纹分析和报告生成流程 |

**访问方式**:
- 本地访问：在浏览器中打开对应的 HTML 文件
- 包含完整的交互式流程图，支持节点点击和路径高亮

**相关文档**:
- [前端 23 个接口文档](../../../frontend-api-interfaces.md) - 所有接口的详细说明

---

## 📋 实现方案文档

**位置**: `docs/voiceprint/`

包含 10 篇详细的实现方案和设计文档，涵盖系统架构、模块设计和问题分析。

| 序号 | 文档名称 | 描述 | 相关模块 |
|------|---------|------|---------|
| 1 | [00-系统架构与功能实现](../../../voiceprint/00-系统架构与功能实现.md) | 系统整体架构和核心功能实现说明 | 全局 |
| 2 | [01-intelli-substation-voiceprint-backend改造方案](../../../voiceprint/01-intelli-substation-voiceprint-backend改造方案.md) | 后端系统改造方案和迁移计划 | ABP框架 |
| 3 | [02-python-voiceprint-service方案](../../../voiceprint/02-python-voiceprint-service方案.md) | Python 声纹分析服务设计方案 | 声纹分析 |
| 4 | [03-raspberry-pi-agent方案](../../../voiceprint/03-raspberry-pi-agent方案.md) | 树莓派边缘采集代理方案 | 边缘计算 |
| 5 | [分析报表需求文档](../../../voiceprint/分析报表需求文档.md) | 声纹分析报表的功能需求和设计 | 报表系统 |
| 6 | [声纹数据库实体介绍文档](../../../voiceprint/声纹数据库实体介绍文档.md) | 声纹相关数据库实体和关系说明 | 数据模型 |
| 7 | [多设备高频采集下音频时长不足原因分析](../../../voiceprint/多设备高频采集下音频时长不足原因分析.md) | 音频采集时长问题的技术分析 | 音频采集 |
| 8 | [已处理音频下载接口优化方案](../../../voiceprint/已处理音频下载接口优化方案.md) | 批量下载接口的优化方案 | API优化 |
| 9 | [已处理音频定期清理功能代码解释](../../../voiceprint/已处理音频定期清理功能代码解释.md) | 音频清理后台任务的实现说明 | Hangfire |
| 10 | [需求变更-算法切换与1分钟分段识别](../../../voiceprint/需求变更-算法切换与1分钟分段识别.md) | 算法模式切换和分段识别的变更说明 | 声纹算法 |

**访问方式**:
- 本地访问：直接在 Obsidian 或 Markdown 编辑器中打开
- 包含完整的技术方案、代码示例和架构说明

**相关文档**:
- Obsidian 知识库：[系统架构](../Architecture/系统架构.md)
- Obsidian 知识库：[声纹分析模块](../Modules/声纹分析模块.md)

---

## 🔌 前端接口文档

**位置**: `docs/frontend-api-interfaces.md`

完整的 23 个前端使用接口文档，包含路径、方法、功能描述和使用位置。

**文档内容**:
- **接口汇总**: 按模块分类的接口统计
- **详细列表**: 每个接口的完整说明
- **调用方式**: 前端请求封装和类型定义
- **WebSocket 连接**: 实时通信配置

**模块分类**:
| 模块 | 接口数量 | 主要功能 |
|------|---------|---------|
| 账户管理 | 2 | 登录、获取账户信息 |
| 告警管理 | 4 | 告警统计、列表、处理 |
| 仪表板 | 8 | 巡视统计、设备状态、报表导出 |
| 设备台账 | 6 | 设备树、趋势、异常分布 |
| 声纹分析 | 3 | 标准音频库、测试音频、报告生成 |

**访问方式**:
- [查看完整文档](../../../frontend-api-interfaces.md)

**相关文档**:
- [前端 API 流程图](#前端-api-流程图) - 可视化展示接口调用流程
- [Obsidian 知识库首页](../README.md) - 所有 API 文档索引

---

## 🔗 与 Obsidian 知识库的关系

### 文档定位

| 文档类型 | 位置 | 用途 | 维护方式 |
|---------|------|------|---------|
| **外部文档** | `docs/` 目录 | 详细的实现方案、流程图和接口文档 | 独立 Markdown 文件 |
| **Obsidian 知识库** | `docs/voiceprint_docs/` | 结构化的技术文档和知识管理 | Obsidian 笔记链接 |

### 双向链接指引

#### 从外部文档到 Obsidian 知识库

在 `docs/` 目录下的文档中引用 Obsidian 知识库：

```markdown
参见 Obsidian 知识库：
- [系统架构](voiceprint_docs/voiceprint_backend_docs/Architecture/系统架构.md)
- [API 文档](voiceprint_docs/voiceprint_backend_docs/ApiDocs/README.md)
- [部署指南](voiceprint_docs/voiceprint_backend_docs/DeploymentGuide/部署指南.md)
```

#### 从 Obsidian 知识库到外部文档

在 Obsidian 笔记中引用外部文档：

```markdown
参见外部实现方案：
- [系统架构说明](../../../voiceprint/00-系统架构与功能实现.md)
- [Python 声纹服务方案](../../../voiceprint/02-python-voiceprint-service方案.md)
- [前端接口文档](../../../frontend-api-interfaces.md)
```

### 文档更新策略

1. **外部文档更新**:
   - 实现方案变更时更新 `docs/voiceprint/` 文档
   - API 变更时更新 `docs/frontend-api-interfaces.md`
   - 流程变更时重新生成 `docs/data-flow-diagrams/` HTML 文件

2. **Obsidian 知识库同步**:
   - 定期将外部文档的关键信息同步到 Obsidian
   - 使用双向链接保持文档关联
   - 在 Obsidian 中创建摘要和索引文档

3. **版本控制**:
   - 所有文档都在 Git 版本控制下
   - 使用提交消息记录文档更新
   - 重要变更添加版本标签

---

## 📌 快速导航

### 按主题查找

**系统架构和设计**:
- [系统架构与功能实现](../../../voiceprint/00-系统架构与功能实现.md)
- [后端改造方案](../../../voiceprint/01-intelli-substation-voiceprint-backend改造方案.md)
- Obsidian: [系统架构](../Architecture/系统架构.md)

**前端集成**:
- [前端 23 个接口文档](../../../frontend-api-interfaces.md)
- [前端 API 流程图](../../../data-flow-diagrams/)
- Obsidian: [API 文档索引](../API/README.md)

**声纹分析**:
- [Python 声纹服务方案](../../../voiceprint/02-python-voiceprint-service方案.md)
- [分析报表需求文档](../../../voiceprint/分析报表需求文档.md)
- [声纹数据库实体介绍](../../../voiceprint/声纹数据库实体介绍文档.md)
- Obsidian: [声纹分析模块](../Modules/声纹分析模块.md)

**问题分析**:
- [音频时长不足原因分析](../../../voiceprint/多设备高频采集下音频时长不足原因分析.md)
- [下载接口优化方案](../../../voiceprint/已处理音频下载接口优化方案.md)
- Obsidian: [常见问题](../Troubleshooting/常见问题.md)

---

## 🛠️ 文档贡献

### 添加新的外部文档

1. 在 `docs/` 目录下创建新的 Markdown 文件
2. 在本索引中添加文档链接和描述
3. 在相关 Obsidian 笔记中添加双向链接
4. 提交 Git 变更

### 更新现有文档

1. 更新源文档内容
2. 检查并更新相关链接
3. 同步变更到 Obsidian 知识库
4. 记录更新日志

---

## 📚 相关资源

- [项目 README](../../../README.md)
- [CLAUDE.md](../../../CLAUDE.md) - 项目开发指南
- [Obsidian 知识库首页](../README.md)
- [Resources 目录说明](README.md) - 本地资源文件管理

---

**最后更新**: 2026-06-03  
**维护者**: 后端开发团队  
**文档版本**: v1.0.0
