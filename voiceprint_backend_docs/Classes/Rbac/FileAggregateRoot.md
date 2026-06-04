# FileAggregateRoot — 文件管理聚合根

## 基本信息

- **实体名称**：`FileAggregateRoot`
- **数据库表**：`File`
- **模块位置**：`module/rbac/Yi.Framework.Rbac.Domain/Entities/`
- **继承关系**：`AggregateRoot<Guid>` → `IAuditedObject`

## 实体说明

文件管理聚合根用于系统中的文件管理，记录上传文件的元数据。支持文件类型检测、路径管理、缩略图生成等功能。

## 字段说明

### 主键与文件信息
- `Id` (Guid) - 主键（也是文件标识）
- `FileName` (string) - 原始文件名
- `FileSize` (decimal) - 文件大小（字节）
- `FilePath` (string) - 存储路径

### 审计信息
- `CreationTime` (DateTime) - 创建时间
- `CreatorId` (Guid?) - 创建者ID
- `LastModificationTime` (DateTime?) - 最后修改时间
- `LastModifierId` (Guid?) - 最后修改者ID

## 业务方法

### GetFileType() - 获取文件类型
根据文件名自动识别文件类型（图片、视频、文档等）

### GetSaveFilePath() - 获取存储路径
返回文件的完整存储路径

### GetQueryFileSavePath(bool?) - 获取查询路径
根据是否需要缩略图返回相应路径

## 业务规则
1. **文件类型识别**：根据扩展名自动分类
2. **路径组织**：按文件类型组织目录结构
3. **缩略图支持**：图片文件自动生成缩略图
4. **唯一标识**：使用GUID作为文件名避免冲突

## 文件类型分类
- 图片类型
- 视频类型  
- 文档类型
- 其他类型

## 相关服务
- FileService - 文件管理服务

## 存储路径规则
```
wwwroot/
├── image/        # 图片文件
├── video/        # 视频文件
├── document/     # 文档文件
└── thumbnail/     # 缩略图
```

## 业务价值
- 统一管理系统中的所有文件
- 提供文件上传、下载、删除功能
- 支持图片缩略图自动生成
- 按类型组织文件存储结构

---
> **最后更新**：2026-06-04
> **源码位置**：`module/rbac/Yi.Framework.Rbac.Domain/Entities/FileAggregateRoot.cs`
