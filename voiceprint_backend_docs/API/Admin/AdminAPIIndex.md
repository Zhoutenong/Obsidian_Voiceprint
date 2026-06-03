# RBAC 管理端 API 汇总

## 概述

RBAC（基于角色的访问控制）模块提供完整的用户权限管理功能，包括用户管理、角色管理、菜单管理、部门管理、岗位管理、系统配置、日志管理和监控等功能。

## 基础路径

所有管理端 API 都在 `/api/app` 路径下，按服务名组织：

```
/api/app/{service-name}/{controller}/{action}
```

---

## 服务模块总览

| 服务名 | 功能描述 | 主要控制器 |
|-------|---------|-----------|
| rbac | 核心权限管理 | UserService, RoleService, MenuService |
| system | 系统基础管理 | DeptService, PostService, DictionaryService |
| config | 系统配置 | ConfigService, NoticeService |
| log | 日志管理 | LoginLogService, OperationLogService |
| monitor | 系统监控 | OnlineService, MonitorServerService, MonitorCacheService |
| account | 账户认证 | AccountService, AuthService |
| file | 文件管理 | FileService |

---

## API 端点清单

### 用户管理 (UserService)

| 方法 | 路径 | 描述 | 权限码 |
|-----|------|------|--------|
| GET | `/api/app/rbac/user` | 获取用户列表（分页） | `system:user:list` |
| GET | `/api/app/rbac/user/{id}` | 获取用户详情 | `system:user:query` |
| POST | `/api/app/rbac/user` | 创建用户 | `system:user:add` |
| PUT | `/api/app/rbac/user/{id}` | 更新用户 | `system:user:edit` |
| DELETE | `/api/app/rbac/user/{id}` | 删除用户 | `system:user:remove` |
| DELETE | `/api/app/rbac/user` | 批量删除用户 | `system:user:remove` |
| PUT | `/api/app/rbac/user/{id}/{state}` | 更新用户状态 | `system:user:edit` |
| POST | `/api/app/rbac/user/{id}/role` | 为用户分配角色 | `system:user:edit` |
| POST | `/api/app/rbac/user/{id}/post` | 为用户分配岗位 | `system:user:edit` |
| POST | `/api/app/rbac/user/{id}/dept` | 为用户分配部门 | `system:user:edit` |

### 角色管理 (RoleService)

| 方法 | 路径 | 描述 | 权限码 |
|-----|------|------|--------|
| GET | `/api/app/rbac/role` | 获取角色列表（分页） | `system:role:list` |
| GET | `/api/app/rbac/role/{id}` | 获取角色详情 | `system:role:query` |
| POST | `/api/app/rbac/role` | 创建角色 | `system:role:add` |
| PUT | `/api/app/rbac/role/{id}` | 更新角色 | `system:role:edit` |
| DELETE | `/api/app/rbac/role/{id}` | 删除角色 | `system:role:remove` |
| DELETE | `/api/app/rbac/role` | 批量删除角色 | `system:role:remove` |
| PUT | `/api/app/rbac/role/{id}/{state}` | 更新角色状态 | `system:role:edit` |
| POST | `/api/app/rbac/role/{id}/menu` | 为角色分配菜单 | `system:role:edit` |
| POST | `/api/app/rbac/role/{id}/dept` | 设置角色数据权限 | `system:role:edit` |
| GET | `/api/app/rbac/role/{roleId}/user` | 获取角色下的用户 | `system:role:query` |
| POST | `/api/app/rbac/role/auth-user` | 为用户分配角色 | `system:role:edit` |
| DELETE | `/api/app/rbac/role/auth-user` | 取消用户角色分配 | `system:role:edit` |

### 菜单管理 (MenuService)

| 方法 | 路径 | 描述 | 权限码 |
|-----|------|------|--------|
| GET | `/api/app/rbac/menu` | 获取菜单列表（分页） | `system:menu:list` |
| GET | `/api/app/rbac/menu/{id}` | 获取菜单详情 | `system:menu:query` |
| POST | `/api/app/rbac/menu` | 创建菜单 | `system:menu:add` |
| PUT | `/api/app/rbac/menu/{id}` | 更新菜单 | `system:menu:edit` |
| DELETE | `/api/app/rbac/menu/{id}` | 删除菜单 | `system:menu:remove` |
| DELETE | `/api/app/rbac/menu` | 批量删除菜单 | `system:menu:remove` |
| GET | `/api/app/rbac/menu/role/{roleId}` | 获取角色菜单 | `system:menu:query` |

### 部门管理 (DeptService)

| 方法 | 路径 | 描述 | 权限码 |
|-----|------|------|--------|
| GET | `/api/app/rbac/dept` | 获取部门列表（分页） | `system:dept:list` |
| GET | `/api/app/rbac/dept/{id}` | 获取部门详情 | `system:dept:query` |
| POST | `/api/app/rbac/dept` | 创建部门 | `system:dept:add` |
| PUT | `/api/app/rbac/dept/{id}` | 更新部门 | `system:dept:edit` |
| DELETE | `/api/app/rbac/dept/{id}` | 删除部门 | `system:dept:remove` |
| DELETE | `/api/app/rbac/dept` | 批量删除部门 | `system:dept:remove` |
| GET | `/api/app/rbac/dept/role/{roleId}` | 获取角色部门 | `system:dept:query` |

### 岗位管理 (PostService)

| 方法 | 路径 | 描述 | 权限码 |
|-----|------|------|--------|
| GET | `/api/app/rbac/post` | 获取岗位列表（分页） | `system:post:list` |
| GET | `/api/app/rbac/post/{id}` | 获取岗位详情 | `system:post:query` |
| POST | `/api/app/rbac/post` | 创建岗位 | `system:post:add` |
| PUT | `/api/app/rbac/post/{id}` | 更新岗位 | `system:post:edit` |
| DELETE | `/api/app/rbac/post/{id}` | 删除岗位 | `system:post:remove` |
| DELETE | `/api/app/rbac/post` | 批量删除岗位 | `system:post:remove` |

### 字典管理 (DictionaryService)

| 方法 | 路径 | 描述 | 权限码 |
|-----|------|------|--------|
| GET | `/api/app/rbac/dictionary` | 获取字典列表（分页） | `system:dictionary:list` |
| GET | `/api/app/rbac/dictionary/{id}` | 获取字典详情 | `system:dictionary:query` |
| POST | `/api/app/rbac/dictionary` | 创建字典 | `system:dictionary:add` |
| PUT | `/api/app/rbac/dictionary/{id}` | 更新字典 | `system:dictionary:edit` |
| DELETE | `/api/app/rbac/dictionary/{id}` | 删除字典 | `system:dictionary:remove` |
| DELETE | `/api/app/rbac/dictionary` | 批量删除字典 | `system:dictionary:remove` |

### 字典类型管理 (DictionaryTypeService)

| 方法 | 路径 | 描述 | 权限码 |
|-----|------|------|--------|
| GET | `/api/app/rbac/dictionary-type` | 获取字典类型列表 | `system:dictionary-type:list` |
| GET | `/api/app/rbac/dictionary-type/{id}` | 获取字典类型详情 | `system:dictionary-type:query` |
| POST | `/api/app/rbac/dictionary-type` | 创建字典类型 | `system:dictionary-type:add` |
| PUT | `/api/app/rbac/dictionary-type/{id}` | 更新字典类型 | `system:dictionary-type:edit` |
| DELETE | `/api/app/rbac/dictionary-type/{id}` | 删除字典类型 | `system:dictionary-type:remove` |
| DELETE | `/api/app/rbac/dictionary-type` | 批量删除字典类型 | `system:dictionary-type:remove` |

### 系统配置 (ConfigService)

| 方法 | 路径 | 描述 | 权限码 |
|-----|------|------|--------|
| GET | `/api/app/rbac/config` | 获取配置列表（分页） | `system:config:list` |
| GET | `/api/app/rbac/config/{id}` | 获取配置详情 | `system:config:query` |
| POST | `/api/app/rbac/config` | 创建配置 | `system:config:add` |
| PUT | `/api/app/rbac/config/{id}` | 更新配置 | `system:config:edit` |
| DELETE | `/api/app/rbac/config/{id}` | 删除配置 | `system:config:remove` |
| DELETE | `/api/app/rbac/config` | 批量删除配置 | `system:config:remove` |

### 通知公告 (NoticeService)

| 方法 | 路径 | 描述 | 权限码 |
|-----|------|------|--------|
| GET | `/api/app/rbac/notice` | 获取通知列表（分页） | `system:notice:list` |
| GET | `/api/app/rbac/notice/{id}` | 获取通知详情 | `system:notice:query` |
| POST | `/api/app/rbac/notice` | 创建通知 | `system:notice:add` |
| PUT | `/api/app/rbac/notice/{id}` | 更新通知 | `system:notice:edit` |
| DELETE | `/api/app/rbac/notice/{id}` | 删除通知 | `system:notice:remove` |
| DELETE | `/api/app/rbac/notice` | 批量删除通知 | `system:notice:remove` |
| POST | `/api/app/rbac/notice/online/{id}` | 上线通知 | `system:notice:edit` |
| POST | `/api/app/rbac/notice/offline/{id}` | 下线通知 | `system:notice:edit` |

### 登录日志 (LoginLogService)

| 方法 | 路径 | 描述 | 权限码 |
|-----|------|------|--------|
| GET | `/api/app/rbac/login-log` | 获取登录日志列表（分页） | `system:login-log:list` |
| GET | `/api/app/rbac/login-log/{id}` | 获取登录日志详情 | `system:login-log:query` |

### 操作日志 (OperationLogService)

| 方法 | 路径 | 描述 | 权限码 |
|-----|------|------|--------|
| GET | `/api/app/rbac/operation-log` | 获取操作日志列表（分页） | `system:operation-log:list` |
| GET | `/api/app/rbac/operation-log/{id}` | 获取操作日志详情 | `system:operation-log:query` |

### 在线用户 (OnlineService)

| 方法 | 路径 | 描述 | 权限码 |
|-----|------|------|--------|
| GET | `/api/app/rbac/online` | 获取在线用户列表 | `monitor:online:list` |
| DELETE | `/api/app/rbac/online/{connectionId}` | 强制退出用户 | `monitor:online:force-out` |

### 服务器监控 (MonitorServerService)

| 方法 | 路径 | 描述 | 权限码 |
|-----|------|------|--------|
| GET | `/api/app/rbac/monitor-server/info` | 获取服务器信息 | `monitor:server:info` |

### 缓存监控 (MonitorCacheService)

| 方法 | 路径 | 描述 | 权限码 |
|-----|------|------|--------|
| GET | `/api/app/rbac/monitor-cache/name` | 获取所有缓存名称 | `monitor:cache:list` |
| GET | `/api/app/rbac/monitor-cache/key/{cacheName}` | 获取缓存键列表 | `monitor:cache:list` |
| GET | `/api/app/rbac/monitor-cache/value/{cacheName}/{cacheKey}` | 获取缓存值 | `monitor:cache:query` |
| DELETE | `/api/app/rbac/monitor-cache/key/{cacheName}` | 删除缓存键 | `monitor:cache:remove` |
| DELETE | `/api/app/rbac/monitor-cache/value/{cacheName}/{cacheKey}` | 删除缓存值 | `monitor:cache:remove` |
| DELETE | `/api/app/rbac/monitor-cache/clear` | 清空所有缓存 | `monitor:cache:clear` |

### 账户管理 (AccountService)

| 方法 | 路径 | 描述 | 权限码 |
|-----|------|------|--------|
| GET | `/api/app/account/info` | 获取当前用户信息 | 无需权限 |
| PUT | `/api/app/account/info` | 更新当前用户信息 | 无需权限 |
| GET | `/api/app/account/router/{routerType}` | 获取用户路由菜单 | 无需权限 |
| POST | `/api/app/account/captcha-phone` | 发送手机验证码 | 无需权限 |
| POST | `/api/app/account/captcha-phone/repassword` | 验证码重置密码 | 无需权限 |
| POST | `/api/app/account/captcha-phone/bind` | 绑定手机号 | 无需权限 |
| POST | `/api/app/account/icon` | 上传头像 | 无需权限 |

### 文件管理 (FileService)

| 方法 | 路径 | 描述 | 权限码 |
|-----|------|------|--------|
| GET | `/api/app/rbac/file/{code}` | 获取文件 | `system:file:query` |
| POST | `/api/app/rbac/file` | 上传文件 | `system:file:upload` |
| DELETE | `/api/app/rbac/file/{id}` | 删除文件 | `system:file:remove` |
| GET | `/api/app/rbac/file` | 获取文件列表（分页） | `system:file:list` |

---

## 权限码规范

权限码采用 `模块:功能:操作` 的格式：

| 部分 | 示例 | 说明 |
|-----|------|------|
| 模块 | `system` | 系统模块 |
| 功能 | `user` | 用户功能 |
| 操作 | `list`, `add`, `edit`, `remove`, `query` | 列表查询、新增、修改、删除、详情查询 |

---

## 通用参数说明

### 分页参数

所有列表接口支持分页：

```typescript
interface PagedRequest {
  skipCount: number;      // 跳过记录数（页码 * 每页大小）
  maxResultCount: number; // 每页记录数
  sorting?: string;       // 排序字段（如 "CreationTime Desc"）
}
```

### 批量删除参数

```typescript
interface DeleteManyInput {
  ids: Guid[];  // 要删除的ID列表
}
```

---

## 通用响应格式

### 成功响应

```typescript
interface ApiResult<T> {
  result: T;          // 实际数据
  success: boolean;   // 是否成功
  error?: null;       // 错误信息
}

// 分页响应
interface PagedResultDto<T> {
  totalCount: number;      // 总记录数
  items: T[];               // 数据列表
}
```

### 错误响应

```typescript
interface ErrorResult {
  result: null;
  success: false;
  error: {
    message: string;        // 错误消息
    details?: string;       // 详细信息
    code?: string;          // 错误码
  };
}
```

---

## 认证说明

### JWT 认证

所有 API（除账户登录相关）需要在请求头中携带 JWT Token：

```
Authorization: Bearer {token}
```

### Token 获取

通过登录接口获取：

```bash
POST /api/app/account/my-login
```

---

## 相关文档

- [[用户角色菜单API]] - 用户、角色、菜单详细 API 文档
- [[系统配置API]] - 系统配置与日志 API 文档
- [[监控API]] - 监控与 SignalR Hub API 文档
- [[账户模块API]] - 账户认证与登录 API 文档
- [[权限系统设计]] - 权限系统设计文档
- [[数据权限设计]] - 数据权限实现文档

---

## 快速导航

| 文档 | 描述 |
|-----|------|
| [[用户角色菜单API]] | 用户管理、角色管理、菜单管理、部门管理、岗位管理 |
| [[系统配置API]] | 字典管理、系统配置、通知公告、日志管理 |
| [[监控API]] | 在线用户、服务器监控、缓存监控 |
