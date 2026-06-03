# Yi.Framework.AspNetCore.Authentication.OAuth

## 概述

`Yi.Framework.AspNetCore.Authentication.OAuth` 是基于 ASP.NET Core Authentication 的第三方 OAuth 登录框架，支持 QQ 和 Gitee 两种登录方式。该模块遵循标准的 OAuth 2.0 授权码流程，提供安全的第三方身份认证集成。

**支持的登录提供商：**
- 🔵 **QQ 互联** - Tencent QQ 开放平台
- 🟠 **Gitee** - OSChina 开源社区

**核心特性：**
- ✅ 标准 OAuth 2.0 授权码流程
- ✅ 自定义 Claim 映射
- ✅ 统一的错误处理机制
- ✅ 灵活的配置选项

## 架构设计

### 模块结构

```
Yi.Framework.AspNetCore.Authentication.OAuth/
├── OAuthAuthenticationHandler.cs        # 基础 OAuth 处理器
├── AuthenticationOAuthOptions.cs       # 基础配置选项
├── AuthenticationConstants.cs           # 常量定义
├── AuthticationErrCodeModel.cs         # 错误码模型
├── QQ/                                  # QQ 登录实现
│   ├── QQAuthenticationHandler.cs
│   ├── QQAuthenticationOptions.cs
│   ├── QQAuthenticationDefaults.cs
│   ├── QQAuthenticationConstants.cs
│   └── QQAuthticationcationHttpModel.cs
└── Gitee/                               # Gitee 登录实现
    ├── GiteeAuthenticationHandler.cs
    ├── GiteeAuthenticationOptions.cs
    ├── GiteeAuthenticationDefaults.cs
    ├── GiteeAuthenticationConstants.cs
    └── GiteeAuthticationcationHttpModel.cs
```

### 核心组件

**1. OAuthAuthenticationHandler<TOptions>**

抽象基类，实现 OAuth 2.0 授权码流程的核心逻辑：

```csharp
public abstract class OauthAuthenticationHandler<TOptions> : AuthenticationHandler<TOptions> 
    where TOptions : AuthenticationSchemeOptions, new()
{
    public abstract string AuthenticationSchemeNmae { get; }
    protected IHttpClientFactory HttpClientFactory { get; }
    protected HttpClient HttpClient { get; }

    protected abstract Task<List<Claim>> GetAuthTicketAsync(string code);
    
    protected override async Task<AuthenticateResult> HandleAuthenticateAsync()
    {
        // 1. 验证 code 参数
        // 2. 调用 GetAuthTicketAsync 获取用户信息
        // 3. 生成 AuthenticationTicket
    }
}
```

**2. 具体实现类**

```csharp
// QQ 登录处理器
public class QQAuthenticationHandler : OauthAuthenticationHandler<QQAuthenticationOptions>

// Gitee 登录处理器  
public class GiteeAuthenticationHandler : OauthAuthenticationHandler<GiteeAuthenticationOptions>
```

## OAuth 2.0 流程

### 完整授权流程

```
┌─────────┐                    ┌──────────────┐                  ┌─────────┐
│  用户   │                    │   应用系统   │                  │ QQ/Gitee│
└────┬────┘                    └──────┬───────┘                  └────┬────┘
     │                                │                               │
     │ 1. 点击登录                     │                               │
     ├───────────────────────────────>│                               │
     │                                │ 2. 重定向到授权页面              │
     │                                ├──────────────────────────────>│
     │                                │                               │
     │                        3. 用户授权                            │
     │                                │<──────────────────────────────│
     │                                │                               │
     │                        4. 回调带 code                         │
     │                                │                               │
     │ 5. 回调到应用                   │                               │
     │<───────────────────────────────│                               │
     │                                │                               │
     │                                │ 6. 用 code 换 access_token     │
     │                                ├──────────────────────────────>│
     │                                │ 7. 返回 access_token           │
     │                                │<──────────────────────────────│
     │                                │                               │
     │                                │ 8. 获取用户信息                 │
     │                                ├──────────────────────────────>│
     │                                │ 9. 返回用户信息                 │
     │                                │<──────────────────────────────│
     │                                │                               │
     │                        10. 生成本地 Session                     │
     │<───────────────────────────────│                               │
```

## QQ 登录配置

### QQAuthenticationOptions

```csharp
public class QQAuthenticationOptions : AuthenticationOAuthOptions
{
    public QQAuthenticationOptions()
    {
        ClaimsIssuer = QQAuthenticationDefaults.Issuer;
        CallbackPath = QQAuthenticationDefaults.CallbackPath;

        // OAuth 端点配置
        AuthorizationEndpoint = QQAuthenticationDefaults.AuthorizationEndpoint;
        TokenEndpoint = QQAuthenticationDefaults.TokenEndpoint;
        UserInformationEndpoint = QQAuthenticationDefaults.UserInformationEndpoint;

        // 请求权限
        Scope.Add("get_user_info");

        // Claim 映射
        ClaimActions.MapJsonKey(ClaimTypes.Name, "nickname");
        ClaimActions.MapJsonKey(ClaimTypes.Gender, "gender");
        ClaimActions.MapJsonKey(Claims.PictureUrl, "figureurl");
        // ... 更多映射
    }

    public bool ApplyForUnionId { get; set; }
    public string UserIdentificationEndpoint { get; set; }
}
```

### QQ 授权流程

**1. 授权端点：**
```
https://graph.qq.com/oauth2.0/authorize
```

**2. 获取 Access Token：**
```csharp
var tokenQueryKv = new List<KeyValuePair<string, string?>>
{
    new("grant_type", "authorization_code"),
    new("client_id", Options.ClientId),
    new("client_secret", Options.ClientSecret),
    new("redirect_uri", Options.RedirectUri),
    new("code", code)
};
```

**3. 获取用户信息：**
```csharp
var userInfoQueryKv = new List<KeyValuePair<string, string?>>
{
    new("access_token", tokenModel.access_token),
    new("oauth_consumer_key", Options.ClientId),
    new("openid", tokenModel.openid),
};
```

### QQ Claim 返回

```csharp
List<Claim> claims = new()
{
    new Claim(Claims.AvatarFullUrl, userInfoMdoel.figureurl_qq_2),
    new Claim(Claims.AvatarUrl, userInfoMdoel.figureurl_qq_1),
    new Claim(AuthenticationConstants.OpenId, tokenModel.openid),
    new Claim(AuthenticationConstants.Name, userInfoMdoel.nickname),
    new Claim(AuthenticationConstants.AccessToken, tokenModel.access_token),
};
```

## Gitee 登录配置

### GiteeAuthenticationOptions

```csharp
public class GiteeAuthenticationOptions : AuthenticationOAuthOptions
{
    public GiteeAuthenticationOptions()
    {
        ClaimsIssuer = GiteeAuthenticationDefaults.Issuer;
        CallbackPath = GiteeAuthenticationDefaults.CallbackPath;

        AuthorizationEndpoint = GiteeAuthenticationDefaults.AuthorizationEndpoint;
        TokenEndpoint = GiteeAuthenticationDefaults.TokenEndpoint;
        UserInformationEndpoint = GiteeAuthenticationDefaults.UserInformationEndpoint;

        Scope.Add("user_info");
        Scope.Add("emails");

        ClaimActions.MapJsonKey(ClaimTypes.NameIdentifier, "id");
        ClaimActions.MapJsonKey(ClaimTypes.Name, "login");
        ClaimActions.MapJsonKey(ClaimTypes.Email, "email");
    }

    public string UserEmailsEndpoint { get; set; }
}
```

### Gitee 授权流程

**1. 授权端点：**
```
https://gitee.com/oauth/authorize
```

**2. 获取 Access Token（POST）：**
```csharp
var tokenQueryKv = new List<KeyValuePair<string, string?>>
{
    new("grant_type", "authorization_code"),
    new("client_id", Options.ClientId),
    new("client_secret", Options.ClientSecret),
    new("redirect_uri", Options.RedirectUri),
    new("code", code)
};
```

**3. 获取用户信息：**
```csharp
var userInfoQueryKv = new List<KeyValuePair<string, string?>>
{
    new("access_token", tokenModel.access_token),
};
```

### Gitee Claim 返回

```csharp
List<Claim> claims = new()
{
    new Claim(Claims.AvatarUrl, userInfoMdoel.avatar_url),
    new Claim(Claims.Url, userInfoMdoel.url),
    new Claim(AuthenticationConstants.OpenId, userInfoMdoel.id.ToString()),
    new Claim(AuthenticationConstants.Name, userInfoMdoel.name),
    new Claim(AuthenticationConstants.AccessToken, tokenModel.access_token)
};
```

## 配置使用

### Startup.cs 配置

```csharp
public void ConfigureServices(IServiceCollection services)
{
    // QQ 登录配置
    services.AddAuthentication()
        .AddQQ<QQAuthenticationOptions>(QQAuthenticationDefaults.AuthenticationScheme, options =>
        {
            options.ClientId = "your_qq_app_id";
            options.ClientSecret = "your_qq_app_key";
            options.RedirectUri = "https://yourdomain.com/signin-qq";
        });

    // Gitee 登录配置
    services.AddAuthentication()
        .AddGitee<GiteeAuthenticationOptions>(GiteeAuthenticationDefaults.AuthenticationScheme, options =>
        {
            options.ClientId = "your_gitee_client_id";
            options.ClientSecret = "your_gitee_client_secret";
            options.RedirectUri = "https://yourdomain.com/signin-gitee";
        });
}
```

### appsettings.json 配置

```json
{
  "Authentication": {
    "QQ": {
      "ClientId": "your_qq_app_id",
      "ClientSecret": "your_qq_app_key",
      "RedirectUri": "https://yourdomain.com/signin-qq"
    },
    "Gitee": {
      "ClientId": "your_gitee_client_id",
      "ClientSecret": "your_gitee_client_secret",
      "RedirectUri": "https://yourdomain.com/signin-gitee"
    }
  }
}
```

## 错误处理

### 统一错误处理

模块提供统一的错误响应验证：

```csharp
protected virtual void VerifyErrResponse(string content)
{
    AuthticationErrCodeModel.VerifyErrResponse(content);
}
```

### HTTP 请求错误

```csharp
if (!response.IsSuccessStatusCode)
{
    throw new Exception($"授权服务器请求错误,请求地址:{queryUrl},错误信息：{content}");
}
```

### 认证失败

```csharp
if (!Context.Request.Query.ContainsKey("code"))
{
    return AuthenticateResult.Fail("回调未包含code参数");
}
```

## 应用申请

### QQ 互联申请

1. 访问 [QQ 互联平台](https://connect.qq.com/)
2. 创建应用，获取 `App ID` 和 `App Key`
3. 配置回调域名：`https://yourdomain.com/signin-qq`
4. 等待审核通过

### Gitee 申请

1. 访问 [Gitee 开放平台](https://gitee.com/oauth/applications)
2. 创建第三方应用
3. 获取 `Client ID` 和 `Client Secret`
4. 配置回调地址：`https://yourdomain.com/signin-gitee`

## 安全建议

1. **HTTPS 强制**: 生产环境必须使用 HTTPS
2. **State 参数**: 建议实现 CSRF 防护
3. **Token 存储**: Access Token 应加密存储
4. **Scope 最小化**: 仅请求必要的权限范围
5. **定期轮换**: 定期更新 Client Secret

## 参考资源

- [QQ 互联文档](https://wiki.connect.qq.com/)
- [Gitee OAuth 文档](https://gitee.com/api/v5/oauth_doc)
- [ASP.NET Core Authentication](https://docs.microsoft.com/aspnet/core/security/authentication/)
