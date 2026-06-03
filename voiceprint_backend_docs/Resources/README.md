# Resources 目录说明

本目录用于存放文档所需的图片、配置文件示例、代码片段等资源文件。

## 目录结构

### images/
存放各类图片资源：

```
images/
├── architecture/         # 架构图
├── ui-screenshots/      # UI截图
├── diagrams/            # 各类图表
├── logos/               # Logo和品牌资源
└── icons/               # 图标资源
```

### configs/
存放配置文件示例：

```
configs/
├── appsettings.example.json           # 完整配置示例
├── appsettings.low-resource.json      # 低资源配置模式
├── appsettings.high-performance.json  # 高性能配置模式
├── docker-compose.example.yml         # Docker编排示例
└── nginx.example.conf                 # Nginx配置示例
```

### samples/
存放代码示例和样本数据：

```
samples/
├── api-requests/          # API请求示例（Postman, curl等）
├── iec61850/              # IEC61850模型文件示例
│   ├── .icd
│   └── .iid
├── audio-samples/         # 音频样本文件
└── test-data/             # 测试数据
```

### templates/
存放文档模板：

```
templates/
├── document-template.md   # 文档模板
├── api-doc-template.md    # API文档模板
└── diagram-template.md    # 图表模板
```

## 资源引用规范

### 在 Markdown 中引用图片

**Obsidian / Markdown 标准语法**：
```markdown
![架构图](images/architecture/system-overview.png)
```

**指定尺寸和位置**：
```markdown
<img src="images/architecture/system-overview.png" width="600" alt="系统架构图">
```

### 引用配置文件

**代码块引用**：
```markdown
参见配置示例：[`appsettings.json`](configs/appsettings.example.json)

```json
--8<-- "configs/appsettings.example.json"
```

### 引用代码示例

**直接引用**：
```markdown
[`API请求示例`](samples/api-requests/device-list.http)
```

**嵌入代码块**：
```markdown
```http
--8<-- "samples/api-requests/device-list.http"
```
```

## 文件命名约定

### 图片文件
- 使用描述性名称：`system-architecture.png`
- 避免空格和特殊字符
- 小写字母加连字符
- 包含版本号（如有必要）：`api-v2.0-flow.png`

### 配置文件
- 使用 `.example.` 或 `.sample.` 后缀
- 保留原始文件扩展名：`appsettings.example.json`
- 明确配置类型：`nginx.example.conf`

### 代码示例
- 使用描述性名称：`device-create-api.http`
- 包含 HTTP 方法的 API 示例使用 `.http` 扩展名
- 语言示例使用对应扩展名：`auth-service.cs`

## 资源管理规范

### 添加新资源
1. 确定资源类型和目录
2. 使用命名约定创建文件
3. 在本文档中更新索引
4. 在相关文档中添加引用

### 删除资源
1. 检查是否有文档引用该资源
2. 更新或删除相关引用
3. 删除资源文件
4. 更新本文档索引

### 资源版本控制
- 大型二进制文件（如高清截图）考虑使用 Git LFS
- 敏感配置信息不要提交到仓库
- 使用 `.gitignore` 排除敏感文件

## 当前资源索引

### 架构图
- `images/architecture/` - 系统架构相关图片

### 配置示例
- `configs/appsettings.example.json` - 应用配置完整示例
- `configs/docker-compose.example.yml` - Docker 编排示例

### 代码示例
- `samples/api-requests/` - REST API 请求示例集
- `samples/iec61850/` - IEC61850 协议模型文件

### 音频样本
- `samples/audio-samples/` - 声纹分析测试音频

## 相关文档

- [图表索引](../Diagrams/图表索引.md) - 所有架构图和流程图的位置
- [系统配置](../DeploymentGuide/系统配置.md) - 系统配置说明
- [API 文档](../ApiDocs/README.md) - API 使用示例
