# 大文本存储管理器 (LargeTextStorageManager)

## 概述

**LargeTextStorageManager** 处理超过 TDengine 字符串字段长度限制（4000 字节）的大文本数据，自动将大文本存储到文件系统中，并在数据库中记录文件路径。

**源码位置**：`module/ast-intellisubdata/Ast.IntelliSubData.Application/Managers/LargeTextStorageManager.cs`

## 核心功能

### 1. 大文本处理

**ProcessLargeTextAsync** - 智能处理大文本数据
- 文本长度 ≤ 4000 字节：直接存储到数据库
- 文本长度 > 4000 字节：存储到文件系统

### 2. 文件路径生成

**新格式**（推荐）：
```
{StorageDirectory}/{DeviceId}/{SensorKey}/{Ts}_{Property}_{PointId}.data
```

**旧格式**（兼容）：
```
{StorageDirectory}/{yyyyMMdd_HHmmss}_{Guid}.data
```

### 3. 文件内容读取

**ReadFileContentAsync** - 读取文件内容
- 支持 `file:` 前缀的路径识别
- 自动读取文件并返回内容
- 文件不存在时返回 null

## 存储策略

### 目录结构
```
{StorageDirectory}/
  ├── {DeviceId}/
  │   ├── {SensorKey}/
  │   │   ├── {Ts}_{Property}_{PointId}.data
  │   │   └── ...
  │   └── ...
  └── ...
```

### 文件命名规则
- 清理非法字符（替换为下划线）
- 移除开头和结尾的空格或点
- 避免 Windows 保留名称（CON、PRN、AUX 等）
- 限制长度不超过 200 字符

### 路径清理
- 替换所有文件系统不允许的字符
- 统一使用下划线分隔
- 确保跨平台兼容性

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

**参数说明**：
- `Enabled` - 是否启用大文本存储
- `MaxStringLength` - TDengine 字符串字段最大长度（字节）
- `StorageDirectory` - 文件存储根目录
- `FileExtension` - 文件扩展名
- `AutoCreateDirectory` - 是否自动创建目录

## 使用示例

### 处理大文本
```csharp
var storageDto = new LargeTextStorageDto(
    largeTextContent,
    "device-001",
    "temperature-sensor",
    "description",
    DateTime.Now,
    "point-guid-123"
);

var processedText = await _largeTextStorageManager.ProcessLargeTextAsync(storageDto);
// 返回: "file:device-001/temperature-sensor/20250603220000_description_point-guid-123.data"
```

### 读取文件内容
```csharp
var filePath = "file:device-001/temperature-sensor/20250603220000_description_point-guid-123.data";
var content = await _largeTextStorageManager.ReadFileContentAsync(filePath);
// 返回原始大文本内容
```

## 性能优化

### 推荐做法
- 批量处理时减少文件 I/O 次数
- 合理设置 `MaxStringLength` 避免频繁触发文件存储
- 定期清理不再需要的文件

### 注意事项
- 文件存储需要磁盘空间监控
- 备份时需要同时备份数据库和文件目录
- 删除数据时需要同步删除文件

## 相关文档

- [[AstPointDataService]]
- [[TDengine 集成]]
- [[数据存储策略]]
- [[PointData 实体]]

---

**最后更新**：2026-06-03
