# TDengine 子表名管理器 (ChildTableNameManager)

## 概述

**ChildTableNameManager** 负责 TDengine 子表名的生成和规范化，确保子表名符合 TDengine 的命名规范、唯一且具有可读性。

**源码位置**：`module/ast-intellisubdata/Ast.IntelliSubData.Application/Managers/ChildTableNameManager.cs`

## 核心职责

1. **命名规范化** - 清理非法字符，只保留字母、数字和下划线
2. **长度控制** - 确保表名不超过 TDengine 192 字符限制
3. **唯一性保证** - 使用哈希值避免冲突

## 命名规则

### 基础格式
```
ast_pointdata_{clean_deviceId}_{clean_sensorKey}_{clean_property}
```

### 清理规则
- 替换所有非字母、数字、下划线字符为下划线
- 转换为小写
- 空值替换为 `null_or_empty`

### 超长处理
当表名超过 192 字符时：
```
ast_pointdata_{truncated_tags}_{short_hash}
```

- `truncated_tags` - 截断的清理后标签组合
- `short_hash` - 8 位 SHA256 哈希值

## 使用示例

```csharp
var manager = new ChildTableNameManager(logger);

// 生成子表名
var childTableName = manager.GenerateChildTableName(
    "ast_pointdata",
    "device-001",
    "temperature-sensor",
    "current_value"
);
// 结果: ast_pointdata_device_001_temperature_sensor_current_value
```

## 相关文档

- [[AstPointDataService]]
- [[TDengine 集成]]
- [[时序数据管理]]

---

**最后更新**：2026-06-03
