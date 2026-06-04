# BackupDataBaseJob — 数据库备份任务

## 基本信息

- **任务名称**：`BackupDataBaseJob`
- **模块位置**：`module/rbac/Yi.Framework.Rbac.Application/Jobs/`
- **任务类型**：`HangfireBackgroundWorkerBase` - Hangfire后台任务
- **Cron表达式**：`0 0 0,12 * * ?`（每天00点和12点执行）
- **命名空间**：`Yi.Framework.Rbac.Application.Jobs`

## 任务概述

BackupDataBaseJob 是RBAC模块的数据库定时备份任务。它每天在00点和12点自动执行数据库备份操作，确保数据安全。

## 核心职责

1. **定时备份** - 按照Cron表达式定时执行备份
2. **配置控制** - 通过配置开关启用/禁用备份
3. **日志记录** - 记录备份开始和完成日志

## 依赖注入

| 依赖 | 职责 |
|------|------|
| `ISqlSugarDbContext` | 数据库上下文（提供BackupDataBase方法） |
| `IOptions<RbacOptions>` | RBAC配置选项 |

## 配置选项

### RbacOptions

```csharp
public class RbacOptions
{
    public bool EnableDataBaseBackup { get; set; }  // 是否启用数据库备份
}
```

### appsettings.json 配置

```json
{
  "RbacOptions": {
    "EnableDataBaseBackup": true
  }
}
```

## 执行计划

### Cron表达式

```
0 0 0,12 * * ?
```

**说明**：
- `0 0 * * *` - 每天00点（午夜）
- `0 0 12 * * *` - 每天12点（中午）
- 两个时间点都会执行备份

### 执行时间

- **第一次**：每天 00:00
- **第二次**：每天 12:00

## 工作流程

```
1. 检查配置开关
   │
   ├─ if (_options.Value.EnableDataBaseBackup)
   │  └─ 继续执行
   │
2. 记录开始日志
   │
   ├─ logger.LogWarning("正在进行数据库备份")
   │
3. 执行数据库备份
   │
   ├─ _dbContext.BackupDataBase()
   │  │
   ├─ 备份整个数据库到文件
   ├─ 生成备份文件
   └─ 验证备份完整性
   │
4. 记录完成日志
   │
   └─ logger.LogWarning("数据库备份已完成")
```

## 依赖方法

### SqlSugarDbContext.BackupDataBase()

**功能**：
执行数据库备份操作。

**备份内容**：
- 所有表结构和数据
- 生成SQL备份文件
- 支持多种数据库类型

## 日志记录

### 警告日志 (LogWarning)

```csharp
// 备份开始
"正在进行数据库备份"

// 备份完成
"数据库备份已完成"
```

## 性能考虑

### 备份时间

- **选择低峰期**：00:00和12:00通常是系统访问低峰期
- **避免高峰期**：不在业务高峰期执行备份
- **备份时长**：取决于数据库大小

### 存储空间

- **定期清理**：建议定期清理旧备份文件
- **异地备份**：建议将备份文件同步到异地存储
- **备份验证**：定期验证备份文件的完整性

## 错误处理

当前实现比较简单，主要通过日志记录。建议增强：

### 建议的改进

1. **异常捕获**
```csharp
public override Task DoWorkAsync(CancellationToken cancellationToken)
{
    try
    {
        if (_options.Value.EnableDataBaseBackup)
        {
            _dbContext.BackupDataBase();
            logger.LogWarning("数据库备份已完成");
        }
    }
    catch (Exception ex)
    {
        logger.LogError(ex, "数据库备份失败");
        // 可选：发送告警通知
    }
}
```

2. **备份验证**
```csharp
// 备份完成后验证文件
var backupFile = "backup_file_path";
if (!File.Exists(backupFile))
{
    logger.LogError("备份文件未生成: {BackupFile}", backupFile);
}
```

3. **备份通知**
```csharp
// 发送备份结果通知
await _notificationService.SendAsync("数据库备份", 
    isSuccess ? "成功" : "失败");
```

## 维护建议

### 1. 定期检查备份

- 每周检查备份是否正常执行
- 验证备份文件的完整性
- 测试备份恢复流程

### 2. 存储管理

- 监控备份文件占用的磁盘空间
- 定期清理过期备份文件（如保留最近30天）
- 考虑增量备份策略

### 3. 异地存储

- 将备份文件同步到异地服务器
- 使用云存储服务（如OSS、S3等）
- 确保备份文件的地理冗余

### 4. 恢复演练

- 定期进行备份恢复演练
- 验证备份的有效性
- 文档化恢复流程

## 使用场景

### 1. 定时自动备份

系统每天自动在00:00和12:00执行备份，无需人工干预。

### 2. 手动触发备份

虽然主要依赖定时任务，但也可以通过Hangfire Dashboard手动触发：
- 访问 `/hangfire` 界面
- 找到"数据库备份"任务
- 点击"触发"按钮手动执行

### 3. 配置控制

如需临时禁用备份（如维护期间）：
```json
{
  "RbacOptions": {
    "EnableDataBaseBackup": false
  }
}
```

## 相关任务

- [PointDataCleanupJob] - 点位数据清理任务
- [VoiceprintProcessedCleanupJob] - 声纹处理后清理任务
- [PatrolSystemCleanupJob] - 巡检系统清理任务

## 注意事项

1. **数据库大小** - 大型数据库备份可能需要较长时间
2. **磁盘空间** - 确保足够的磁盘空间用于存储备份
3. **权限要求** - 执行备份需要数据库文件系统读写权限
4. **并发控制** - 备份期间避免大型数据库操作
5. **业务影响** - 备份可能会略微影响数据库性能

---

> **最后更新**：2026-06-04  
> **源码位置**：`module/rbac/Yi.Framework.Rbac.Application/Jobs/BackupDataBaseJob.cs`
> **任务ID**：`数据库备份` (Hangfire RecurringJobId)
