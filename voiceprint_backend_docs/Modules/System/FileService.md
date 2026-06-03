# FileService (文件服务)

## 概述
FileService 是系统的文件管理服务，提供文件上传、下载、删除和报表模板管理功能。该服务采用日期分目录存储策略，确保文件组织清晰且易于管理。

## 职责
- 处理文件上传并生成访问路径
- 提供文件下载功能，支持正确的 MIME 类型识别
- 管理报表模板文件的下载
- 支持文件删除操作
- 自动创建和管理文件存储目录结构

## 主要接口

### UploadFileAsync
上传文件到系统存储

**参数：**
- `file` (IFormFile) - 上传的文件对象

**返回：**
- `Task<FileInfoDto>` - 文件信息，包含文件名、大小、路径和上传时间

**特性：**
- `[OperLog("上传文件", OperEnum.Insert)]` - 记录操作日志

### DownloadFileAsync
下载已上传的文件

**参数：**
- `path` (string) - 文件路径（相对于 /files/ 目录）

**返回：**
- `Task<FileStreamResult>` - 文件流结果

### DownloadReportAsync
下载报表模板文件

**参数：**
- `fileName` (string) - 报表文件名

**返回：**
- `Task<FileStreamResult>` - 报表文件流结果

### DeleteFileAsync
删除指定文件

**参数：**
- `path` (string) - 文件路径（相对于 /files/ 目录）

**返回：**
- `Task<bool>` - 删除操作结果

**特性：**
- `[OperLog("删除文件", OperEnum.Delete)]` - 记录操作日志
- `[NonAction]` - 不能直接通过 HTTP 调用

## 存储结构

### 目录组织
```
store/
├── files/              # 用户上传文件
│   └── YYYYMMDD/      # 按日期分目录
│       └── filename.ext
└── report/            # 报表文件
    └── result/
        └── template.docx
```

### 路径配置
- **基础存储路径** - `AppDomain.CurrentDomain.BaseDirectory/store`
- **用户文件路径** - `store/files/YYYYMMDD/`
- **报表路径** - `store/report/result/`

## 数据传输对象

### FileInfoDto
```csharp
public class FileInfoDto
{
    public string FileName { get; set; }      // 文件名
    public long FileSize { get; set; }        // 文件大小（字节）
    public string FilePath { get; set; }      // 文件访问路径
    public DateTime UploadTime { get; set; }  // 上传时间
    public string ContentType { get; set; }    // 内容类型（MIME）
}
```

## 技术特性

### MIME 类型识别
使用 `FileExtensionContentTypeProvider` 自动识别文件类型：
- 支持常见文件扩展名的 MIME 类型映射
- 未识别类型默认返回 `application/octet-stream`

### 文件访问路径
上传后的文件通过 HTTP 直接访问：
```
/files/YYYYMMDD/filename.ext
```

### 异常处理
所有文件操作都包含完善的异常处理：
- 文件为空检查
- 文件不存在验证
- 详细的错误日志记录
- 用户友好的异常消息

## 认证与授权
- `[Authorize]` - 需要身份认证才能访问文件服务

## 使用示例

### 上传文件
```http
POST /api/app/file/upload-file
Content-Type: multipart/form-data

file: [binary]
```

### 下载文件
```http
GET /api/app/file/download-file?path=/files/20250603/document.pdf
```

### 下载报表
```http
GET /api/app/file/download-report?fileName=template.docx
```

## 相关服务
- [[IFileService]] - 文件服务接口
- [[FileExtensionContentTypeProvider]] - MIME 类型提供程序

## 相关文档
- [[文件存储配置]]
- [[操作日志系统]]
