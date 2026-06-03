# IntelliSub 自定义 HTTP API 文档

本文档详细说明了 `ast-intellisub` 模块中 ABP 自动 CRUD 之外的自定义 HTTP 端点。

## 目录

- [PatrolRecordService（巡检记录）](#patrolrecordservice巡检记录)
- [PatrolTaskService（巡检任务）](#patroltaskservice巡检任务)
- [AlarmRecordService（告警记录）](#alarmrecordservice告警记录)
- [PointDataService（点位数据）](#pointdataservice点位数据)
- [StrategyStateService（策略状态）](#strategystateservice策略状态)
- [AlarmCategoryService（告警分类）](#alarmcategoryservice告警分类)
- [ReportController（报告）](#reportcontroller报告)

---

## PatrolRecordService（巡检记录）

### 1. 停止巡检

**接口路径：** `POST /api/app/patrol-record/{id}/stop`

**功能说明：** 终止正在执行的巡检任务

**请求参数：**

| 参数名 | 类型 | 位置 | 必填 | 说明 |
|--------|------|------|------|------|
| id | Guid | path | 是 | 巡检记录ID |

**响应格式：** 无返回内容（HTTP 204）

**使用场景：**
- 用户手动停止正在执行的巡检任务
- 系统检测到异常情况需要中止巡检
- 巡检任务执行时间过长需要强制终止

**示例：**
```bash
POST /api/app/patrol-record/3fa85f64-5717-4562-b3fc-2c963f66afa6/stop
```

---

### 2. 系统恢复

**接口路径：** `POST /api/app/patrol-record/system/recover`

**功能说明：** 手动触发系统启动时的异常恢复处理，检测并处理因进程异常终止而留下的僵尸任务

**请求参数：** 无

**响应格式：** 无返回内容（HTTP 204）

**使用场景：**
- 系统异常重启后需要清理僵尸任务
- 巡检任务状态异常时手动恢复
- 定期维护时清理遗留任务

**示例：**
```bash
POST /api/app/patrol-record/system/recover
```

---

### 3. 系统清理

**接口路径：** `POST /api/app/patrol-record/system/cleanup`

**功能说明：** 手动触发检查并清理长时间运行的任务

**请求参数：** 无

**响应格式：** 无返回内容（HTTP 204）

**使用场景：**
- 清理运行时间超过预期最大时长的巡检任务
- 定期维护时清理长时间运行的任务
- 系统资源紧张时释放被长时间任务占用的资源

**示例：**
```bash
POST /api/app/patrol-record/system/cleanup
```

---

### 4. 健康检查

**接口路径：** `GET /api/app/patrol-record/system/health`

**功能说明：** 获取巡检系统的健康状态，包括任务统计、僵尸任务信息等

**请求参数：** 无

**响应格式：**
```json
{
  "checkTime": "2026-06-03T10:30:00",
  "isHealthy": true,
  "totalTasks": 100,
  "completedTasks": 85,
  "abortedTasks": 5,
  "runningTasks": 3,
  "abnormalResults": 7,
  "completionRate": 85.0,
  "abnormalRate": 7.0,
  "zombieTasks": [
    {
      "recordId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
      "taskName": "日常巡检任务",
      "startTime": "2026-06-03T08:00:00",
      "runningDuration": "2小时30分钟",
      "expectedMaxDuration": "2小时0分钟",
      "isTimeout": true
    }
  ]
}
```

**响应字段说明：**

| 字段 | 类型 | 说明 |
|------|------|------|
| checkTime | DateTime | 健康检查时间 |
| isHealthy | bool | 系统是否健康 |
| totalTasks | int | 最近24小时总任务数 |
| completedTasks | int | 已完成任务数 |
| abortedTasks | int | 已中止任务数 |
| runningTasks | int | 正在运行任务数 |
| abnormalResults | int | 异常结果任务数 |
| completionRate | double | 任务完成率（百分比） |
| abnormalRate | double | 异常率（百分比） |
| zombieTasks | array | 僵尸任务列表 |

**使用场景：**
- 系统监控面板显示巡检系统状态
- 定期健康检查
- 运维人员查看系统运行情况

**示例：**
```bash
GET /api/app/patrol-record/system/health
```

---

## PatrolTaskService（巡检任务）

### 1. 启用/禁用任务

**接口路径：** `PUT /api/app/patrol-task/{id}/enabled`

**功能说明：** 启用或禁用指定的巡检任务

**请求参数：**

**Path 参数：**

| 参数名 | 类型 | 必填 | 说明 |
|--------|------|------|------|
| id | Guid | 是 | 巡检任务ID |

**Body 参数：**
```json
{
  "enabled": true
}
```

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| enabled | bool | 是 | 是否启用 |

**响应格式：**
```json
{
  "id": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
  "name": "日常巡检任务",
  "enabled": true,
  // ... 其他任务字段
}
```

**使用场景：**
- 临时暂停某个巡检任务
- 启用已配置的巡检任务
- 维护期间禁用巡检任务

**示例：**
```bash
PUT /api/app/patrol-task/3fa85f64-5717-4562-b3fc-2c963f66afa6/enabled
Content-Type: application/json

{
  "enabled": false
}
```

---

### 2. 测试 Cron 表达式

**接口路径：** `POST /api/app/patrol-task/test-cron`

**功能说明：** 测试 Cron 表达式的有效性并返回未来几次执行时间

**请求参数：**

**Body 参数：**
```json
{
  "name": "测试任务",
  "cronExpression": "0 0 2 * * ?",
  "substationId": "3fa85f64-5717-4562-b3fc-2c963f66afa6"
  // ... 其他任务配置字段
}
```

**响应格式：**
```json
{
  "isValid": true,
  "nextExecutions": [
    "2026-06-04T02:00:00",
    "2026-06-05T02:00:00",
    "2026-06-06T02:00:00",
    "2026-06-07T02:00:00",
    "2026-06-08T02:00:00"
  ],
  "errorMessage": null
}
```

**响应字段说明：**

| 字段 | 类型 | 说明 |
|------|------|------|
| isValid | bool | Cron 表达式是否有效 |
| nextExecutions | array | 未来5次执行时间（ISO 8601格式） |
| errorMessage | string | 错误信息（表达式无效时） |

**使用场景：**
- 创建或修改巡检任务前验证 Cron 表达式
- 查看任务的执行计划
- 调试定时任务配置

**示例：**
```bash
POST /api/app/patrol-task/test-cron
Content-Type: application/json

{
  "name": "测试任务",
  "cronExpression": "0 0 2 * * ?",
  "substationId": "3fa85f64-5717-4562-b3fc-2c963f66afa6"
}
```

---

## AlarmRecordService（告警记录）

### 1. 批量更新状态（按条件）

**接口路径：** `PUT /api/app/alarm-record/batch-update-status`

**功能说明：** 按条件批量更新告警记录的处理状态

**请求参数：**

**Body 参数：**
```json
{
  "substationId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
  "targetStatus": 2,
  "filterStatus": 1,
  "alarmLevel": 3,
  "startTime": "2026-06-01T00:00:00",
  "endTime": "2026-06-03T23:59:59",
  "monitoredObjectIds": [
    "3fa85f64-5717-4562-b3fc-2c963f66afa6"
  ],
  "monitoredPointIds": [
    "3fa85f64-5717-4562-b3fc-2c963f66afb6"
  ],
  "updateReason": "批量处理完成"
}
```

**请求字段说明：**

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| substationId | Guid | 是 | 变电站ID |
| targetStatus | int | 是 | 目标状态（1-未处理，2-已处理，3-已忽略） |
| filterStatus | int | 否 | 按当前状态过滤（可选） |
| alarmLevel | int | 否 | 按告警级别过滤（可选） |
| startTime | DateTime | 否 | 开始时间过滤（可选） |
| endTime | DateTime | 否 | 结束时间过滤（可选） |
| monitoredObjectIds | array | 否 | 监测对象ID列表（可选） |
| monitoredPointIds | array | 否 | 监测点位ID列表（可选） |
| updateReason | string | 否 | 更新原因（可选，最多500字符） |

**响应格式：**
```json
{
  "success": true,
  "updatedCount": 150,
  "failedCount": 0,
  "message": "成功更新150条告警记录"
}
```

**使用场景：**
- 批量处理某个变电站的所有未处理告警
- 批量忽略某个级别的低优先级告警
- 定期维护时批量标记告警为已处理

**示例：**
```bash
PUT /api/app/alarm-record/batch-update-status
Content-Type: application/json

{
  "substationId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
  "targetStatus": 2,
  "filterStatus": 1,
  "updateReason": "批量处理完成"
}
```

---

### 2. 批量更新（按ID）

**接口路径：** `PUT /api/app/alarm-record/batch-update-by-ids`

**功能说明：** 按告警记录ID列表批量更新处理状态

**请求参数：**

**Body 参数：**
```json
{
  "alarmIds": [
    "3fa85f64-5717-4562-b3fc-2c963f66afa6",
    "3fa85f64-5717-4562-b3fc-2c963f66afa7",
    "3fa85f64-5717-4562-b3fc-2c963f66afa8"
  ],
  "targetStatus": 2,
  "updateReason": "已确认处理"
}
```

**请求字段说明：**

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| alarmIds | array | 是 | 告警记录ID列表 |
| targetStatus | int | 是 | 目标状态（1-未处理，2-已处理，3-已忽略） |
| updateReason | string | 否 | 更新原因（可选） |

**响应格式：**
```json
{
  "success": true,
  "updatedCount": 3,
  "failedCount": 0,
  "message": "成功更新3条告警记录"
}
```

**使用场景：**
- 用户在告警列表中选择多条告警批量处理
- 批量标记特定告警为已处理
- 批量忽略已确认的误报

**示例：**
```bash
PUT /api/app/alarm-record/batch-update-by-ids
Content-Type: application/json

{
  "alarmIds": [
    "3fa85f64-5717-4562-b3fc-2c963f66afa6",
    "3fa85f64-5717-4562-b3fc-2c963f66afa7"
  ],
  "targetStatus": 2,
  "updateReason": "已确认处理"
}
```

---

## PointDataService（点位数据）

### 1. 历史数据查询

**接口路径：** `GET /api/app/point-data/history`

**功能说明：** 查询点位的历史数据，支持时间范围聚合和降采样，用于绘制趋势图

**请求参数：**

**Query 参数：**

| 参数名 | 类型 | 必填 | 说明 |
|--------|------|------|------|
| deviceId | string | 是 | 设备ID |
| sensorKey | string | 是 | 传感器Key |
| startTime | DateTime | 是 | 开始时间（ISO 8601） |
| endTime | DateTime | 是 | 结束时间（ISO 8601） |
| intervalInMinutes | int | 否 | 聚合间隔（分钟），默认1分钟 |
| skipCount | int | 否 | 跳过记录数，默认0 |
| maxResultCount | int | 否 | 最大返回记录数，默认10 |

**响应格式：**
```json
{
  "totalCount": 1440,
  "items": [
    {
      "timestamp": "2026-06-03T10:00:00",
      "value": 25.5,
      "deviceId": "gateway001",
      "sensorKey": "temperature_01",
      "unit": "℃"
    },
    {
      "timestamp": "2026-06-03T10:01:00",
      "value": 25.8,
      "deviceId": "gateway001",
      "sensorKey": "temperature_01",
      "unit": "℃"
    }
  ]
}
```

**使用场景：**
- 绘制点位数据趋势图
- 分析历史数据变化
- 生成报表时查询历史数据
- 数据分析平台查询时序数据

**示例：**
```bash
GET /api/app/point-data/history?deviceId=gateway001&sensorKey=temperature_01&startTime=2026-06-03T00:00:00&endTime=2026-06-03T23:59:59&intervalInMinutes=5&skipCount=0&maxResultCount=100
```

---

### 2. 按告警状态过滤

**接口路径：** `GET /api/app/point-data/filtered-by-alarm-status`

**功能说明：** 根据告警状态过滤点位数据，返回正常或异常的数据记录

**请求参数：**

**Query 参数：**

| 参数名 | 类型 | 必填 | 说明 |
|--------|------|------|------|
| deviceId | string | 否 | 设备ID |
| sensorKey | string | 否 | 传感器Key |
| isOpen | bool | 是 | 是否为异常数据（true=异常，false=正常） |
| startTime | DateTime | 否 | 开始时间（ISO 8601） |
| endTime | DateTime | 否 | 结束时间（ISO 8601） |
| skipCount | int | 否 | 跳过记录数，默认0 |
| maxResultCount | int | 否 | 最大返回记录数，默认10 |

**响应格式：**
```json
{
  "totalCount": 250,
  "items": [
    {
      "id": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
      "deviceId": "gateway001",
      "sensorKey": "temperature_01",
      "value": 85.5,
      "timestamp": "2026-06-03T10:30:00",
      "isOpen": true,
      "alarmLevel": 4
    }
  ]
}
```

**使用场景：**
- 查询所有异常数据用于分析
- 查询正常数据作为基准
- 告警分析时获取关联数据
- 数据质量检查

**示例：**
```bash
GET /api/app/point-data/filtered-by-alarm-status?deviceId=gateway001&isOpen=true&startTime=2026-06-03T00:00:00&endTime=2026-06-03T23:59:59&skipCount=0&maxResultCount=50
```

---

## StrategyStateService（策略状态）

### 1. 清除所有状态

**接口路径：** `PUT /api/app/strategy-state/clear-all`

**功能说明：** 清空所有策略状态记录

**请求参数：** 无

**响应格式：**
```json
{
  "success": true,
  "totalCount": 1000,
  "clearedCount": 1000,
  "failedCount": 0,
  "message": "成功清空1000条策略状态"
}
```

**使用场景：**
- 系统重置后清空所有状态
- 策略配置变更后需要重新计算状态
- 数据清理时删除历史状态

**示例：**
```bash
PUT /api/app/strategy-state/clear-all
```

---

### 2. 按策略关系清除状态

**接口路径：** `PUT /api/app/strategy-state/clear-by-binding-rel-ids`

**功能说明：** 批量清空指定绑定项策略关系的所有状态

**请求参数：**

**Body 参数：**
```json
[
  "3fa85f64-5717-4562-b3fc-2c963f66afa6",
  "3fa85f64-5717-4562-b3fc-2c963f66afa7",
  "3fa85f64-5717-4562-b3fc-2c963f66afa8"
]
```

**响应格式：**
```json
{
  "success": true,
  "totalCount": 150,
  "clearedCount": 150,
  "failedCount": 0,
  "message": "成功清空150条策略状态"
}
```

**使用场景：**
- 特定策略配置变更后清除相关状态
- 删除策略关系时清理关联状态
- 重新初始化特定策略

**示例：**
```bash
PUT /api/app/strategy-state/clear-by-binding-rel-ids
Content-Type: application/json

[
  "3fa85f64-5717-4562-b3fc-2c963f66afa6",
  "3fa85f64-5717-4562-b3fc-2c963f66afa7"
]
```

---

### 3. 按点位关系清除状态

**接口路径：** `PUT /api/app/strategy-state/clear-by-point-binding-rel-ids`

**功能说明：** 批量清空指定点位绑定关系的所有策略状态

**请求参数：**

**Body 参数：**
```json
[
  "3fa85f64-5717-4562-b3fc-2c963f66afa6",
  "3fa85f64-5717-4562-b3fc-2c963f66afa7"
]
```

**响应格式：**
```json
{
  "success": true,
  "totalCount": 50,
  "clearedCount": 50,
  "failedCount": 0,
  "message": "成功清空50条策略状态"
}
```

**使用场景：**
- 点位配置变更后清除相关策略状态
- 删除点位时清理关联状态
- 重新初始化点位的策略计算

**示例：**
```bash
PUT /api/app/strategy-state/clear-by-point-binding-rel-ids
Content-Type: application/json

[
  "3fa85f64-5717-4562-b3fc-2c963f66afa6"
]
```

---

### 4. 批量删除

**接口路径：** `DELETE /api/app/strategy-state/batch`

**功能说明：** 批量删除指定的策略状态记录

**请求参数：**

**Body 参数：**
```json
[
  "3fa85f64-5717-4562-b3fc-2c963f66afa6",
  "3fa85f64-5717-4562-b3fc-2c963f66afa7",
  "3fa85f64-5717-4562-b3fc-2c963f66afa8"
]
```

**响应格式：**
```json
{
  "success": true,
  "totalCount": 3,
  "clearedCount": 3,
  "failedCount": 0,
  "message": "成功删除3条策略状态"
}
```

**使用场景：**
- 删除指定的策略状态记录
- 数据清理时删除特定状态
- 误创建状态时的删除操作

**示例：**
```bash
DELETE /api/app/strategy-state/batch
Content-Type: application/json

[
  "3fa85f64-5717-4562-b3fc-2c963f66afa6",
  "3fa85f64-5717-4562-b3fc-2c963f66afa7"
]
```

---

## AlarmCategoryService（告警分类）

### 1. 获取所有分类

**接口路径：** `GET /api/app/alarm-category/all`

**功能说明：** 获取所有告警分类，用于下拉选择等场景

**请求参数：** 无

**响应格式：**
```json
[
  {
    "id": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
    "name": "温度告警",
    "code": "TEMP_ALARM",
    "type": 1,
    "level": 3,
    "description": "设备温度异常告警"
  },
  {
    "id": "3fa85f64-5717-4562-b3fc-2c963f66afa7",
    "name": "声纹告警",
    "code": "VOICEPRINT_ALARM",
    "type": 2,
    "level": 4,
    "description": "声纹识别异常告警"
  }
]
```

**使用场景：**
- 告警配置页面的分类选择
- 告警筛选器中的分类选项
- 报表统计时的分类维度

**示例：**
```bash
GET /api/app/alarm-category/all
```

---

### 2. 按类型查询

**接口路径：** `GET /api/app/alarm-category/by-type`

**功能说明：** 根据类别类型获取告警分类列表

**请求参数：**

**Query 参数：**

| 参数名 | 类型 | 必填 | 说明 |
|--------|------|------|------|
| type | int | 是 | 类别类型（1-设备告警，2-环境告警，3-安全告警等） |

**响应格式：**
```json
[
  {
    "id": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
    "name": "温度告警",
    "code": "TEMP_ALARM",
    "type": 1,
    "level": 3,
    "description": "设备温度异常告警"
  },
  {
    "id": "3fa85f64-5717-4562-b3fc-2c963f66afa7",
    "name": "湿度告警",
    "code": "HUMIDITY_ALARM",
    "type": 1,
    "level": 2,
    "description": "环境湿度异常告警"
  }
]
```

**使用场景：**
- 根据告警类型筛选分类
- 特定类型告警的配置界面
- 按类型统计告警数据

**示例：**
```bash
GET /api/app/alarm-category/by-type?type=1
```

---

## ReportController（报告）

### 1. 综合运行报告

**接口路径：** `POST /api/app/report/comprehensive-operation-report`

**功能说明：** 生成变电站综合运行报告，包含设备统计、告警统计、巡检统计等信息

**请求参数：**

**Body 参数：**
```json
{
  "substationId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
  "startTime": "2026-06-01T00:00:00",
  "endTime": "2026-06-03T23:59:59",
  "includeAlarmDetails": true,
  "includePatrolDetails": true,
  "includeDeviceStatistics": true
}
```

**请求字段说明：**

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| substationId | Guid | 是 | 变电站ID |
| startTime | DateTime | 是 | 报告开始时间 |
| endTime | DateTime | 是 | 报告结束时间 |
| includeAlarmDetails | bool | 否 | 是否包含告警详情 |
| includePatrolDetails | bool | 否 | 是否包含巡检详情 |
| includeDeviceStatistics | bool | 否 | 是否包含设备统计 |

**响应格式：**
```json
{
  "substationId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
  "substationName": "110kV城东变电站",
  "reportPeriod": {
    "startTime": "2026-06-01T00:00:00",
    "endTime": "2026-06-03T23:59:59"
  },
  "summary": {
    "totalDevices": 150,
    "onlineDevices": 148,
    "offlineDevices": 2,
    "totalAlarms": 25,
    "processedAlarms": 20,
    "pendingAlarms": 5,
    "totalPatrols": 30,
    "completedPatrols": 28,
    "abnormalPatrols": 2
  },
  "deviceStatistics": {
    "normalDevices": 145,
    "warningDevices": 3,
    "alarmDevices": 2
  },
  "alarmStatistics": {
    "byLevel": [
      { "level": 1, "count": 5 },
      { "level": 2, "count": 10 },
      { "level": 3, "count": 8 },
      { "level": 4, "count": 2 }
    ],
    "byCategory": [
      { "category": "温度告警", "count": 12 },
      { "category": "声纹告警", "count": 8 },
      { "category": "湿度告警", "count": 5 }
    ]
  },
  "patrolStatistics": {
    "completionRate": 93.33,
    "abnormalRate": 6.67
  },
  "generatedAt": "2026-06-03T15:30:00"
}
```

**使用场景：**
- 生成日报、周报、月报
- 运营分析报告生成
- 数据导出和存档
- 管理层查看综合运营情况

**示例：**
```bash
POST /api/app/report/comprehensive-operation-report
Content-Type: application/json

{
  "substationId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
  "startTime": "2026-06-01T00:00:00",
  "endTime": "2026-06-03T23:59:59",
  "includeAlarmDetails": true,
  "includePatrolDetails": true,
  "includeDeviceStatistics": true
}
```

---

## 附录

### 通用响应格式

所有 API 的响应都遵循统一的格式约定：

**成功响应：**
- HTTP 状态码：200 OK（或 204 No Content）
- Content-Type：application/json

**错误响应：**
```json
{
  "error": {
    "code": "ERROR_CODE",
    "message": "错误描述信息",
    "details": "详细错误信息"
  }
}
```

### 认证方式

所有 API 都需要 JWT 认证，在请求头中携带：

```
Authorization: Bearer {access_token}
```

或通过查询字符串：

```
?access_token={access_token}
```

### 分页参数

对于支持分页的接口，使用以下标准参数：

| 参数名 | 类型 | 默认值 | 说明 |
|--------|------|--------|------|
| skipCount | int | 0 | 跳过的记录数 |
| maxResultCount | int | 10 | 最大返回记录数（通常不超过1000） |

### 枚举值参考

**告警状态 (AlarmStatusEnum)：**
- 1: 未处理
- 2: 已处理
- 3: 已忽略

**告警级别 (AlarmLevelEnum)：**
- 1: 提示
- 2: 一般
- 3: 严重
- 4: 紧急

**告警类别类型 (AlarmCategoryTypeEnum)：**
- 1: 设备告警
- 2: 环境告警
- 3: 安全告警
- 4: 运维告警

---

## 更新记录

| 版本 | 日期 | 说明 |
|------|------|------|
| v1.0.0 | 2026-06-03 | 初始版本，涵盖所有自定义 HTTP 端点 |

---

**文档维护：** 如有 API 变更，请及时更新本文档。
