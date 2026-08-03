---
title: MRSS 一期软件详细设计说明书
chapter: 第12章 api 设计
version: v1.0
---

# 第12章 api 设计

## 12.1 概述

api 模块是 MRSS 对外 HTTP、WebSocket 接口的处理层，负责路由、请求解析、认证上下文提取、参数校验、调用 service、结果序列化和错误响应。api 不是 Controller，不包含设备执行队列、任务状态机或厂商协议。

Communication 负责连接和传输；api 负责业务接口；service 负责业务用例；Controller 负责内部控制逻辑。任何接口都不能绕过 service 直接写数据库或调用 Access。

## 12.2 设计目标

1. 统一 URL、方法、请求体、响应体、错误码和分页格式。
2. 支持 HTTP 查询、配置、任务、控制、历史、告警和审计接口。
3. 支持 WebSocket 实时状态、任务、告警和操作结果推送。
4. 每次请求生成或传递 trace_id。
5. 每个写操作使用权限、范围、幂等和审计策略。
6. 控制命令返回 accepted/operation_id，不伪造同步完成。
7. 明确停止、急停和普通命令的优先级。
8. 保护 Token、密码、图片路径和内部错误信息。
9. 保证仿真、真实和混合模式下接口不变化。
10. 为客户端版本兼容和后续扩展保留字段。

## 12.3 接口分层

| 层 | 职责 |
| --- | --- |
| HTTP Router | URL 和 HTTP 方法匹配 |
| Handler | 读取请求、调用 service、返回响应 |
| Request DTO | JSON 到业务命令的结构校验 |
| Auth Middleware | 构造 AuthContext |
| Idempotency Middleware | 提取和校验幂等键 |
| Response Mapper | ServiceResult 到 HTTP 响应 |
| WebSocket Handler | 认证、订阅、消息路由 |
| Communication | Socket、连接、背压和心跳 |

Handler 的函数只做协议适配，不写业务判断。复杂判断必须进入 service。

## 12.4 通用请求头

| Header | 说明 |
| --- | --- |
| Authorization | Bearer Access Token |
| X-Request-Id | 客户端请求标识，可被服务端规范化 |
| X-Trace-Id | 链路追踪标识 |
| Idempotency-Key | 写操作幂等键 |
| If-Match | 资源版本控制 |
| Content-Type | application/json |

服务端生成 canonical trace_id。客户端提供的值只允许在格式合法时作为关联值。

## 12.5 通用响应

成功响应：

~~~json
{
  "code": "OK",
  "message": "success",
  "trace_id": "tr_xxx",
  "data": {}
}
~~~

异步操作响应：

~~~json
{
  "code": "ACCEPTED",
  "message": "operation accepted",
  "trace_id": "tr_xxx",
  "data": {
    "operation_id": "op_xxx",
    "status": "ACCEPTED"
  }
}
~~~

分页响应：

~~~json
{
  "code": "OK",
  "trace_id": "tr_xxx",
  "data": {
    "items": [],
    "page": 1,
    "page_size": 50,
    "total": 0,
    "has_more": false
  }
}
~~~

错误响应：

~~~json
{
  "code": "RESOURCE_BUSY",
  "message": "resource is busy",
  "trace_id": "tr_xxx",
  "details": {
    "resource_id": "ARM_R1"
  }
}
~~~

生产环境不返回堆栈、SQL、密码、Token、设备凭据和内部文件绝对路径。

## 12.6 URL 版本

一期使用 /api 前缀。版本兼容通过字段演进和服务端能力协商维护；出现不兼容变化时使用 /api/v2，而不是悄悄改变旧接口语义。

示例：

~~~text
/api/auth/login
/api/system/state
/api/devices
/api/arms/R1/movej
/api/cameras/HIK_FIX1/snapshot
/api/radiation/GAM1/sample
/api/tasks
/api/task-runs
/api/alarms
/api/audit/events
~~~

## 12.7 认证接口

### 12.7.1 登录

~~~text
POST /api/auth/login
permission: public
~~~

请求：

~~~json
{
  "username": "operator",
  "password": "********",
  "client_type": "web"
}
~~~

响应包含 access_token、refresh_token、expires_in、token_type、session_id 和用户概要。密码只在 TLS 连接内传输，并在处理后尽快从临时内存清除。

### 12.7.2 刷新

~~~text
POST /api/auth/refresh
permission: refresh token
~~~

旧 Refresh Token 使用后立即轮换。重放触发会话撤销。

### 12.7.3 退出

~~~text
POST /api/auth/logout
permission: authenticated
~~~

注销当前会话，并写入审计。

### 12.7.4 当前用户

~~~text
GET /api/auth/me
permission: authenticated
~~~

返回用户、角色、权限摘要和范围，不返回密码哈希、Token 或内部安全配置。

## 12.8 系统和健康接口

| 方法 | 路径 | 权限 | 说明 |
| --- | --- | --- | --- |
| GET | /api/system/state | system:read | 系统状态 |
| GET | /api/system/mode | system:read | 当前模式 |
| POST | /api/system/mode | system:mode | 请求模式切换 |
| GET | /api/system/health | health:read | 运行健康 |
| GET | /api/system/readiness | health:read | 是否可接收业务 |
| POST | /api/system/maintenance | system:maintenance | 进入维护 |
| GET | /api/system/config | system:config:read | 配置摘要 |
| PUT | /api/system/config | system:config | 更新配置 |

健康接口不得仅以 HTTP 200 表示所有设备正常。响应中区分 process、database、communication、device、safety 和 scheduler 状态。

## 12.9 设备查询接口

~~~text
GET /api/devices
GET /api/devices/{device_id}
GET /api/devices/{device_id}/history
GET /api/devices/{device_id}/capabilities
GET /api/devices/{device_id}/health
~~~

设备列表返回：

~~~json
{
  "device_id": "R1",
  "device_type": "realman_arm",
  "mode": "AUTO",
  "state": "IDLE",
  "connection": "CONNECTED",
  "permit": true,
  "owner": "task:123",
  "fault": null,
  "capabilities": ["MOVEJ", "JOG", "STOP"]
}
~~~

permit、owner 和 fault 是服务端状态，不接受客户端伪造。

## 12.10 机械臂控制接口

一期支持 MOVEJ、JOG、STOP 和轨迹/协同命令。

### 12.10.1 MOVEJ

~~~text
POST /api/arms/{arm_id}/movej
permission: device:control
~~~

请求主要字段：

~~~json
{
  "joints": [0, 0, 0, 0, 0, 0],
  "speed": 0.2,
  "acceleration": 0.2,
  "wait": false
}
~~~

服务端校验关节数量、范围、速度、加速度、模式、OWNER、资源、Permit 和设备状态。返回 operation_id，不保证设备已到位。

### 12.10.2 JOG

~~~text
POST /api/arms/{arm_id}/jog
permission: device:control
~~~

JOG 请求必须带方向、速度、持续时间或租约信息。客户端断线时，Controller 应按安全策略停止持续点动。

### 12.10.3 STOP

~~~text
POST /api/arms/{arm_id}/stop
permission: safety:stop 或 device:control
~~~

STOP 使用最高优先级通道，立即取消可取消命令并请求设备停止。即使普通 OWNER 已失效，安全停止仍可执行。接口成功表示 STOP 已被接受，不代表设备已经完全静止；最终结果通过 operation 查询或 WebSocket 推送。

## 12.11 相机接口

### 固定相机

~~~text
POST /api/cameras/HIK_FIX1/snapshot
permission: device:control
~~~

### PTZ 相机

~~~text
POST /api/cameras/HIK_PTZ1/ptz
POST /api/cameras/HIK_PTZ1/preset
POST /api/cameras/HIK_PTZ1/snapshot
permission: device:control
~~~

拍照响应包含 operation_id、image_id、image_path 或受控资源 URL、snapshot_result 和时间。真实模式下图片文件由受控媒体存储管理；数据库只保存资源元数据和引用。

## 12.12 辐射接口

~~~text
GET /api/radiation/GAM1
POST /api/radiation/GAM1/sample
GET /api/radiation/GAM1/history
permission: sensor:read 或 sensor:sample
~~~

采样结果必须携带 value、unit、quality、sample_time、alarm_level 和 operation_id。单位由设备配置和数据模型确定，例如 uSv/h 或 CPS，不在 Handler 中硬编码换算。

## 12.13 任务接口

| 方法 | 路径 | 说明 |
| --- | --- | --- |
| GET | /api/tasks | 查询任务定义 |
| POST | /api/tasks | 创建任务 |
| GET | /api/tasks/{id} | 查询任务 |
| PUT | /api/tasks/{id} | 修改任务 |
| DELETE | /api/tasks/{id} | 删除任务 |
| POST | /api/tasks/{id}/enable | 启用 |
| POST | /api/tasks/{id}/disable | 停用 |
| POST | /api/tasks/{id}/start | 启动运行 |
| POST | /api/task-runs/{id}/pause | 暂停 |
| POST | /api/task-runs/{id}/resume | 恢复 |
| POST | /api/task-runs/{id}/cancel | 取消 |
| GET | /api/task-runs/{id}/events | 运行事件 |

创建任务采用嵌套 points/actions，服务端返回 definition_version。启动时带 idempotency key，并保存启动时定义快照。

## 12.14 操作结果接口

~~~text
GET /api/operations/{operation_id}
POST /api/operations/{operation_id}/cancel
~~~

操作状态：

~~~text
ACCEPTED -> RUNNING -> SUCCEEDED
ACCEPTED -> REJECTED
RUNNING -> FAILED
RUNNING -> UNKNOWN
RUNNING -> CANCELLED
~~~

UNKNOWN 表示设备或网络结果无法确认，客户端不得把它显示为成功。恢复服务可以查询设备状态或执行补偿动作。

## 12.15 告警接口

~~~text
GET /api/alarms
GET /api/alarms/{alarm_id}
POST /api/alarms/{alarm_id}/ack
POST /api/alarms/{alarm_id}/clear
GET /api/alarms/history
~~~

告警确认不等于告警恢复。ack 记录操作者和时间；clear 只能由告警条件恢复或具有明确权限的维护操作完成。

## 12.16 历史与审计接口

历史接口统一支持：

~~~text
from
to
page
page_size
sort
device_id
task_id
level
status
trace_id
~~~

服务端限制最大时间范围和 page_size。审计接口不允许普通运维用户查看密码、Token、网络凭据或完整内部异常。

## 12.17 WebSocket 协议

连接：

~~~text
/ws
~~~

消息统一格式：

~~~json
{
  "type": "subscribe",
  "request_id": "req_xxx",
  "payload": {
    "topics": ["device.state", "task.event", "alarm.changed"]
  }
}
~~~

服务端事件：

~~~json
{
  "type": "event",
  "event_id": "evt_xxx",
  "topic": "device.state",
  "seq": 100,
  "trace_id": "tr_xxx",
  "payload": {}
}
~~~

主题权限：

| 主题 | 所需权限 |
| --- | --- |
| device.state | device:read |
| task.event | task:read |
| operation.result | operation:read |
| alarm.changed | alarm:read |
| audit.event | audit:read |
| safety.changed | safety:read |

客户端断线重连后使用 last_seq 请求补发；补发窗口外返回 RESYNC_REQUIRED，客户端重新执行查询接口。

## 12.18 Handler 设计

~~~text
class DeviceHandler {
public:
    HttpResponse list(const HttpRequest&, AuthContext);
    HttpResponse get(const HttpRequest&, AuthContext);
    HttpResponse movej(const HttpRequest&, AuthContext);
    HttpResponse jog(const HttpRequest&, AuthContext);
    HttpResponse stop(const HttpRequest&, AuthContext);
private:
    DeviceService& service_;
    RequestDecoder& decoder_;
    ResponseMapper& mapper_;
};
~~~

Handler 不保存请求级状态，不跨请求缓存 AuthContext，不直接持有数据库会话。依赖通过 AppContext 注入。

## 12.19 参数校验

校验分三层：

1. 结构校验：JSON 字段类型、必填字段和长度；
2. 语义校验：数值范围、枚举、设备能力和版本；
3. 执行前校验：模式、OWNER、资源、Permit、故障和急停。

结构错误返回 400；权限错误返回 403；运行状态拒绝返回业务错误码。不能仅依靠前端校验。

## 12.20 限流与背压

- 登录、刷新、历史查询和 WebSocket 订阅使用独立限流桶；
- 普通页面事件允许丢弃旧状态更新，但不丢失告警变化和操作最终结果；
- STOP、急停和故障事件使用高优先级队列；
- 单个连接的发送队列有上限，超过上限则断开并要求重新同步；
- 服务器不因为一个慢客户端阻塞设备控制线程。

## 12.21 错误与 HTTP 映射

| 业务错误 | HTTP |
| --- | --- |
| VALIDATION_ERROR | 400 |
| AUTH_TOKEN_INVALID | 401 |
| AUTH_PERMISSION_DENIED | 403 |
| RESOURCE_NOT_FOUND | 404 |
| IDEMPOTENCY_CONFLICT | 409 |
| RESOURCE_BUSY | 409 |
| RATE_LIMITED | 429 |
| DEVICE_OFFLINE | 503 |
| SAFETY_NOT_PERMITTED | 409 |
| ESTOP_LATCHED | 409 |
| INTERNAL_ERROR | 500 |
| SERVICE_UNAVAILABLE | 503 |

## 12.22 安全要求

1. 生产环境只允许 TLS。
2. 禁止 Token 出现在 URL 查询参数。
3. CORS、Origin 和 WebSocket 来源按部署配置白名单。
4. JSON 请求限制大小和嵌套深度。
5. 图片资源 URL 使用短期授权或内部代理。
6. 审计和日志对密码、Token、凭据脱敏。
7. 设备 ID、排序字段和过滤字段使用白名单。
8. 文件路径由服务端生成，不接受任意路径。
9. 所有控制接口必须有权限、范围和审计。
10. 安全接口禁止缓存。

## 12.23 测试设计

- 路由和方法测试；
- JSON Schema 和边界测试；
- 认证、权限和范围测试；
- 幂等和版本控制测试；
- WebSocket 认证、订阅、断线补发测试；
- 慢客户端、队列满和连接断开测试；
- 设备离线、Permit=0、急停锁存测试；
- 错误脱敏和安全扫描；
- 端到端 Web -> api -> service -> Controller -> Device -> 事件测试。

## 12.24 验收追踪

1. 所有一期核心功能可通过统一 API 调用。
2. api 不直接依赖数据库和厂商 SDK。
3. Controller 与 API Handler 命名和职责不混淆。
4. MOVEJ/JOG/STOP、拍照和辐射采样均有 operation_id。
5. WebSocket 事件与 HTTP 查询最终一致。
6. 访问控制、幂等、审计、限流和错误脱敏可验证。
7. 急停和 STOP 不被普通队列阻塞。
8. 仿真切换到真实设备时 URL 和报文语义不改变。

## 12.25 本章小结

本章完成了 api 模块的接口设计，定义了 HTTP、WebSocket、认证、设备、机械臂、相机、GAM1、任务、操作、告警、历史和审计接口，并明确了统一响应、错误、鉴权、幂等、背压和安全要求。api 只做协议适配，业务逻辑由 service 承担，内部控制由 Controller 承担。