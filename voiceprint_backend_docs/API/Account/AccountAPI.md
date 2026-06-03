# 账户模块 API 文档

## 概述
账户模块提供用户认证、登录、登出、密码管理等账户管理功能。

## 基础路径
```
/account
```

---

## 用户登录

### 用户登录

**接口**: `POST /my-login`

**描述**: 用户使用用户名和密码登录系统，支持验证码校验。

**请求参数**:
```json
{
  "userName": "admin",
  "password": "123456",
  "uuid": "guid-here",
  "code": "1234"
}
```

**请求参数说明**:

| 参数 | 类型 | 必填 | 说明 |
|-----|------|------|------|
| userName | string | 是 | 用户名 |
| password | string | 是 | 密码 |
| uuid | string | 否 | 验证码UUID（与图片验证码配合使用） |
| code | string | 否 | 验证码（系统配置需要时必填） |

**响应**:
```json
{
  "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "refreshToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "userName": "admin"
}
```

**响应数据结构**:

| 字段 | 类型 | 说明 |
|-----|------|------|
| token | string | JWT访问令牌 |
| refreshToken | string | 刷新令牌 |
| userName | string | 用户名 |

---

## 验证码

### 获取验证码图片

**接口**: `GET /my-captcha-image`

**描述**: 获取登录验证码图片，用于验证码校验。

**响应**:
```json
{
  "uuid": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
  "img": "base64-encoded-image-data",
  "isEnableCaptcha": true
}
```

**响应数据结构**:

| 字段 | 类型 | 说明 |
|-----|------|------|
| uuid | string | 验证码唯一标识（提交登录时需要携带） |
| img | byte[] | 验证码图片的Base64编码数据 |
| isEnableCaptcha | bool | 是否启用验证码校验 |

---

## 数据结构

### 登录请求 (LoginInputVo)

```typescript
interface LoginInputVo {
  userName: string;    // 用户名
  password: string;    // 密码
  uuid?: string;       // 验证码UUID
  code?: string;       // 验证码
}
```

### 登录响应 (LoginOutputDto)

```typescript
interface LoginOutputDto {
  token: string;          // JWT访问令牌
  refreshToken: string;   // 刷新令牌
  userName: string;        // 用户名
}
```

### 验证码图片 (CaptchaImageDto)

```typescript
interface CaptchaImageDto {
  uuid: string;              // 验证码UUID
  img: byte[];               // 图片数据（Base64）
  isEnableCaptcha: boolean;  // 是否启用验证码
}
```

---

## 认证流程

1. **获取验证码**（可选）
   - 调用 `GET /my-captcha-image` 获取验证码图片
   - 保存返回的 `uuid` 用于后续登录

2. **提交登录**
   - 调用 `POST /my-login` 提交用户名和密码
   - 如果系统启用了验证码，需要携带 `uuid` 和 `code`

3. **获取Token**
   - 登录成功后返回 `token` 和 `refreshToken`
   - 后续请求需要在Header中携带：`Authorization: Bearer {token}`

---

## 错误码

| 错误码 | 说明 |
|-------|------|
| 400 | 请求参数错误（用户名或密码为空） |
| 401 | 用户名或密码错误 |
| 403 | 验证码错误 |
| 423 | 用户已被禁用 |
| 500 | 服务器错误 |

---

## 前端使用示例

### 获取验证码

```typescript
const response = await fetch('/account/my-captcha-image');
const captchaData = await response.json();

// 显示验证码图片
const imgElement = document.getElementById('captcha-img');
imgElement.src = `data:image/png;base64,${captchaData.img}`;

// 保存UUID用于登录
const captchaUuid = captchaData.uuid;
```

### 用户登录

```typescript
const response = await fetch('/account/my-login', {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json'
  },
  body: JSON.stringify({
    userName: 'admin',
    password: '123456',
    uuid: captchaUuid,  // 如果启用了验证码
    code: '1234'        // 用户输入的验证码
  })
});

if (response.ok) {
  const loginData = await response.json();
  
  // 保存Token
  localStorage.setItem('token', loginData.token);
  localStorage.setItem('refreshToken', loginData.refreshToken);
  
  // 后续请求携带Token
  // headers: { 'Authorization': `Bearer ${loginData.token}` }
} else {
  console.error('登录失败');
}
```

### 完整登录流程示例

```typescript
// 1. 获取验证码
const getCaptcha = async () => {
  const response = await fetch('/account/my-captcha-image');
  return await response.json();
};

// 2. 登录
const login = async (userName: string, password: string, captchaCode?: string) => {
  const captcha = await getCaptcha();
  
  const response = await fetch('/account/my-login', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      userName,
      password,
      uuid: captcha.uuid,
      code: captchaCode
    })
  });
  
  if (response.ok) {
    const data = await response.json();
    localStorage.setItem('token', data.token);
    return data;
  } else {
    throw new Error('登录失败');
  }
};

// 使用
login('admin', '123456', '1234');
```

---

## 注意事项

1. **验证码配置**: 验证码功能可在系统配置中启用或禁用
2. **Token过期**: Token有过期时间，过期后需要使用refreshToken刷新或重新登录
3. **密码加密**: 前端建议对密码进行加密后再传输
4. **并发登录**: 同一用户多次登录会使之前的Token失效

---

## 相关文档

- [[认证系统]] - 认证系统设计文档
- [[JWT配置]] - JWT令牌配置说明
- [[用户管理API]] - 用户管理接口文档

---

## 前端使用位置

- `src/store/user.ts` - 用户状态管理
- `src/api/account.ts` - 账户API封装
- `src/views/login/Login.vue` - 登录页面
