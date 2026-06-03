# VoiceprintStandardAudioEntity

声纹标准音频库实体，用于存储设备正常运行时的标准音频样本，用于异常检测对比。

## 表信息

- **表名**: `vp_standard_audio_library`
- **主键**: `Id` (Guid)

## 字段列表

| 字段名 | 类型 | 说明 | 外键 |
|--------|------|------|------|
| `Id` | Guid | 主键 | - |
| `AudioName` | string(128) | 标准音频名称 | - |
| `AudioType` | string(64) | 音频类型/分类 | - |
| `AudioFilePath` | string(512) | 音频文件路径 | - |
| `AudioMd5` | string(64) | 音频文件 MD5 校验值 | - |
| `Description` | string(256) | 描述信息 | - |
| `CreatedAt` | DateTime | 创建时间 | - |
| `UpdatedAt` | DateTime? | 最近更新时间 | - |

## 关联实体

### 入站关系 (Inbound)

- **声纹分析任务** — 读取标准音频作为对比基准
- **VoiceprintAnalysisService** — 使用标准音频进行特征匹配

## 服务读写

### 读取服务

- **VoiceprintStandardAudioService** — 查询标准音频库列表
- **VoiceprintAnalysisService** — 加载标准音频进行对比分析

### 写入服务

- **VoiceprintStandardAudioService** — 创建、更新、删除标准音频记录
- **AdminImportService** — 批量导入标准音频库

## 业务规则

1. **音频类型**: 通常按设备类型或监测部位分类（如 "变压器"、"GIS"）
2. **文件校验**: `AudioMd5` 用于确保音频文件完整性
3. **对比基准**: 标准音频作为设备正常运行的声音基准，用于检测异常
4. **文件管理**: 音频文件存储在指定目录，数据库仅记录路径

## 源码位置

`module/ast-voiceprint/Ast.Voiceprint.Domain/Entities/VoiceprintStandardAudioEntity.cs`
