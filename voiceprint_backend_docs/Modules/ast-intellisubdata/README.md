# ast-intellisubdata 模块文档

## 概述

**ast-intellisubdata** 是智能变电站监控系统的时序数据管理模块，负责传感器点位数据的采集、存储、查询和分析。

## 模块架构

### 核心层次

```
┌─────────────────────────────────────────┐
│   Application.Contracts (接口层)         │
│   - DTOs                                │
│   - Service Interfaces                  │
└─────────────────────────────────────────┘
                    ↓
┌─────────────────────────────────────────┐
│   Application (应用服务层)                │
│   - Services                            │
│   - Managers                            │
└─────────────────────────────────────────┘
                    ↓
┌─────────────────────────────────────────┐
│   Domain (领域层)                        │
│   - Entities                            │
│   - Repositories                        │
└─────────────────────────────────────────┘
                    ↓
┌─────────────────────────────────────────┐
│   SqlSugarCore (数据访问层)              │
│   - Repositories Impl                   │
│   - Helpers                             │
└─────────────────────────────────────────┘
```

## 核心服务

### [[AstPointDataService]]
时序数据入库的核心服务，提供点位数据的完整 CRUD 操作。

**主要功能**：
- 数据创建和存储
- 多维度数据查询
- 历史数据聚合
- 过期数据清理

### 数据管理组件

#### [[ChildTableNameManager]]
- TDengine 子表名生成
- 命名规范化和长度控制
- 哈希冲突避免

#### [[TimeSeriesDbHelper]]
- 数据库兼容性处理
- 聚合查询 SQL 构建
- 跨数据库语法转换

#### [[LargeTextStorageManager]]
- 超大文本文件存储
- 文件路径管理
- 内容读写操作

### 数据模型

#### [[PointData]]
点位数据实体，支持 TDengine 超级表和普通表两种模式。

**字段设计**：
- Tag 结构：DeviceId + SensorKey + Property
- 数据字段：Val（数值）/ StrVal（文本）
- 扩展字段：GroupId、ExtInfo、AlarmLevel

## 数据存储策略

### TDengine 模式
- 超级表：`ast_pointdata`
- 子表自动创建：基于 Tag 组合
- 高性能时序查询

### SQLite 模式
- 普通表：`ast_pointdata`
- 索引优化：Tag 字段 + 时间戳
- 适合边缘设备部署

### 数据类型支持

| 类型 | 枚举 | 存储字段 | 适用场景 |
|------|------|---------|---------|
| 整数 | Int | Val | 计数值、状态码 |
| 浮点 | Float | Val | 温度、电压、电流 |
| 字符串 | String | StrVal | 状态描述、文本 |
| JSON | Json | StrVal | 复杂对象、数组 |
| 枚举 | Enum | Val | 预定义状态 |

## 查询能力

### 基础查询
- 时间范围过滤
- 设备/传感器维度
- 点位 ID 列表查询
- 最新数据查询

### 聚合查询
- 时间窗口聚合（1-10080 分钟）
- 统计指标：平均值、最大值、最小值
- 查询负载保护（最大 5000 点）
- 分页支持

### 原始数据查询
- 不分页原始数据导出
- 机械特性分析支持
- 自定义数据处理

## 性能优化

### 写入优化
- TDengine 批量插入
- 子表自动创建
- 大文本文件存储

### 查询优化
- Tag 索引（TDengine 天然支持）
- 时间范围索引（SQLite）
- 聚合查询缓存

### 存储优化
- 数据保留期管理
- 定期过期数据清理
- 大文本文件分离存储

## 配置选项

### LargeTextStorageOptions
```json
{
  "Enabled": true,
  "MaxStringLength": 4000,
  "StorageDirectory": "data/large-text",
  "FileExtension": ".data",
  "AutoCreateDirectory": true
}
```

### TDengine 配置
```json
{
  "DbType": "TDengine",
  "ConnectionString": "Server=127.0.0.1;Port=6030;Database=ast",
  "UserName": "root",
  "Password": "taosdata"
}
```

## 使用场景

### 1. 实时监控
- 最新数据刷新
- 多点位状态显示
- 告警级别监控

### 2. 历史分析
- 趋势曲线绘制
- 统计数据分析
- 报表生成

### 3. 设备管理
- 设备健康状态
- 传感器数据校验
- 数据质量监控

### 4. 机械特性分析
- 分合闸次数统计
- 动作时间分析
- 原始数据导出

## 相关文档

### 服务文档
- [[AstPointDataService]]
- [[ChildTableNameManager]]
- [[TimeSeriesDbHelper]]
- [[LargeTextStorageManager]]

### 数据模型
- [[PointData]]
- [[PointDataDto]]
- [[DataPointDto]]

### 功能文档
- [[时序数据管理]]
- [[TDengine 集成]]
- [[数据存储策略]]

## 快速链接

### 源码位置
- **服务**：`module/ast-intellisubdata/Ast.IntelliSubData.Application/Services/`
- **实体**：`module/ast-intellisubdata/Ast.IntelliSubData.Domain/Entities/`
- **仓储**：`module/ast-intellisubdata/Ast.IntelliSubData.SqlSugarCore/Repositories/`
- **工具**：`module/ast-intellisubdata/Ast.IntelliSubData.SqlSugarCore/Helpers/`

### 相关模块
- [[ast-intellisub]] - 变电站监控模块
- [[ast-voiceprint]] - 声纹分析模块
- [[isapi]] - 工业协议集成

---

**模块版本**：v1.0.0
**最后更新**：2026-06-03
