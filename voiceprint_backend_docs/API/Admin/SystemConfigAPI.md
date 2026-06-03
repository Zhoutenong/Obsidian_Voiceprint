# 系统配置 API 文档

## 概述

本文档详细说明 RBAC 模块中的系统配置、字典管理、通知公告、登录日志和操作日志的管理接口。

## 基础路径

```
/api/app/rbac
```

---

## 字典管理

字典管理用于维护系统中的数据字典，如性别、状态等枚举值。

### 获取字典列表

**接口**: `GET /api/app/rbac/dictionary`

**描述**: 获取字典列表，支持分页和条件查询。

**权限**: `system:dictionary:list`

**请求参数**:

```typescript
interface DictionaryGetListInputVo {
  skipCount: number;           // 跳过记录数（必填）
  maxResultCount: number;      // 每页记录数（必填）
  dictionaryType?: string;     // 字典类型（模糊查询）
  dictLabel?: string;          // 字典标签（模糊查询）
  state?: boolean;             // 状态（true=启用，false=禁用）
  sorting?: string;            // 排序字段
}
```

**响应示例**:
```json
{
  "result": {
    "totalCount": 50,
    "items": [
      {
        "id": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
        "dictType": "sys_user_sex",
        "dictTypeLabel": "用户性别",
        "dictLabel": "男",
        "dictValue": "1",
        "dictSort": 1,
        "cssClass": "default",
        "listClass": "primary",
        "isDefault": false,
        "state": true,
        "remark": "性别男",
        "creationTime": "2024-01-01T00:00:00Z"
      }
    ]
  }
}
```

---

### 创建字典

**接口**: `POST /api/app/rbac/dictionary`

**描述**: 创建新字典项。

**权限**: `system:dictionary:add`

**请求参数**:

```typescript
interface DictionaryCreateInputVo {
  dictType: string;           // 字典类型（必填）
  dictLabel: string;          // 字典标签（必填）
  dictValue: string;          // 字典键值（必填）
  dictSort?: number;          // 显示顺序
  cssClass?: string;          // 样式属性
  listClass?: string;         // 表格回显样式
  isDefault?: boolean;        // 是否默认值
  state?: boolean;            // 状态
  remark?: string;            // 备注
}
```

**响应**: 返回创建的字典信息 `DictionaryGetOutputDto`

---

### 更新字典

**接口**: `PUT /api/app/rbac/dictionary/{id}`

**描述**: 更新字典信息。

**权限**: `system:dictionary:edit`

**路径参数**:

| 参数 | 类型 | 说明 |
|-----|------|------|
| id | Guid | 字典ID |

**请求参数**: 同 `DictionaryCreateInputVo`

**响应**: 返回更新后的字典信息 `DictionaryGetOutputDto`

---

### 删除字典

**接口**: `DELETE /api/app/rbac/dictionary/{id}`

**描述**: 删除指定字典项。

**权限**: `system:dictionary:remove`

**路径参数**:

| 参数 | 类型 | 说明 |
|-----|------|------|
| id | Guid | 字典ID |

**响应**: `204 No Content`

---

## 字典类型管理

### 获取字典类型列表

**接口**: `GET /api/app/rbac/dictionary-type`

**描述**: 获取字典类型列表。

**权限**: `system:dictionary-type:list`

**请求参数**:

```typescript
interface DictionaryTypeGetListInputVo {
  skipCount: number;           // 跳过记录数（必填）
  maxResultCount: number;      // 每页记录数（必填）
  dictName?: string;           // 字典名称（模糊查询）
  dictType?: string;           // 字典类型（模糊查询）
  state?: boolean;             // 状态（true=启用，false=禁用）
  sorting?: string;            // 排序字段
}
```

**响应示例**:
```json
{
  "result": {
    "totalCount": 10,
    "items": [
      {
        "id": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
        "dictName": "用户性别",
        "dictType": "sys_user_sex",
        "state": true,
        "remark": "用户性别列表",
        "creationTime": "2024-01-01T00:00:00Z"
      }
    ]
  }
}
```

---

### 创建字典类型

**接口**: `POST /api/app/rbac/dictionary-type`

**描述**: 创建新字典类型。

**权限**: `system:dictionary-type:add`

**请求参数**:

```typescript
interface DictionaryTypeCreateInputVo {
  dictName: string;           // 字典名称（必填）
  dictType: string;           // 字典类型（必填，唯一）
  state?: boolean;            // 状态
  remark?: string;            // 备注
}
```

**响应**: 返回创建的字典类型信息 `DictionaryTypeGetOutputDto`

---

## 系统配置管理

系统配置管理用于维护系统的全局配置参数。

### 获取配置列表

**接口**: `GET /api/app/rbac/config`

**描述**: 获取系统配置列表，支持分页和条件查询。

**权限**: `system:config:list`

**请求参数**:

```typescript
interface ConfigGetListInputVo {
  skipCount: number;           // 跳过记录数（必填）
  maxResultCount: number;      // 每页记录数（必填）
  configName?: string;         // 配置名称（模糊查询）
  configKey?: string;          // 配置键名（模糊查询）
  configType?: string;         // 配置类型
  sorting?: string;            // 排序字段
}
```

**响应示例**:
```json
{
  "result": {
    "totalCount": 20,
    "items": [
      {
        "id": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
        "configName": "主框架页脚",
        "configKey": "sys.index.footerName",
        "configValue": "IntelliSubstation",
        "configType": "primary",
        "isCache": true,
        "remark": "系统主框架页脚显示名称",
        "creationTime": "2024-01-01T00:00:00Z"
      }
    ]
  }
}
```

---

### 获取配置详情

**接口**: `GET /api/app/rbac/config/{id}`

**描述**: 获取指定配置的详细信息。

**权限**: `system:config:query`

**路径参数**:

| 参数 | 类型 | 说明 |
|-----|------|------|
| id | Guid | 配置ID |

**响应**: 返回配置信息 `ConfigGetOutputDto`

---

### 创建配置

**接口**: `POST /api/app/rbac/config`

**描述**: 创建新系统配置。

**权限**: `system:config:add`

**请求参数**:

```typescript
interface ConfigCreateInputVo {
  configName: string;           // 配置名称（必填）
  configKey: string;           // 配置键名（必填，唯一）
  configValue: string;         // 配置键值（必填）
  configType?: string;         // 配置类型
  isCache?: boolean;            // 是否缓存
  remark?: string;              // 备注
}
```

**响应**: 返回创建的配置信息 `ConfigGetOutputDto`

---

### 更新配置

**接口**: `PUT /api/app/rbac/config/{id}`

**描述**: 更新系统配置。

**权限**: `system:config:edit`

**路径参数**:

| 参数 | 类型 | 说明 |
|-----|------|------|
| id | Guid | 配置ID |

**请求参数**: 同 `ConfigCreateInputVo`

**响应**: 返回更新后的配置信息 `ConfigGetOutputDto`

---

### 删除配置

**接口**: `DELETE /api/app/rbac/config/{id}`

**描述**: 删除指定配置。

**权限**: `system:config:remove`

**路径参数**:

| 参数 | 类型 | 说明 |
|-----|------|------|
| id | Guid | 配置ID |

**响应**: `204 No Content`

---

## 通知公告管理

通知公告管理用于发布和管理系统通知公告。

### 获取通知列表

**接口**: `GET /api/app/rbac/notice`

**描述**: 获取通知公告列表，支持分页和条件查询。

**权限**: `system:notice:list`

**请求参数**:

```typescript
interface NoticeGetListInput {
  skipCount: number;           // 跳过记录数（必填）
  maxResultCount: number;      // 每页记录数（必填）
  noticeTitle?: string;        // 通知标题（模糊查询）
  noticeType?: string;         // 通知类型（notice=通知，announcement=公告）
  noticeSourceType?: string;   // 通知来源
  state?: boolean;             // 状态（true=上线，false=下线）
  sorting?: string;            // 排序字段
}
```

**响应示例**:
```json
{
  "result": {
    "totalCount": 15,
    "items": [
      {
        "id": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
        "noticeTitle": "系统维护通知",
        "noticeType": "notice",
        "noticeTypeName": "通知",
        "noticeSource": "admin",
        "noticeSourceType": "系统",
        "content": "系统将于今晚进行维护...",
        "state": true,
        "isTop": true,
        "orderNum": 1,
        "creationTime": "2024-01-01T00:00:00Z"
      }
    ]
  }
}
```

---

### 创建通知

**接口**: `POST /api/app/rbac/notice`

**描述**: 创建新通知公告。

**权限**: `system:notice:add`

**请求参数**:

```typescript
interface NoticeCreateInput {
  noticeTitle: string;           // 通知标题（必填）
  noticeType: string;           // 通知类型（notice=通知，announcement=公告）
  noticeSource: string;         // 通知来源（必填）
  content: string;              // 通知内容（必填）
  state?: boolean;              // 状态（默认为下线）
  isTop?: boolean;              // 是否置顶
  orderNum?: number;            // 排序号
}
```

**响应**: 返回创建的通知信息 `NoticeGetOutputDto`

---

### 更新通知

**接口**: `PUT /api/app/rbac/notice/{id}`

**描述**: 更新通知公告。

**权限**: `system:notice:edit`

**路径参数**:

| 参数 | 类型 | 说明 |
|-----|------|------|
| id | Guid | 通知ID |

**请求参数**: 同 `NoticeCreateInput`

**响应**: 返回更新后的通知信息 `NoticeGetOutputDto`

---

### 上线通知

**接口**: `POST /api/app/rbac/notice/online/{id}`

**描述**: 上线通知公告。

**权限**: `system:notice:edit`

**路径参数**:

| 参数 | 类型 | 说明 |
|-----|------|------|
| id | Guid | 通知ID |

**响应**: 返回更新后的通知信息 `NoticeGetOutputDto`

---

### 下线通知

**接口**: `POST /api/app/rbac/notice/offline/{id}`

**描述**: 下线通知公告。

**权限**: `system:notice:edit`

**路径参数**:

| 参数 | 类型 | 说明 |
|-----|------|------|
| id | Guid | 通知ID |

**响应**: 返回更新后的通知信息 `NoticeGetOutputDto`

---

## 登录日志管理

登录日志记录用户的登录行为。

### 获取登录日志列表

**接口**: `GET /api/app/rbac/login-log`

**描述**: 获取登录日志列表，支持分页和条件查询。

**权限**: `system:login-log:list`

**请求参数**:

```typescript
interface LoginLogGetListInputVo {
  skipCount: number;           // 跳过记录数（必填）
  maxResultCount: number;      // 每页记录数（必填）
  loginUser?: string;          // 登录用户（模糊查询）
  loginIp?: string;             // 登录IP（模糊查询）
  startTime?: Date;             // 开始时间
  endTime?: Date;               // 结束时间
  sorting?: string;            // 排序字段
}
```

**响应示例**:
```json
{
  "result": {
    "totalCount": 1000,
    "items": [
      {
        "id": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
        "loginUser": "admin",
        "loginIp": "192.168.1.100",
        "loginLocation": "广东省深圳市",
        "browser": "Chrome",
        "os": "Windows 10",
        "msg": "登录成功",
        "state": true,
        "creationTime": "2024-01-01T00:00:00Z"
      }
    ]
  }
}
```

---

### 获取登录日志详情

**接口**: `GET /api/app/rbac/login-log/{id}`

**描述**: 获取指定登录日志的详细信息。

**权限**: `system:login-log:query`

**路径参数**:

| 参数 | 类型 | 说明 |
|-----|------|------|
| id | Guid | 日志ID |

**响应**: 返回登录日志信息 `LoginLogGetListOutputDto`

---

## 操作日志管理

操作日志记录用户的操作行为。

### 获取操作日志列表

**接口**: `GET /api/app/rbac/operation-log`

**描述**: 获取操作日志列表，支持分页和条件查询。

**权限**: `system:operation-log:list`

**请求参数**:

```typescript
interface OperationLogGetListInputVo {
  skipCount: number;           // 跳过记录数（必填）
  maxResultCount: number;      // 每页记录数（必填）
  title?: string;              // 操作模块（模糊查询）
  operUser?: string;           // 操作人员（模糊查询）
  operType?: string;           // 操作类型（other=其他，insert=新增，update=修改，delete=删除，grant=授权）
  startTime?: Date;            // 开始时间
  endTime?: Date;              // 结束时间
  sorting?: string;            // 排序字段
}
```

**响应示例**:
```json
{
  "result": {
    "totalCount": 5000,
    "items": [
      {
        "id": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
        "title": "用户管理",
        "businessType": "user",
        "method": "com.example.controller.UserController.createUser",
        "requestMethod": "POST",
        "operType": "insert",
        "operTypeName": "新增",
        "operUser": "admin",
        "operUrl": "/api/app/rbac/user",
        "operIp": "192.168.1.100",
        "operLocation": "广东省深圳市",
        "operParam": "{\"userName\":\"test\",\"nickName\":\"测试\"}",
        "jsonResult": "{\"result\":{\"id\":\"...\",\"userName\":\"test\"}}",
        "state": true,
        "errorMsg": null,
        "costTime": 150,
        "creationTime": "2024-01-01T00:00:00Z"
      }
    ]
  }
}
```

---

### 获取操作日志详情

**接口**: `GET /api/app/rbac/operation-log/{id}`

**描述**: 获取指定操作日志的详细信息。

**权限**: `system:operation-log:query`

**路径参数**:

| 参数 | 类型 | 说明 |
|-----|------|------|
| id | Guid | 日志ID |

**响应**: 返回操作日志信息 `OperationLogGetListOutputDto`

---

## 数据结构

### 字典数据结构

```typescript
interface DictionaryGetOutputDto {
  id: Guid;                    // 字典ID
  dictType: string;            // 字典类型
  dictTypeLabel: string;       // 字典类型名称
  dictLabel: string;           // 字典标签
  dictValue: string;           // 字典键值
  dictSort: number;            // 显示顺序
  cssClass: string;            // 样式属性
  listClass: string;           // 表格回显样式
  isDefault: boolean;          // 是否默认值
  state: boolean;              // 状态
  remark: string;              // 备注
  creationTime: Date;          // 创建时间
}
```

### 系统配置数据结构

```typescript
interface ConfigGetOutputDto {
  id: Guid;                    // 配置ID
  configName: string;          // 配置名称
  configKey: string;          // 配置键名
  configValue: string;        // 配置键值
  configType: string;          // 配置类型
  isCache: boolean;            // 是否缓存
  remark: string;              // 备注
  creationTime: Date;          // 创建时间
}
```

### 通知公告数据结构

```typescript
interface NoticeGetOutputDto {
  id: Guid;                    // 通知ID
  noticeTitle: string;         // 通知标题
  noticeType: string;         // 通知类型（notice=通知，announcement=公告）
  noticeTypeName: string;     // 通知类型名称
  noticeSource: string;       // 通知来源
  noticeSourceType: string;   // 通知来源类型
  content: string;            // 通知内容
  state: boolean;              // 状态（true=上线，false=下线）
  isTop: boolean;             // 是否置顶
  orderNum: number;            // 排序号
  creationTime: Date;          // 创建时间
}
```

### 登录日志数据结构

```typescript
interface LoginLogGetListOutputDto {
  id: Guid;                    // 日志ID
  loginUser: string;           // 登录用户
  loginIp: string;             // 登录IP
  loginLocation: string;       // 登录地点
  browser: string;              // 浏览器类型
  os: string;                  // 操作系统
  msg: string;                 // 登录消息
  state: boolean;              // 登录状态（true=成功，false=失败）
  creationTime: Date;          // 登录时间
}
```

### 操作日志数据结构

```typescript
interface OperationLogGetListOutputDto {
  id: Guid;                    // 日志ID
  title: string;               // 操作模块
  businessType: string;        // 业务类型
  method: string;              // 方法名称
  requestMethod: string;       // 请求方式
  operType: string;           // 操作类型（other/insert/update/delete/grant）
  operTypeName: string;        // 操作类型名称
  operUser: string;            // 操作人员
  operUrl: string;             // 请求URL
  operIp: string;              // 操作IP
  operLocation: string;        // 操作地点
  operParam: string;           // 请求参数
  jsonResult: string;         // 返回结果
  state: boolean;              // 操作状态（true=成功，false=失败）
  errorMsg: string;            // 错误消息
  costTime: number;            // 消耗时间（毫秒）
  creationTime: Date;          // 操作时间
}
```

---

## 前端使用示例

### 获取字典列表

```typescript
const response = await fetch('/api/app/rbac/dictionary?skipCount=0&maxResultCount=100&dictType=sys_user_sex', {
  headers: {
    'Authorization': `Bearer ${token}`
  }
});

const data = await response.json();
const genders = data.result.items;
console.log(genders);  // 性别字典列表
```

### 创建系统配置

```typescript
const response = await fetch('/api/app/rbac/config', {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json',
    'Authorization': `Bearer ${token}`
  },
  body: JSON.stringify({
    configName: '系统名称',
    configKey: 'sys.app.name',
    configValue: 'IntelliSubstation',
    configType: 'primary',
    isCache: true
  })
});

const config = await response.json();
console.log(config.result);  // 创建的配置信息
```

### 发布通知

```typescript
const response = await fetch('/api/app/rbac/notice', {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json',
    'Authorization': `Bearer ${token}`
  },
  body: JSON.stringify({
    noticeTitle: '系统维护通知',
    noticeType: 'notice',
    noticeSource: 'admin',
    content: '系统将于今晚22:00-24:00进行维护，请提前做好准备。',
    isTop: true,
    state: false  // 默认下线，需要手动上线
  })
});
```

### 上线通知

```typescript
const response = await fetch(`/api/app/rbac/notice/online/${noticeId}`, {
  method: 'POST',
  headers: {
    'Authorization': `Bearer ${token}`
  }
});
```

### 查询登录日志

```typescript
const response = await fetch('/api/app/rbac/login-log?skipCount=0&maxResultCount=20&startTime=2024-01-01&endTime=2024-01-31', {
  headers: {
    'Authorization': `Bearer ${token}`
  }
});

const data = await response.json();
console.log(data.result.items);  // 登录日志列表
```

---

## 注意事项

1. **配置缓存**: 系统配置支持缓存机制，修改配置后可能需要清理缓存才能生效
2. **通知发布**: 通知创建后默认为下线状态，需要手动上线才能显示给用户
3. **日志保留**: 登录日志和操作日志建议定期清理，避免数据量过大
4. **敏感数据**: 操作日志中的请求参数可能包含敏感信息，需要谨慎处理

---

## 相关文档

- [[AdminAPI索引]] - RBAC 管理端 API 总览
- [[用户角色菜单API]] - 用户、角色、菜单管理 API 文档
- [[监控API]] - 监控与 SignalR Hub API 文档
- [[日志系统设计]] - 日志系统设计文档
