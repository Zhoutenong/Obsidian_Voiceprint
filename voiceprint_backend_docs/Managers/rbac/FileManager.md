# FileManager — 文件管理器（RBAC）

## 基本信息

- **Manager名称**：`FileManager`
- **模块位置**：`module/rbac/Yi.Framework.Rbac.Domain/Managers/`
- **继承关系**：`DomainService` → `IFileManager`
- **生命周期**：依赖注入（瞬态）
- **命名空间**：`Yi.Framework.Rbac.Domain.Managers`

## Manager概述

FileManager 是RBAC模块的文件管理器，负责文件上传、保存和缩略图生成。它支持批量文件上传、自动目录创建，并能为图片文件自动生成缩略图。

## 核心职责

1. **批量文件上传** - 支持批量创建文件记录
2. **文件保存** - 将上传的文件保存到磁盘
3. **缩略图生成** - 为图片文件自动生成压缩缩略图
4. **目录管理** - 自动检查并创建文件存储目录

## 依赖注入

| 依赖 | 职责 |
|------|------|
| `IGuidGenerator` | GUID生成器 |
| `IRepository<FileAggregateRoot>` | 文件聚合根仓储 |
| `IImageCompressor` | 图片压缩器 |

## 核心方法

### 1. CreateAsync - 批量创建文件记录

**签名**：
```csharp
public async Task<List<FileAggregateRoot>> CreateAsync(IEnumerable<IFormFile> files)
```

**功能**：
批量上传文件并创建数据库记录。

**执行流程**：

```
1. 参数验证
   │
   ├─ if (files.Count() == 0)
   │  └─ 抛出 ArgumentException("文件上传为空！")
   │
2. 遍历每个文件
   │
   ├─ foreach (var file in files)
   │  │
   │  ├─ 创建 FileAggregateRoot
   │  │  ├─ Id = _guidGenerator.Create()
   │  │  ├─ FileName = file.FileName
   │  │  ├─ FileSizeKb = file.Length / 1024（转KB）
   │  │
   │  ├─ 检查并创建目录
   │  │  └─ data.CheckDirectoryOrCreate()
   │  │
   │  └─ 添加到列表
   │     └─ entities.Add(data)
   │
3. 批量插入数据库
   │
   └─ await _repository.InsertManyAsync(entities)
   
4. 返回文件实体列表
```

**异常抛出**：
- `ArgumentException` - 文件列表为空

**特点**：
- **批量处理** - 一次上传多个文件
- **自动目录** - 自动创建存储目录
- **单位转换** - 文件大小自动转换为KB

### 2. SaveFileAsync - 保存文件到磁盘

**签名**：
```csharp
public async Task SaveFileAsync(FileAggregateRoot file, Stream fileStream)
```

**功能**：
保存文件到磁盘，如果是图片则生成缩略图。

**执行流程**：

```
1. 获取文件保存路径
   │
   ├─ var filePath = file.GetSaveFilePath()
   │
2. 保存原始文件
   │
   ├─ using (var stream = new FileStream(filePath, FileMode.CreateNew))
   │  └─ await fileStream.CopyToAsync(stream)
   │
3. 重置流位置
   │
   └─ fileStream.Position = 0
   │
4. 判断文件类型
   │
   ├─ var fileType = file.GetFileType()
   │
5. 如果是图片，生成缩略图
   │
   ├─ if (FileTypeEnum.image == fileType)
   │  │
   │  ├─ 获取缩略图保存路径
   │  │  └─ var thumbnailSavePath = file.GetAndCheakThumbnailSavePath(true)
   │  │
   │  ├─ 尝试压缩图片
   │  │  ├─ var compressResult = await _imageCompressor.CompressAsync(...)
   │  │
   │  ├─ 处理压缩结果
   │  │  │
   │  │  ├─ Done → 使用压缩后的流
   │  │  ├─ Canceled → 抛出异常
   │  │  └─ 其他 → 抛出异常
   │  │
   │  ├─ 捕获异常
   │  │  │
   │  │  ├─ NotSupportedException → 记录信息日志
   │  │  └─ 其他异常 → 记录错误日志
   │  │
   │  └─ 最终使用原始流作为备份
   │     └─ compressImageStream = fileStream
   │
   └─ 保存缩略图
      └─ using (var stream = new FileStream(thumbnailSavePath, ...))
         └─ await compressImageStream.CopyToAsync(stream)
```

**图片处理逻辑**：

```csharp
// 1. 压缩图片
var compressResult = await _imageCompressor.CompressAsync(
    fileStream,           // 原始图片流
    file.GetMimeMapping() // MIME类型（如image/jpeg）
);

// 2. 检查压缩结果
if (compressResult.State == ImageProcessState.Done)
{
    // 压缩成功，获取压缩后的流
    compressImageStream = compressResult.Result;
}
else if (compressResult.State == ImageProcessState.Canceled)
{
    // 图片无法进一步压缩（可能已经很小）
    throw new NotSupportedException($"当前图片无法再进行压缩");
}
else
{
    // 图片不支持压缩
    throw new NotSupportedException($"当前图片不支持压缩");
}

// 3. 如果压缩失败，使用原始文件
catch (Exception)
{
    compressImageStream = fileStream; // 回退到原始流
}

// 4. 保存缩略图
await compressImageStream.CopyToAsync(thumbnailStream);
```

## 相关实体

### FileAggregateRoot - 文件聚合根

```csharp
public class FileAggregateRoot : Entity<Guid>
{
    public Guid Id { get; protected set; }
    public string FileName { get; set; }      // 文件名
    public decimal FileSizeKb { get; set; }   // 文件大小（KB）
}
```

**关键方法**：
- `GetSaveFilePath()` - 获取文件保存路径
- `GetAndCheakThumbnailSavePath(bool create)` - 获取/创建缩略图路径
- `CheckDirectoryOrCreate()` - 检查并创建目录
- `GetFileType()` - 获取文件类型枚举
- `GetMimeMapping()` - 获取MIME类型

## 文件存储结构

```
存储根目录/
├─ 原始文件/
│  ├─ {year}/{month}/{day}/
│  │  ├─ {guid}_filename.jpg
│  │  ├─ {guid}_document.pdf
│  │  └─ ...
└─ 缩略图/
   └─ {year}/{month}/{day}/
      ├─ {guid}_filename_thumb.jpg
      └─ ...
```

**命名规则**：
- 原始文件：`{GUID}_{FileName}`
- 缩略图：`{GUID}_{FileName}_thumb`

## 图片处理

### 支持的图片类型

根据 `FileTypeEnum.image` 判断，通常包括：
- JPEG
- PNG
- GIF
- BMP
- WEBP

### 缩略图生成策略

1. **尝试压缩** - 使用 `IImageCompressor` 压缩图片
2. **失败回退** - 如果压缩失败，使用原始文件
3. **双重保存** - 原始文件和缩略图都保存

### 错误处理

```csharp
try
{
    // 尝试压缩
    var compressResult = await _imageCompressor.CompressAsync(...);
}
catch (NotSupportedException ex)
{
    // 记录信息日志（图片无法压缩）
    Logger.LogInformation(ex, ex.Message);
}
catch (Exception ex)
{
    // 记录错误日志
    Logger.LogError(ex, ex.Message);
}
finally
{
    // 无论如何都保存一个文件（可能是原始文件）
    compressImageStream = fileStream;
}
```

## 使用场景

### 1. 单文件上传

```csharp
// Controller中
var file = Request.Form.Files["file"];

// 1. 创建文件记录
var fileEntities = await _fileManager.CreateAsync(new[] { file });
var fileEntity = fileEntities.First();

// 2. 保存文件
using (var stream = file.OpenReadStream())
{
    await _fileManager.SaveFileAsync(fileEntity, stream);
}

// 3. 返回文件ID
return Ok(new { FileId = fileEntity.Id });
```

### 2. 批量文件上传

```csharp
// Controller中
var files = Request.Form.Files;

// 1. 批量创建文件记录
var fileEntities = await _fileManager.CreateAsync(files);

// 2. 逐个保存文件
foreach (var fileEntity in fileEntities)
{
    var file = files[f => f.Name == fileEntity.FileName];
    using (var stream = file.OpenReadStream())
    {
        await _fileManager.SaveFileAsync(fileEntity, stream);
    }
}

// 3. 返回文件ID列表
return Ok(new { FileIds = fileEntities.Select(f => f.Id) });
```

### 3. 图片上传与缩略图

```csharp
// 上传图片
var imageFile = Request.Form.Files["image"];

// 创建记录（会自动创建目录）
var fileEntities = await _fileManager.CreateAsync(new[] { imageFile });
var fileEntity = fileEntities.First();

// 保存（会自动生成缩略图）
using (var stream = imageFile.OpenReadStream())
{
    await _fileManager.SaveFileAsync(fileEntity, stream);
}

// fileEntity.GetSaveFilePath()      -> 原始图片路径
// fileEntity.GetThumbnailSavePath() -> 缩略图路径
```

## 设计特点

1. **批量支持** - 支持一次上传多个文件
2. **自动目录** - 按日期自动创建目录结构
3. **图片优化** - 自动为图片生成缩略图
4. **容错设计** - 压缩失败时使用原始文件
5. **流处理** - 使用Stream避免内存溢出
6. **GUID命名** - 避免文件名冲突

## 性能考虑

### 内存优化

```csharp
// 使用Stream而非byte[]
// 避免大文件占用过多内存
using (var stream = file.OpenReadStream())
{
    await _fileManager.SaveFileAsync(fileEntity, stream);
}
```

### 批量插入

```csharp
// 使用InsertManyAsync而非循环Insert
await _repository.InsertManyAsync(entities);
```

### 目录创建

```csharp
// 文件实体负责创建目录
// 避免重复检查
data.CheckDirectoryOrCreate();
```

## 错误处理

### 参数验证

```csharp
if (files.Count() == 0)
{
    throw new ArgumentException("文件上传为空！");
}
```

### 图片压缩异常

- `NotSupportedException` - 图片不支持压缩或已是最小
- 其他异常 - 记录日志并回退到原始文件

## 日志记录

### 信息日志

```csharp
Logger.LogInformation(ex, ex.Message);
// 图片无法再进行压缩（不视为错误）
```

### 错误日志

```csharp
Logger.LogError(ex, ex.Message);
// 图片处理过程中发生的其他错误
```

## 安全考虑

1. **文件名** - 使用GUID避免文件名冲突和路径遍历
2. **目录隔离** - 按日期分目录存储
3. **大小限制** - 在上层Controller控制文件大小
4. **类型验证** - 通过MIME类型验证文件类型

## 相关服务

- `FileService` - 文件服务（应用层）
- `FileAggregateRoot` - 文件聚合根

## 相关实体

- `FileAggregateRoot` - 文件聚合根

## 扩展建议

### 1. 文件类型验证

```csharp
public async Task<List<FileAggregateRoot>> CreateAsync(
    IEnumerable<IFormFile> files,
    IEnumerable<string> allowedExtensions)
{
    foreach (var file in files)
    {
        var extension = Path.GetExtension(file.FileName);
        if (!allowedExtensions.Contains(extension))
        {
            throw new NotSupportedException($"不支持的文件类型: {extension}");
        }
    }
    // ...
}
```

### 2. 病毒扫描

```csharp
public async Task SaveFileAsync(FileAggregateRoot file, Stream fileStream)
{
    // 保存前扫描病毒
    await _virusScanner.ScanAsync(fileStream);

    // 保存文件
    // ...
}
```

### 3. 文件加密

```csharp
public async Task SaveFileAsync(FileAggregateRoot file, Stream fileStream)
{
    // 加密文件流
    var encryptedStream = _encryptionService.Encrypt(fileStream);

    // 保存加密后的文件
    // ...
}
```

---

> **最后更新**：2026-06-04  
> **源码位置**：`module/rbac/Yi.Framework.Rbac.Domain/Managers/FileManager.cs`  
> **接口定义**：`module/rbac/Yi.Framework.Rbac.Domain/Managers/IFileManager.cs`
