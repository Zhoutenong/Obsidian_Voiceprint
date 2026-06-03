# Yi.Framework.AspNetCore

AspNetCore 模块提供 Web API 层的统一返回格式、路由约定、异常处理和 Swagger 配置。

## 核心组件

### 1. 统一返回结果

#### RESTfulResult

标准 API 响应格式：

```csharp
public class RESTfulResult<TResult>
{
    public int StatusCode { get; set; }
    public bool Success { get; set; }
    public TResult? Data { get; set; }
    public object? Errors { get; set; }
    public DateTime Timestamp { get; set; } = DateTime.Now;
}
```

**示例响应：**
```json
{
  "statusCode": 200,
  "success": true,
  "data": {
    "id": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
    "userName": "admin"
  },
  "errors": null,
  "timestamp": "2026-06-03T10:30:00Z"
}
```

**错误响应：**
```json
{
  "statusCode": 403,
  "success": false,
  "data": null,
  "errors": "权限不足",
  "timestamp": "2026-06-03T10:30:00Z"
}
```

### 2. 全局异常过滤器

#### FriendlyExceptionFilter

捕获所有未处理异常，转换为友好错误响应：

**位置：** `framework/Yi.Framework.AspNetCore/UnifyResult/Fiters/FriendlyExceptionFilter.cs`

```csharp
public class FriendlyExceptionFilter : IAsyncExceptionFilter
{
    public async Task OnExceptionAsync(ExceptionContext context)
    {
        // 排除 WebSocket 请求
        if (context.HttpContext.IsWebSocketRequest()) return;

        // 如果异常已处理，不再处理
        if (context.ExceptionHandled) return;

        // 解析异常信息
        var exceptionMetadata = GetExceptionMetadata(context);

        // 执行规范化异常处理
        IUnifyResultProvider unifyResult = context.GetRequiredService<IUnifyResultProvider>();
        context.Result = unifyResult.OnException(context, exceptionMetadata);

        // 记录日志
        var logger = context.HttpContext.RequestServices
            .GetRequiredService<ILogger<FriendlyExceptionFilter>>();
        logger.LogError(context.Exception, context.Exception.Message);
    }
}
```

**异常类型处理：**

| 异常类型 | HTTP 状态码 | 说明 |
|---------|-------------|------|
| `UserFriendlyException` | 403 | 用户友好异常（业务异常） |
| `AbpValidationException` | 400 | 参数验证异常 |
| `AbpAuthorizationException` | 401/403 | 授权异常 |
| `BusinessException` | 400 | 业务异常 |
| 其他异常 | 500 | 服务器内部错误 |

**GetExceptionMetadata 逻辑：**

```csharp
public static ExceptionMetadata GetExceptionMetadata(ActionContext context)
{
    var exception = /* ... */;
    var statusCode = StatusCodes.Status500InternalServerError;

    // 用户友好异常
    if (exception is UserFriendlyException friendlyException)
    {
        int.TryParse(friendlyException.Code, out statusCode);
        statusCode = statusCode == 0 ? 403 : statusCode;
        errors = friendlyException.Message;
        data = friendlyException.Data;
    }

    // 验证异常
    if (exception is AbpValidationException validationException)
    {
        statusCode = 400;
        errors = validationException.ValidationErrors
            .Select(x => x.ErrorMessage)
            .ToList();
    }

    return new ExceptionMetadata(statusCode, errors, data);
}
```

### 3. 成功响应过滤器

#### SucceededUnifyResultFilter

统一成功响应格式，包装返回数据：

```csharp
public class SucceededUnifyResultFilter : IActionFilter
{
    public void OnActionExecuted(ActionExecutedContext context)
    {
        if (context.Result is not ObjectResult objectResult) return;

        // 排除非统一返回的场景
        if (context.ActionDescriptor.EndpointMetadata
            .Any(x => x is NonUnifyAttribute)) return;

        // 获取响应数据
        var data = objectResult.Value;

        // 设置响应状态码
        OnResponseStatusCodes(context);

        // 包装为 RESTfulResult
        IUnifyResultProvider unifyResult = context.GetRequiredService<IUnifyResultProvider>();
        context.Result = unifyResult.OnSucceeded(context, data);
    }
}
```

### 4. 结果提供器

#### RESTfulResultProvider

实现 `IUnifyResultProvider` 接口，定义返回格式：

**位置：** `framework/Yi.Framework.AspNetCore/UnifyResult/Providers/RESTfulResultProvider.cs`

```csharp
public class RESTfulResultProvider : IUnifyResultProvider
{
    // 设置响应状态码
    public static void SetResponseStatusCodes(HttpContext context, int statusCode, UnifyResultSettingsOptions unifyResultSettings)
    {
        // 篡改响应状态码
        if (unifyResultSettings.AdaptStatusCodes != null)
        {
            var adaptStatusCode = unifyResultSettings.AdaptStatusCodes
                .FirstOrDefault(u => u[0] == statusCode);
            if (adaptStatusCode != null)
            {
                context.Response.StatusCode = adaptStatusCode[1];
                return;
            }
        }

        // 如果为 null，所有请求错误的状态码设置为 200
        if (unifyResultSettings.Return200StatusCodes == null)
            context.Response.StatusCode = 200;
        // 否则只有里面的才设置为 200
        else if (unifyResultSettings.Return200StatusCodes.Contains(statusCode))
            context.Response.StatusCode = 200;
    }

    // 异常返回值
    public IActionResult OnException(ExceptionContext context, ExceptionMetadata metadata)
    {
        return new JsonResult(RESTfulResult(
            metadata.StatusCode,
            data: metadata.Data,
            errors: metadata.Errors
        ));
    }

    // 成功返回值
    public IActionResult OnSucceeded(ActionExecutedContext context, object data)
    {
        return new JsonResult(RESTfulResult(StatusCodes.Status200OK, true, data));
    }

    // 验证失败返回值
    public IActionResult OnValidateFailed(ActionExecutedContext context, ValidationMetadata metadata)
    {
        return new JsonResult(RESTfulResult(
            metadata.StatusCode,
            data: null,
            errors: metadata.ValidationErrors
        ));
    }
}
```

### 5. 路由约定

#### YiConventionalRouteBuilder

ABP 风格的 RESTful 路由构建器：

**位置：** `framework/Yi.Framework.AspNetCore/Mvc/YiConventionalRouteBuilder.cs`

```csharp
public class YiConventionalRouteBuilder : ConventionalRouteBuilder
{
    public override string Build(
        string rootPath,
        string controllerName,
        ActionModel action,
        string httpMethod,
        ConventionalControllerSetting configuration)
    {
        // 构建路由前缀（API 版本 + 服务名）
        var apiRoutePrefix = GetApiRoutePrefix(action, configuration);
        var controllerNameInUrl = NormalizeUrlControllerName(rootPath, controllerName, action, httpMethod, configuration);

        var url = $"{rootPath}/{NormalizeControllerNameCase(controllerNameInUrl, configuration)}";

        // 添加 {id} 路径参数
        var idParameterModel = action.Parameters.FirstOrDefault(p => p.ParameterName == "id");
        if (idParameterModel != null)
        {
            if (TypeHelper.IsPrimitiveExtended(idParameterModel.ParameterType, includeEnums: true))
            {
                url += "/{id}";
            }
            else
            {
                // 复合对象展开为路径参数
                var properties = idParameterModel.ParameterType.GetProperties();
                foreach (var property in properties)
                {
                    url += "/{" + NormalizeIdPropertyNameCase(property, configuration) + "}";
                }
            }
        }

        // 添加 Action 名称
        var actionNameInUrl = NormalizeUrlActionName(rootPath, controllerName, action, httpMethod, configuration);
        if (!actionNameInUrl.IsNullOrEmpty())
        {
            url += $"/{NormalizeActionNameCase(actionNameInUrl, configuration)}";

            // 添加二级 Id（如 xxxId）
            var secondaryIds = action.Parameters
                .Where(p => p.ParameterName.EndsWith("Id", StringComparison.Ordinal))
                .ToList();
            if (secondaryIds.Count == 1)
            {
                url += $"/{{{NormalizeSecondaryIdNameCase(secondaryIds[0], configuration)}}}";
            }
        }

        return url;
    }
}
```

**路由格式：**
```
/api/app/{service-name}/{controller}/{id}/{action}/{secondaryId}
```

**示例：**
```csharp
[Route("api/app/[controller]")]
public class UserController : AppService
{
    [HttpGet]
    public async Task<UserDto> GetAsync(Guid id) // GET /api/app/rbac/user/{id}

    [HttpPost]
    public async Task<UserDto> CreateAsync(CreateUserDto input) // POST /api/app/rbac/user

    [HttpPut("{id}")]
    public async Task<UserDto> UpdateAsync(Guid id, UpdateUserDto input) // PUT /api/app/rbac/user/{id}

    [HttpDelete("{id}")]
    public async Task DeleteAsync(Guid id) // DELETE /api/app/rbac/user/{id}

    [HttpGet("{id}/roles")]
    public async Task<List<RoleDto>> GetRolesAsync(Guid id) // GET /api/app/rbac/user/{id}/roles
}
```

#### YiServiceConvention

服务约定，支持自定义路由配置：

```csharp
public class YiServiceConvention : AbpServiceConvention
{
    protected override void ConfigureSelector(
        string rootPath,
        string controllerName,
        ActionModel action,
        ConventionalControllerSetting? configuration)
    {
        // 移除空选择器
        RemoveEmptySelectors(action.Selectors);

        // 检查 RemoteServiceAttribute
        var remoteServiceAtt = ReflectionHelper.GetSingleAttributeOrDefault<RemoteServiceAttribute>(action.ActionMethod);
        if (remoteServiceAtt != null && !remoteServiceAtt.IsEnabledFor(action.ActionMethod))
        {
            return;
        }

        if (!action.Selectors.Any())
        {
            AddAbpServiceSelector(rootPath, controllerName, action, configuration);
        }
        else
        {
            NormalizeSelectorRoutes(rootPath, controllerName, action, configuration);
        }
    }
}
```

### 6. 配置选项

#### UnifyResultSettingsOptions

统一返回设置选项：

```csharp
public class UnifyResultSettingsOptions
{
    /// 返回 200 状态码的 HTTP 状态码列表
    public int[]? Return200StatusCodes { get; set; }

    /// 支持 MVC 控制器（非 ABP ApplicationService）
    public bool? SupportMvcController { get; set; }

    /// 状态码映射（原状态码 -> 新状态码）
    public int[][]? AdaptStatusCodes { get; set; }
}
```

**PostConfigure 默认值：**
```csharp
public void PostConfigure(UnifyResultSettingsOptions options, IConfiguration configuration)
{
    options.Return200StatusCodes ??= new[] { 401, 403 };
    options.SupportMvcController ??= false;
}
```

**配置示例：**
```json
{
  "UnifyResultSettings": {
    "Return200StatusCodes": [400, 401, 403, 404, 500],
    "SupportMvcController": false,
    "AdaptStatusCodes": [
      [403, 200], // 将 403 映射为 200
      [404, 200]  // 将 404 映射为 200
    ]
  }
}
```

### 7. 非 Unify 返回

#### NonUnifyAttribute

标记不需要统一返回的方法：

```csharp
public class FileController : AppService
{
    [NonUnify]
    [HttpGet("download")]
    public IActionResult DownloadFile()
    {
        // 直接返回文件流，不包装为 RESTfulResult
        var fileBytes = System.IO.File.ReadAllBytes("path/to/file.pdf");
        return File(fileBytes, "application/pdf", "file.pdf");
    }

    [NonUnify]
    [HttpGet("health")]
    public string HealthCheck()
    {
        // 直接返回字符串
        return "OK";
    }
}
```

### 8. Swagger 配置

#### ApiInfoBuilderExtensions

自定义 Swagger UI 配置：

```csharp
public static class ApiInfoBuilderExtensions
{
    public static IServiceCollection AddYiSwagger(this IServiceCollection services)
    {
        services.AddSwaggerGen(options =>
        {
            options.SwaggerDoc("v1", new OpenApiInfo
            {
                Title = "IntelliSubstation Voiceprint API",
                Version = "v1",
                Description = "智能变电站声纹监控系统 API"
            });

            // 添加 JWT 认证
            options.AddSecurityDefinition("Bearer", new OpenApiSecurityScheme
            {
                Description = "JWT Authorization header. Example: \"Authorization: Bearer {token}\"",
                Name = "Authorization",
                In = ParameterLocation.Header,
                Type = SecuritySchemeType.ApiKey,
                Scheme = "Bearer"
            });

            options.AddSecurityRequirement(new OpenApiSecurityRequirement
            {
                {
                    new OpenApiSecurityScheme
                    {
                        Reference = new OpenApiReference { Type = ReferenceType.SecurityScheme, Id = "Bearer" }
                    },
                    Array.Empty<string>()
                }
            });

            // 包含 XML 注释
            var xmlFile = $"{Assembly.GetExecutingAssembly().GetName().Name}.xml";
            var xmlPath = Path.Combine(AppContext.BaseDirectory, xmlFile);
            if (File.Exists(xmlPath))
            {
                options.IncludeXmlComments(xmlPath);
            }
        });

        return services;
    }
}
```

#### SwaggerBuilderExtensions

中间件配置：

```csharp
public static IApplicationBuilder UseYiSwagger(this IApplicationBuilder app)
{
    app.UseSwagger();
    app.UseSwaggerUI(options =>
    {
        options.SwaggerEndpoint("/swagger/v1/swagger.json", "IntelliSubstation Voiceprint API v1");
        options.RoutePrefix = "swagger";
    });

    return app;
}
```

## 使用示例

### 应用服务

```csharp
public class UserAppService : ApplicationService
{
    private readonly IUserRepository _userRepository;

    // GET /api/app/rbac/user/{id}
    public async Task<UserDto> GetAsync(Guid id)
    {
        var user = await _userRepository.GetAsync(id);
        return ObjectMapper.Map<UserDto>(user);
    }

    // POST /api/app/rbac/user
    public async Task<UserDto> CreateAsync(CreateUserDto input)
    {
        var user = ObjectMapper.Map<UserAggregateRoot>(input);
        await _userRepository.InsertAsync(user);
        return ObjectMapper.Map<UserDto>(user);
    }

    // PUT /api/app/rbac/user/{id}
    public async Task<UserDto> UpdateAsync(Guid id, UpdateUserDto input)
    {
        var user = await _userRepository.GetAsync(id);
        user.UserName = input.UserName;
        await _userRepository.UpdateAsync(user);
        return ObjectMapper.Map<UserDto>(user);
    }

    // DELETE /api/app/rbac/user/{id}
    public async Task DeleteAsync(Guid id)
    {
        await _userRepository.DeleteAsync(id);
    }

    // GET /api/app/rbac/user/{id}/roles
    public async Task<List<RoleDto>> GetRolesAsync(Guid id)
    {
        var user = await _userRepository.GetAsync(id);
        return ObjectMapper.Map<List<RoleDto>>(user.Roles);
    }
}
```

### 自定义响应格式

**实现 IUnifyResultProvider：**

```csharp
public class CustomResultProvider : IUnifyResultProvider, ITransientDependency
{
    public IActionResult OnException(ExceptionContext context, ExceptionMetadata metadata)
    {
        return new JsonResult(new
        {
            code = metadata.StatusCode,
            message = metadata.Errors,
            timestamp = DateTime.Now
        });
    }

    public IActionResult OnSucceeded(ActionExecutedContext context, object data)
    {
        return new JsonResult(new
        {
            code = 200,
            result = data,
            timestamp = DateTime.Now
        });
    }
}
```

**注册：**
```csharp
[Dependency(TryRegister = true)]
[ExposeServices(typeof(IUnifyResultProvider))]
public class CustomResultProvider : IUnifyResultProvider, ITransientDependency
{
    // ...
}
```

### 自定义异常

```csharp
public class UserAppService : ApplicationService
{
    public async Task<UserDto> CreateAsync(CreateUserDto input)
    {
        // 检查用户名是否存在
        if (await _userRepository.IsNameExistAsync(input.UserName))
        {
            // 抛出友好异常（返回 403）
            throw new UserFriendlyException("用户名已存在");
        }

        // 业务异常（返回 400）
        if (input.Age < 18)
        {
            throw new BusinessException("年龄必须大于 18 岁");
        }

        var user = ObjectMapper.Map<UserAggregateRoot>(input);
        await _userRepository.InsertAsync(user);
        return ObjectMapper.Map<UserDto>(user);
    }
}
```

### 参数验证

```csharp
public class CreateUserDto
{
    [Required(ErrorMessage = "用户名必填")]
    [StringLength(50, ErrorMessage = "用户名长度不能超过 50")]
    public string UserName { get; set; }

    [Required(ErrorMessage = "密码必填")]
    [StringLength(100, MinimumLength = 6, ErrorMessage = "密码长度必须在 6-100 之间")]
    public string Password { get; set; }

    [EmailAddress(ErrorMessage = "邮箱格式不正确")]
    public string? Email { get; set; }
}
```

**验证失败响应：**
```json
{
  "statusCode": 400,
  "success": false,
  "data": null,
  "errors": [
    "用户名必填",
    "密码长度必须在 6-100 之间"
  ],
  "timestamp": "2026-06-03T10:30:00Z"
}
```

## 模块配置

### YiFrameworkAspNetCoreModule

```csharp
[DependsOn(typeof(YiFrameworkCoreModule))]
public class YiFrameworkAspNetCoreModule : AbpModule
{
    public override void PostConfigureServices(ServiceConfigurationContext context)
    {
        var services = context.Services;

        // 替换 Web 客户端信息提供器
        services.Replace(new ServiceDescriptor(
            typeof(IWebClientInfoProvider),
            typeof(RealIpHttpContextWebClientInfoProvider),
            ServiceLifetime.Transient
        ));
    }
}
```

### 配置 Swagger

```csharp
public override void OnApplicationInitialization(ApplicationInitializationContext context)
{
    var app = context.GetApplicationBuilder();
    var configuration = context.ServiceProvider.GetConfiguration();

    // 启用 Swagger
    if (configuration["App:EnableSwagger"] == "true")
    {
        app.UseYiSwagger();
    }
}
```

## 最佳实践

1. **统一返回**
   - 使用 `RESTfulResult` 作为标准响应格式
   - 避免直接返回非包装数据
   - 使用 `[NonUnify]` 标记特殊场景

2. **异常处理**
   - 抛出 `UserFriendlyException` 实现友好异常
   - 使用 `BusinessException` 标记业务异常
   - 避免直接抛出未处理异常

3. **路由约定**
   - 遵循 RESTful 设计原则
   - 使用名词复数表示资源（如 `/users`）
   - 使用 HTTP 方法表示操作（GET/POST/PUT/DELETE）

4. **参数验证**
   - 使用 Data Annotations 进行验证
   - 返回清晰的验证错误信息
   - 避免在服务层重复验证

5. **Swagger 文档**
   - 添加 XML 注释生成 API 文档
   - 使用 `[Obsolete]` 标记废弃 API
   - 为复杂类型添加 Schema 示例

## 相关文档

- [Framework 概览](./README.md)
- [SqlSugarCore](./SqlSugarCore.md)
- [BackgroundWorkers.Hangfire](./BackgroundWorkersHangfire.md)
