---
title: MRSS 一期软件详细设计说明书
chapter: 第11章 service 设计
version: v1.0
---

# 第11章 service 设计

## 11.1 概述

service 模块承载 MRSS 的应用业务用例，是 api、WebSocket 事件和 Controller/Device 之间的业务编排层。service 将请求转换为业务命令，协调 db、auth、Controller、Device、Access、告警和审计，保证一个业务用例具有清晰的事务边界和状态转换。

service 不承担 HTTP 路由、不解析 WebSocket 帧、不直接拼 SQL、不直接调用厂商 SDK，也不替代 Controller 的设备执行队列。它负责“业务上要做什么以及是否允许开始”，Controller 负责“如何按设备和协同规则执行”。

一期 service 使用 PostgreSQL + SOCI 封装的 Repository；设备通过统一 Device/Access 接口访问，仿真、真实和混合模式共用业务流程。

## 11.2 设计目标

1. 以用例为中心组织业务，而不是以 HTTP 接口为中心堆叠逻辑。
2. 明确每个用例的输入、权限、前置条件、事务和结果。
3. 统一处理任务、设备、用户、告警、审计和系统配置业务。
4. 对同一业务操作提供 operation_id 和幂等语义。
5. 将数据库事务与设备网络调用分离，避免长事务包住设备动作。
6. 将设备运行结果、持久化结果和审计结果分别建模。
7. 保证 R1、R2、两类海康相机和 GAM1 的故障隔离。
8. 在数据库、设备或消息推送失败时返回可诊断结果。
9. 支持同步查询和异步执行，不让 HTTP 请求长期占用设备线程。
10. 为后续拆分进程或服务保留稳定接口。

## 11.3 service 分层

~~~mermaid
flowchart TB
    API[api handler]
    APP[application service]
    DOMAIN[domain policy]
    PORTS[ports]
    REPO[db repositories]
    CTRL[Controller]
    EVENT[events]
    AUDIT[audit]

    API --> APP
    APP --> DOMAIN
    APP --> PORTS
    PORTS --> REPO
    PORTS --> CTRL
    APP --> EVENT
    APP --> AUDIT
~~~

### 11.3.1 Application Service

负责一个完整业务用例的流程编排、权限入口、事务边界、幂等和结果转换。

### 11.3.2 Domain Policy

负责业务规则，例如任务定义校验、设备能力校验、优先级、点位动作合法性和任务状态转换。

### 11.3.3 Port

定义对 db、Controller、事件总线、时钟和审计的抽象接口，service 不依赖具体实现。

### 11.3.4 Event Publisher

发布设备状态、任务进度、告警和审计摘要事件。事件发布失败不能回滚已经完成的安全停止动作。

## 11.4 核心 Service 列表

| Service | 主要职责 |
| --- | --- |
| AuthService | 认证、授权、会话 |
| UserService | 用户和角色管理 |
| DeviceService | 设备注册、状态查询、设备控制入口 |
| TaskService | 任务定义、任务生命周期 |
| ExecutionService | 任务运行、点位、动作、结果 |
| AlarmService | 告警生成、确认、恢复 |
| AuditService | 审计事实写入和查询 |
| ConfigService | 系统配置、设备配置和版本 |
| ResourceService | OWNER、资源锁和租约 |
| SystemService | 系统状态、模式、健康和维护 |
| HistoryService | 设备、任务、传感器和告警历史 |
| MediaService | 图片资源元数据和受控引用 |

## 11.5 ServiceContext

~~~text
ServiceContext {
    AuthContext auth
    TraceContext trace
    RequestMeta request
    Clock& clock
    CancellationToken cancellation
}
~~~

所有公开 service 函数接收 ServiceContext 或其等价对象。内部定时任务没有用户身份时，必须使用显式的 SystemActor，例如 scheduler、watchdog、recovery，并写入审计或系统事件。

## 11.6 统一业务结果

~~~text
ServiceResult<T> {
    bool accepted
    OperationId operation_id
    ResultStatus status
    T data
    Error error
    TraceId trace_id
}
~~~

ResultStatus 至少包括：

- ACCEPTED：已接受异步执行；
- RUNNING：正在执行；
- SUCCEEDED：已完成并确认成功；
- FAILED：已确认失败；
- REJECTED：前置条件或权限不满足；
- UNKNOWN：外部结果无法确认；
- CANCELLED：已取消；
- DEGRADED：完成但存在非关键持久化或推送问题。

业务层不能把“请求已进入队列”直接返回为 SUCCEEDED。

## 11.7 UserService

### 职责

- 用户创建、修改、禁用、解锁；
- 角色和范围分配；
- 用户查询和分页；
- 管理员操作审计；
- 调用 auth 的密码与 Token 版本策略。

### 接口

~~~text
ServiceResult<UserProfile> createUser(ServiceContext&, CreateUserCommand)
ServiceResult<UserProfile> updateUser(ServiceContext&, UpdateUserCommand)
ServiceResult<void> disableUser(ServiceContext&, UserId, Reason)
ServiceResult<void> unlockUser(ServiceContext&, UserId)
ServiceResult<void> changePassword(ServiceContext&, ChangePasswordCommand)
Page<UserProfile> listUsers(ServiceContext&, UserQuery)
~~~

UserService 不直接生成 JWT；密码变化后通知 AuthService 撤销会话。

## 11.8 DeviceService

DeviceService 是设备业务入口，负责：

1. 检查权限和设备范围；
2. 获取设备快照；
3. 校验业务层的操作类型；
4. 创建 operation_id；
5. 调用 ResourceService/Controller；
6. 记录命令和审计；
7. 返回已接受或最终结果。

接口：

~~~text
ServiceResult<DeviceSnapshot> getDevice(ServiceContext&, DeviceId)
Page<DeviceSnapshot> listDevices(ServiceContext&, DeviceQuery)
ServiceResult<OperationAccepted> submitMoveJ(ServiceContext&, MoveJCommand)
ServiceResult<OperationAccepted> submitJog(ServiceContext&, JogCommand)
ServiceResult<OperationAccepted> stop(ServiceContext&, StopCommand)
ServiceResult<OperationAccepted> snapshot(ServiceContext&, SnapshotCommand)
ServiceResult<OperationAccepted> sample(ServiceContext&, SampleCommand)
~~~

STOP 和急停调用使用独立高优先级端口。它们仍记录审计，但不能被普通业务队列、Token 刷新或历史写入阻塞。

## 11.9 TaskService

TaskService 管理任务定义，不负责执行线程。

任务组成：

~~~text
Task
  -> TaskPoint[]
       -> PointAction[]
  -> schedule
  -> priority
  -> loop_policy
  -> resource_policy
  -> enabled
~~~

创建任务的事务边界包括 task、task_point、point_action 和资源声明。设备网络调用不在该事务内。

接口：

~~~text
ServiceResult<Task> createTask(ServiceContext&, CreateTaskCommand)
ServiceResult<Task> updateTask(ServiceContext&, UpdateTaskCommand)
ServiceResult<void> enableTask(ServiceContext&, TaskId)
ServiceResult<void> disableTask(ServiceContext&, TaskId)
ServiceResult<void> deleteTask(ServiceContext&, TaskId)
Page<Task> listTasks(ServiceContext&, TaskQuery)
~~~

任务定义更新必须有版本号。已开始运行的 TaskRun 使用启动时的 definition_version，不因后续编辑而改变。

## 11.10 ExecutionService

ExecutionService 负责将任务定义转换为执行计划，并提交给 Controller。

运行状态：

~~~text
CREATED -> VALIDATING -> WAITING_RESOURCE -> RUNNING
RUNNING -> PAUSED
RUNNING -> SUCCEEDED
RUNNING -> FAILED
RUNNING -> CANCELLED
RUNNING -> UNKNOWN
PAUSED -> RUNNING
PAUSED -> CANCELLED
~~~

接口：

~~~text
ServiceResult<TaskRun> startTask(ServiceContext&, StartTaskCommand)
ServiceResult<void> pauseTask(ServiceContext&, TaskRunId)
ServiceResult<void> resumeTask(ServiceContext&, TaskRunId)
ServiceResult<void> cancelTask(ServiceContext&, CancelTaskCommand)
ServiceResult<TaskRun> getTaskRun(ServiceContext&, TaskRunId)
Page<TaskEvent> listTaskEvents(ServiceContext&, TaskRunQuery)
~~~

startTask 的执行顺序：

1. 检查任务存在、启用和版本。
2. 检查用户权限和任务范围。
3. 校验点位、动作和设备能力。
4. 生成 TaskRun 和 operation_id。
5. 请求 ResourceService 原子申请资源。
6. 创建 Controller 执行上下文。
7. 提交到调度队列。
8. 发布 TASK_ACCEPTED。
9. 后续由事件回调更新 TaskRun。

## 11.11 AlarmService

AlarmService 统一处理系统、设备、传感器和任务告警。

告警事实分为：

- AlarmDefinition：告警定义和阈值；
- AlarmState：当前是否激活、级别和确认状态；
- AlarmEvent：产生、升级、确认、恢复和关闭事件。

接口：

~~~text
void raiseAlarm(AlarmRaiseCommand)
void updateAlarmMetric(AlarmMetricData)
ServiceResult<void> acknowledge(ServiceContext&, AlarmId, AckCommand)
ServiceResult<void> clear(ServiceContext&, AlarmId, ClearCommand)
Page<AlarmRecord> listAlarms(ServiceContext&, AlarmQuery)
~~~

告警去重使用 alarm_key + device_id + source。重复采样只更新 current_value 和 last_seen，不无限创建新记录；级别变化必须创建事件。

## 11.12 AuditService

AuditService 对控制、配置、认证、急停、模式、资源、告警和任务操作写入不可抵赖的审计事实。

接口：

~~~text
void record(AuditEvent event)
Page<AuditEvent> query(ServiceContext&, AuditQuery)
Result<void> flush(CancellationToken)
~~~

AuditService 提供高优先级同步写入和普通事件缓冲两类策略：

- 安全复位、急停、权限变更、用户管理：优先同步确认；
- 设备状态、周期采样、普通页面访问：允许批量缓冲；
- 审计数据库不可用：写入有界本地缓冲并告警；
- 缓冲满：按保留策略处理，不能静默丢弃安全事件。

## 11.13 ResourceService

ResourceService 管理 OWNER 和资源锁，具体规则在第16章展开。service 只通过端口调用：

~~~text
Result<Lease> acquire(ResourceRequest)
Result<void> renew(LeaseId)
Result<void> release(LeaseId)
Result<void> releaseAll(OwnerId)
Result<LeaseSnapshot> inspect(ResourceId)
~~~

任务申请双臂协同资源时必须原子申请 ARM_R1、ARM_R2 和工作空间资源，任一失败则不启动。

## 11.14 ConfigService

配置分为：

1. 启动配置；
2. 设备静态配置；
3. 安全阈值；
4. 任务和调度配置；
5. 用户可编辑运行参数；
6. 只读运行快照。

修改配置的流程：

~~~mermaid
sequenceDiagram
    participant U as User
    participant A as api
    participant C as ConfigService
    participant V as Validator
    participant D as db
    participant E as EventBus

    U->>A: update config
    A->>C: command
    C->>V: validate
    C->>D: begin transaction
    C->>D: save versioned config
    C->>D: audit config change
    C->>D: commit
    C->>E: CONFIG_CHANGED
    E-->>C: consumers reload
~~~

涉及安全许可、急停阈值和设备协议的配置必须在运行前校验，不能让任意配置立即破坏安全约束。

## 11.15 SystemService

SystemService 管理系统级状态和维护操作：

~~~text
SystemState getState()
SystemMode getMode()
ServiceResult<void> requestMode(ServiceContext&, ModeRequest)
ServiceResult<void> enterMaintenance(ServiceContext&)
ServiceResult<void> leaveMaintenance(ServiceContext&)
HealthSnapshot health()
ReadinessSnapshot readiness()
~~~

SystemService 不直接改变设备状态。模式切换需要与 ResourceService、Controller 和 SafetyService 协作，完整流程在第16、17章定义。

## 11.16 事务与设备调用

禁止以下模式：

~~~text
BEGIN DATABASE TRANSACTION
  call camera SDK
  wait network response
  call arm
COMMIT
~~~

推荐模式：

1. 在短事务内校验并创建 operation/task event；
2. 提交事务；
3. 调用 Controller/Device；
4. 根据设备回调启动新的短事务更新结果；
5. 若结果未知，保存 UNKNOWN 并由恢复任务查询确认。

这样可以避免设备网络延迟占用数据库连接和锁。

## 11.17 幂等设计

所有控制类写操作携带 idempotency_key，格式由客户端生成并由服务端与 user/session/route 绑定。相同 key 重复请求：

- 如果原操作已成功，返回原结果；
- 如果仍运行，返回原 operation_id；
- 如果已失败，返回原失败结果；
- 如果请求参数不一致，返回 IDEMPOTENCY_CONFLICT；
- 保留窗口过后不保证再次命中，但新的操作仍需使用新 key。

STOP 可以重复发送，系统按安全命令语义处理，不因重复而返回普通冲突。

## 11.18 事件处理

service 发布以下事件：

~~~text
AUTH_CHANGED
DEVICE_STATE_CHANGED
OPERATION_ACCEPTED
OPERATION_RESULT
TASK_STATE_CHANGED
TASK_POINT_CHANGED
ALARM_CHANGED
RESOURCE_CHANGED
CONFIG_CHANGED
SYSTEM_MODE_CHANGED
SAFETY_STATE_CHANGED
~~~

事件至少包含 event_id、event_type、occurred_at、trace_id、source、aggregate_type、aggregate_id、version 和 payload。事件消费者必须允许重复事件，并使用 event_id 去重。

## 11.19 异常映射

| 来源 | service 处理 |
| --- | --- |
| 参数错误 | VALIDATION_ERROR |
| 权限不足 | AUTH_PERMISSION_DENIED |
| 设备不存在 | DEVICE_NOT_FOUND |
| 设备离线 | DEVICE_OFFLINE |
| 资源冲突 | RESOURCE_BUSY |
| 模式不允许 | MODE_REJECTED |
| Permit 不允许 | SAFETY_NOT_PERMITTED |
| 急停锁存 | ESTOP_LATCHED |
| Controller 拒绝 | CONTROL_REJECTED |
| 设备结果失败 | DEVICE_COMMAND_FAILED |
| 结果未知 | RESULT_UNKNOWN |
| 数据库失败 | PERSISTENCE_ERROR 或 DEGRADED |

不得把异常堆栈直接返回前端；内部日志携带 trace_id 和分类信息。

## 11.20 并发模型

- 查询 Service 可以在线程池并行执行；
- 同一 TaskRun 的状态更新必须按 task_run_id 串行化；
- 同一设备命令由 Controller 保证串行；
- 不同设备可以并行；
- 告警状态按 alarm_key 串行；
- 用户和配置写操作按实体版本控制；
- STOP 和急停不排队等待普通业务操作。

## 11.21 测试设计

1. 每个 Service 的用例单元测试；
2. 使用 Fake Repository、Fake Controller 和 Fake Clock；
3. 验证权限、范围、幂等、事务和事件；
4. 验证设备失败不污染其他设备；
5. 验证任务定义修改不影响已运行版本；
6. 验证结果 UNKNOWN 的恢复流程；
7. 验证数据库断线时安全路径仍工作；
8. 验证重复事件和重复请求不会生成重复事实；
9. 验证所有控制操作写入审计；
10. 验证 WebSocket 事件与查询结果最终一致。

## 11.22 本章小结

本章定义了 MRSS service 层的业务边界和主要用例，明确了 User、Device、Task、Execution、Alarm、Audit、Config、Resource、System 等 Service 的职责、接口、事务、幂等、事件、并发和异常语义。service 负责业务编排，Controller 负责设备执行，db 负责持久化，三者通过稳定端口协作。