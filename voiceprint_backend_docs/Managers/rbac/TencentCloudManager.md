# TencentCloudManager — 腾讯云服务管理器

## 基本信息

- **Manager名称**：`TencentCloudManager`
- **模块位置**：`module/rbac/Yi.Framework.Rbac.Domain/Managers/`
- **继承关系**：`DomainService`
- **生命周期**：依赖注入（瞬态）
- **命名空间**：`Yi.Framework.Rbac.Domain.Managers`

## Manager概述

TencentCloudManager 是腾讯云服务管理器，负责集成腾讯云API服务。当前实现主要支持短信发送功能，使用腾讯云SMS服务发送通知消息。

## 核心职责

1. **短信发送** - 通过腾讯云SMS API发送短信
2. **API集成** - 封装腾讯云SDK调用
3. **错误处理** - 记录API调用日志和异常

## 依赖注入

| 依赖 | 职责 |
|------|------|
| `ILogger<TencentCloudManager>` | 日志记录器 |

## 核心方法

### SendSmsAsync - 发送短信

**签名**：
```csharp
public async Task SendSmsAsync()
```

**功能**：
通过腾讯云SMS服务发送短信消息。

**执行流程**：

```
1. 创建认证凭据
   │
   ├─ Credential cred = new Credential()
   │  ├─ SecretId = "SecretId"  （腾讯云密钥ID）
   │  └─ SecretKey = "SecretKey"（腾讯云密钥）
   │
2. 配置HTTP选项
   │
   ├─ HttpProfile httpProfile = new HttpProfile()
   │  └─ Endpoint = "sms.tencentcloudapi.com"
   │
3. 创建SMS客户端
   │
   └─ SmsClient client = new SmsClient(cred, "", clientProfile)
   │
4. 创建请求对象
   │
   └─ SendSmsRequest req = new SendSmsRequest()
   │
5. 发送短信
   │
   └─ SendSmsResponse resp = await client.SendSms(req)
   │
6. 记录响应
   │
   └─ Logger.LogInformation("腾讯云Sms返回：" + ToJsonString(resp))
   │
7. 异常处理
   │
   └─ catch (Exception e)
      └─ Logger.LogError(e, e.ToString())
```

## 腾讯云SMS集成

### 认证凭据

```csharp
Credential cred = new Credential
{
    SecretId = "SecretId",   // 腾讯云SecretId
    SecretKey = "SecretKey"  // 腾讯云SecretKey
};
```

**获取密钥**：
- 访问腾讯云控制台：https://console.cloud.tencent.com/cam/capi
- 创建或查看SecretId和SecretKey

### 安全注意事项

⚠️ **重要**：代码中的密钥是示例值，实际使用时应该：
1. 使用配置文件存储密钥
2. 不要将密钥硬编码在代码中
3. 使用环境变量或密钥管理服务
4. 定期轮换密钥
5. 参考腾讯云安全指南：https://cloud.tencent.com/document/product/1278/85305

### HTTP配置

```csharp
HttpProfile httpProfile = new HttpProfile();
httpProfile.Endpoint = "sms.tencentcloudapi.com";  // SMS服务端点
```

**SMS端点**：
- 国内短信：`sms.tencentcloudapi.com`
- 国际短信：`sms.intl.tencentcloudapi.com`

## SDK依赖

### NuGet包

```xml
<!-- 腾讯云Common -->
<PackageReference Include="TencentCloudCommon" Version="*" />

<!-- 腾讯云SMS -->
<PackageReference Include="TencentCloudSMS" Version="*" />
```

### 命名空间

```csharp
using TencentCloud.Common.Profile;
using TencentCloud.Common;
using TencentCloud.Sms.V20210111.Models;
using TencentCloud.Sms.V20210111;
```

## 短信发送请求

### SendSmsRequest 参数

```csharp
SendSmsRequest req = new SendSmsRequest();

// 常用参数（需要根据实际需求设置）
req.PhoneNumberSet = new[] { "+8613800138000" };  // 手机号列表
req.TemplateId = "123456";                        // 短信模板ID
req.TemplateParamSet = new[] { "参数1", "参数2" }; // 模板参数
req.SmsSdkAppId = "1400123456";                    // 短信应用ID
req.SignName = "XXX公司";                          // 签名名称
req.SessionContext = "自定义会话ID";               // 会话上下文
req.ExtendCode = "001";                            // 扩展码
req.SenderId = "SenderId";                         // 国内短信 senderid
```

### 短信响应

```csharp
SendSmsResponse resp = await client.SendSms(req);

// 响应字段
resp.RequestId;        // 请求ID
resp.SendStatusSet;    // 发送状态列表
```

## 日志记录

### 成功日志

```csharp
_logger.LogInformation("腾讯云Sms返回：" + AbstractModel.ToJsonString(resp));
```

### 错误日志

```csharp
catch (Exception e)
{
    _logger.LogError(e, e.ToString());
}
```

## 使用场景

### 1. 发送验证码短信

```csharp
await _tencentCloudManager.SendSmsAsync();
```

### 2. 发送通知短信

```csharp
// 告警通知
await _tencentCloudManager.SendSmsAsync();
```

### 3. 发送营销短信

```csharp
// 营销活动通知
await _tencentCloudManager.SendSmsAsync();
```

## 配置示例

### appsettings.json

```json
{
  "TencentCloud": {
    "SecretId": "your-secret-id",
    "SecretKey": "your-secret-key",
    "SmsSdkAppId": "1400123456",
    "SignName": "XXX公司",
    "DefaultTemplateId": "123456"
  }
}
```

### 配置类

```csharp
public class TencentCloudOptions
{
    public string SecretId { get; set; }
    public string SecretKey { get; set; }
    public string SmsSdkAppId { get; set; }
    public string SignName { get; set; }
    public string DefaultTemplateId { get; set; }
}
```

## 设计特点

1. **简单封装** - 对腾讯云SDK进行简单封装
2. **异常安全** - 捕获并记录所有异常
3. **日志完整** - 记录请求响应和错误信息
4. **灵活配置** - 支持自定义HTTP端点

## 限制与改进

### 当前限制

1. **参数硬编码** - SecretId和SecretKey硬编码在代码中
2. **方法签名** - SendSmsAsync无参数，无法动态配置短信内容
3. **返回值** - 方法无返回值，调用者无法知道发送结果
4. **功能单一** - 仅支持SMS，未实现其他腾讯云服务

### 改进建议

#### 1. 配置外部化

```csharp
public class TencentCloudManager : DomainService
{
    private readonly TencentCloudOptions _options;
    
    public TencentCloudManager(
        IOptions<TencentCloudOptions> options,
        ILogger<TencentCloudManager> logger)
    {
        _options = options.Value;
        _logger = logger;
    }
    
    public async Task<bool> SendSmsAsync(
        string phoneNumber,
        string templateId,
        string[] templateParams)
    {
        try
        {
            Credential cred = new Credential
            {
                SecretId = _options.SecretId,
                SecretKey = _options.SecretKey
            };
            // ...
        }
        catch (Exception e)
        {
            _logger.LogError(e, e.ToString());
            return false;
        }
    }
}
```

#### 2. 增加返回值

```csharp
public async Task<SmsResult> SendSmsAsync(
    string[] phoneNumbers,
    string templateId,
    string[] templateParams)
{
    try
    {
        var resp = await client.SendSms(req);
        return new SmsResult
        {
            Success = true,
            RequestId = resp.RequestId,
            SendStatusSet = resp.SendStatusSet
        };
    }
    catch (Exception e)
    {
        _logger.LogError(e, e.ToString());
        return new SmsResult { Success = false, Error = e.Message };
    }
}
```

#### 3. 支持多服务

```csharp
public class TencentCloudManager : DomainService
{
    // SMS服务
    public async Task SendSmsAsync(...) { }
    
    // COS服务
    public async Task UploadFileAsync(...) { }
    
    // CDN服务
    public async Task RefreshCacheAsync(...) { }
}
```

## 腾讯云服务

### 支持的服务

| 服务 | 用途 | 状态 |
|------|------|------|
| SMS | 短信发送 | ✅ 已实现 |
| COS | 对象存储 | ⬜ 待实现 |
| CDN | 内容分发 | ⬜ 待实现 |
| CI | 内容智能 | ⬜ 待实现 |
| VOD | 视点处理 | ⬜ 待实现 |

## 相关服务

- `NoticeService` - 通知服务（可能调用短信发送）
- `AlertService` - 告警服务（可能发送短信通知）

## 费用说明

腾讯云SMS服务采用按量计费：
- 国内短信：按条计费
- 国际短信：按国家和运营商计费
- 详情参考：https://cloud.tencent.com/document/product/382/17274

## 注意事项

1. **密钥安全** - 不要将密钥提交到代码仓库
2. **签名审核** - 短信签名需要通过腾讯云审核
3. **模板审核** - 短信模板需要通过腾讯云审核
4. **频率限制** - 短信发送有频率限制
5. **费用控制** - 设置费用预警避免意外高额费用
6. **错误处理** - 妥善处理发送失败的情况

## 扩展示例

### 发送验证码

```csharp
public async Task<bool> SendVerificationCodeAsync(
    string phoneNumber,
    string code)
{
    var req = new SendSmsRequest
    {
        PhoneNumberSet = new[] { phoneNumber },
        TemplateId = _options.VerificationTemplateId,
        TemplateParamSet = new[] { code },
        SmsSdkAppId = _options.SmsSdkAppId,
        SignName = _options.SignName
    };
    
    try
    {
        var resp = await client.SendSms(req);
        _logger.LogInformation($"验证码短信已发送：{phoneNumber}");
        return true;
    }
    catch (Exception e)
    {
        _logger.LogError(e, $"验证码短信发送失败：{e.Message}");
        return false;
    }
}
```

---

> **最后更新**：2026-06-04  
> **源码位置**：`module/rbac/Yi.Framework.Rbac.Domain/Managers/TencentCloudManager.cs`  
> **腾讯云文档**：https://cloud.tencent.com/document/product/382
