# ARM32堆栈溢出问题

## 问题描述

在 ARM32 架构的边缘设备（如树莓派）上启用 Swagger UI 时，应用程序频繁出现堆栈溢出错误，导致服务崩溃。

### 症状

- 应用启动后短时间内崩溃
- 错误日志显示 "Stack overflow" 或 "Segmentation fault"
- 高并发请求时更容易触发
- 内存堆栈空间不足

### 影响范围

- **平台**: ARM32 设备（树莓派等）
- **组件**: Swagger UI (Swashbuckle.AspNetCore)
- **环境**: 低资源边缘计算设备

## 根本原因

ARM32 架构的栈空间限制比 x86/x64 更严格：

1. **Swagger UI 的反射开销**
   - Swagger 扫描所有 API 端点和类型
   - 大量反射操作消耗栈空间
   - 类型解析和文档生成深度递归

2. **ARM32 栈限制**
   - 默认栈大小: ~8MB (相比 x64 的 16MB+)
   - 每线程栈空间有限
   - 嵌套调用更容易超过栈深度

3. **ABP Framework 复杂性**
   - 大量自动生成的 API 端点
   - 泛型类型和模块化架构
   - 动态代理和拦截器链

## 解决方案

### 方案 1: 禁用 Swagger (推荐生产环境)

```json
{
  "App": {
    "EnableSwagger": false
  }
}
```

**适用场景**:
- ARM32 生产环境
- 低资源边缘设备
- 不需要 API 文档的场景

### 方案 2: 限制请求路径长度

在 `YiAbpWebModule.cs` 中配置：

```csharp
// 限制请求路径长度，减少栈消耗
context.Services.Configure<KestrelServerOptions>(options =>
{
    options.Limits.MaxRequestLineSize = 200; // 200 字符
});
```

### 方案 3: 使用开发文档分离

- **开发环境**: 保留 Swagger，在 x64 机器上测试
- **生产环境**: 禁用 Swagger，使用离线文档

## 实现代码

### YiAbpWebModule.cs 配置

```csharp
// Swagger
// ARM32 平台栈溢出防护：可通过配置禁用 Swagger（生产环境推荐）
var enableSwagger = configuration.GetValue<bool>("App:EnableSwagger", true);
if (enableSwagger)
{
    context.Services.AddYiSwaggerGen<YiAbpWebModule>(options =>
    {
        options.SwaggerDoc("default",
            new OpenApiInfo { Title = "Yi.Framework.Abp", Version = "v1", Description = "集大成者" });
    });
    logger.LogInformation("Swagger UI 已启用");
}
else
{
    logger.LogInformation("Swagger UI 已禁用（减少 ARM32 栈溢出风险）");
}
```

## 预防措施

### 开发规范

1. **环境区分**
   ```json
   // appsettings.LowResource.json
   {
     "App": {
       "EnableSwagger": false  // ARM32 平台建议设为 false
     }
   }
   
   // appsettings.HighPerformance.json
   {
     "App": {
       "EnableSwagger": true
     }
   }
   ```

2. **API 文档替代**
   - 使用离线 OpenAPI/Swagger JSON
   - 部署独立的文档服务器
   - 使用 Postman Collection 或 API Blueprint

3. **代码审查检查点**
   - ARM32 环境必须禁用 Swagger
   - 避免深度递归算法
   - 限制异步操作的嵌套深度

### 监控建议

- 添加栈使用监控
- 记录接近栈阈值的警告
- 定期进行压力测试

## 相关文件

- `/src/Yi.Abp.Web/YiAbpWebModule.cs` - Swagger 配置
- `/src/Yi.Abp.Web/appsettings.json` - 配置文件
- `/src/Yi.Abp.Web/appsettings.LowResource.json` - ARM32 专用配置

## 参考资料

- [ARM32 栈限制说明](https://www.raspberrypi.com/documentation/)
- [Swashbuckle.AspNetCore 性能优化](https://github.com/domaindrivendev/Swashbuckle.AspNetCore)
