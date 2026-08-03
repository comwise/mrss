---
title: MRSS 一期软件详细设计说明书
chapter: 第13章 core 设计
version: v1.0
---

# 第13章 core 设计

## 13.1 概述

core 模块为 Backend 提供公共基础能力，是整个系统的基础支撑层。它不承载用户、任务、设备、告警等具体业务，而是提供类型、错误、时间、配置、线程、队列、事件、日志、追踪、序列化、校验和生命周期等通用能力。

core 必须保持低依赖、稳定和可测试。业务模块可以依赖 core，core 不得反向依赖 api、service、Controller、Device、Access、db 或具体厂商 SDK。

## 13.2 设计目标

1. 统一系统 ID、时间、结果、错误和取消语义。
2. 提供边界明确的线程、执行器、定时器和有界队列。
3. 支持同步、异步和事件驱动模块使用相同基础模型。
4. 不引入隐藏全局状态，依赖通过构造函数或 Context 注入。
5. 在停止流程中优先保证安全命令和资源释放。
6. 对配置、日志、指标和追踪进行统一管理。
7. 让单元测试可以替换时钟、随机数、线程和外部端口。
8. 不让公共工具承担业务判断和设备协议解析。

## 13.3 依赖约束

~~~mermaid
flowchart TB
    CORE[core]
    LIB[lib]
    OS[OS and standard library]
    API[api]
    SERVICE[service]
    DEVICE[Device]
    DB[db]

    CORE --> LIB
    CORE --> OS
    API --> CORE
    SERVICE --> CORE
    DEVICE --> CORE
    DB --> CORE
~~~

core 允许依赖 C++ 标准库、yaml-cpp、日志库、UUID/加密基础库等经过封装的第三方库。业务代码不得直接暴露第三方库类型。

## 13.4 命名空间和目录

~~~text
core/
  types/
  result/
  error/
  time/
  id/
  config/
  concurrency/
  event/
  logging/
  tracing/
  metrics/
  serialization/
  validation/
  lifecycle/
  filesystem/
~~~

命名空间建议为 mrss::core。第三方库统一位于 lib，core 通过 Adapter 封装。

## 13.5 基础类型

### 13.5.1 ID 类型

不使用裸 integer 表示所有业务 ID。至少定义：

~~~text
UserId
RoleId
DeviceId
TaskId
TaskRunId
OperationId
AlarmId
AuditId
ResourceId
SessionId
TraceId
EventId
ImageId
~~~

ID 具有不可混淆的强类型包装和字符串序列化能力。

### 13.5.2 时间类型

内部使用 std::chrono。对外统一 UTC 语义：

~~~text
using TimePoint = std::chrono::time_point<SystemClock>;
using Duration = std::chrono::milliseconds;
using TimestampMs = int64_t;
~~~

所有定时逻辑使用 MonotonicClock；业务事件时间使用 SystemClock。不能用系统时间回拨影响超时和租约。

### 13.5.3 数值和单位

测量值使用 Value + Unit + Quality：

~~~text
MetricValue {
    double value
    std::string unit
    Quality quality
    TimePoint measured_at
}
~~~

core 不定义 uSv/h、CPS、ppm 等业务阈值，只提供单位字段和格式校验。

## 13.6 Result 和 Error

### 13.6.1 Result

~~~text
template<class T>
class Result {
    bool has_value() const;
    const T& value() const;
    const Error& error() const;
};
~~~

对于无返回值使用 Result<void>。禁止用异常作为正常业务分支。

### 13.6.2 Error

~~~text
Error {
    ErrorCode code
    ErrorCategory category
    std::string message
    std::string trace_id
    std::string operation_id
    bool retryable
    std::map<std::string, std::string> attributes
}
~~~

message 不得包含密码、Token、连接串和敏感设备凭据。底层异常转换时保留 cause 摘要和原始分类，不把堆栈返回 API。

错误分类：

- VALIDATION；
- AUTHENTICATION；
- AUTHORIZATION；
- STATE；
- RESOURCE；
- DEVICE；
- COMMUNICATION；
- PERSISTENCE；
- SAFETY；
- INTERNAL；
- CANCELLATION；
- TIMEOUT。

## 13.7 配置系统

### 13.7.1 ConfigLoader

职责：

- 从 YAML 文件和受控环境变量加载配置；
- 合并默认值；
- 校验类型、范围和必填项；
- 生成不可变 ConfigSnapshot；
- 支持版本和热更新通知。

接口：

~~~text
Result<ConfigSnapshot> load(const ConfigSource&)
Result<ConfigSnapshot> reload(const ConfigSource&)
Result<void> validate(const ConfigSnapshot&)
~~~

秘密字段只允许通过环境变量或密钥引用注入，ConfigSnapshot 的日志输出必须自动脱敏。

### 13.7.2 配置分层

~~~text
defaults
  -> deployment file
  -> environment overrides
  -> secret references
  -> runtime editable config
~~~

运行时可编辑配置必须通过 ConfigService 版本化保存。设备安全参数变化前要执行校验和审计。

## 13.8 依赖注入

Backend 使用 AppContext 保存已构造的基础依赖：

~~~text
AppContext {
    ConfigSnapshot config
    Clock& clock
    Logger& logger
    MetricsRegistry& metrics
    TraceProvider& tracing
    ExecutorRegistry& executors
    EventBus& event_bus
    CancellationSource shutdown
}
~~~

业务对象不读取全局单例。单例只允许用于无状态常量或进程级不可变注册表，且必须在架构评审中说明。

## 13.9 执行器

### 13.9.1 Executor

~~~text
class Executor {
public:
    TaskHandle submit(TaskPriority, MoveOnlyFunction<void()>);
    bool trySubmit(TaskPriority, MoveOnlyFunction<void()>);
    void stop(StopMode);
    ExecutorSnapshot snapshot() const;
};
~~~

线程池按用途隔离：

| 执行器 | 内容 |
| --- | --- |
| control | Controller 和高优先级停止 |
| communication | HTTP、WebSocket、TCP、UDP |
| persistence | db、审计、Outbox |
| scheduler | 任务调度和定时 |
| maintenance | 重连、清理、指标和备份 |
| callback | 轻量事件回调 |

控制执行器不得被历史查询和媒体索引占满。STOP 使用高优先级通道。

### 13.9.2 优先级

~~~text
EMERGENCY_STOP
SAFETY_STOP
DEVICE_STOP
CONTROL
TASK
EVENT
PERSISTENCE
MAINTENANCE
~~~

优先级只决定本模块排队顺序，不能绕过设备自身保护。

## 13.10 BoundedQueue

所有跨线程队列必须有容量上限、满策略和指标。

~~~text
BoundedQueue<T> {
    push(item, QueuePriority)
    try_push(item, QueuePriority)
    pop(WaitToken)
    close()
    size()
    dropped()
}
~~~

满策略：

- 急停和安全事件：不得丢弃，必要时走独立通道；
- 操作最终结果：保留；
- 告警状态变化：保留；
- 高频设备状态：允许合并或丢弃旧值；
- 普通调试指标：允许采样丢弃；
- 任何丢弃必须有计数器和告警。

## 13.11 TimerService

TimerService 使用单调时钟，支持一次性和周期任务。

~~~text
TimerId scheduleAfter(Duration, TimerCallback)
TimerId scheduleEvery(Duration, TimerCallback)
Result<void> cancel(TimerId)
Result<void> reschedule(TimerId, Duration)
~~~

回调不在 TimerService 内执行复杂业务，而是投递到指定 Executor。停止时先停止产生新回调，再等待已提交任务或按策略取消。

## 13.12 EventBus

EventBus 传递进程内事件，不保证跨进程持久化。需要持久化的业务事实由 service + db 写入。

~~~text
Subscription subscribe(EventType, EventHandler, SubscriptionOptions)
Result<void> publish(Event event)
Result<void> unsubscribe(SubscriptionId)
~~~

事件支持同步、异步和有界队列模式。事件处理器必须幂等，不能在回调内无限递归发布同类事件。

## 13.13 Cancellation

取消分为：

- 用户取消；
- 任务取消；
- 模式切换取消；
- 超时取消；
- 安全停止；
- 进程关闭。

~~~text
CancellationToken {
    bool isCancellationRequested() const;
    void throwIfCancellationRequested() const;
}
~~~

安全停止不能依赖普通业务取消标志，必须有直接到 Controller/Access 的停止信号。

## 13.14 生命周期组件

每个可管理模块实现：

~~~text
class Component {
public:
    virtual Result<void> initialize(const AppContext&) = 0;
    virtual Result<void> start() = 0;
    virtual void requestStop(StopReason) = 0;
    virtual Result<void> stop(Duration timeout) = 0;
    virtual ComponentState state() const = 0;
};
~~~

状态：

~~~text
CREATED -> INITIALIZING -> INITIALIZED -> STARTING -> RUNNING
RUNNING -> STOP_REQUESTED -> STOPPING -> STOPPED
INITIALIZING -> FAILED
STARTING -> FAILED
STOPPING -> FAILED
~~~

组件必须允许重复查询状态，不能在析构函数中执行不可控的长时间网络操作。

## 13.15 日志

Logger 提供结构化日志：

~~~text
log(level,
    event_name,
    trace_id,
    operation_id,
    component,
    attributes)
~~~

日志级别：

- TRACE：开发调试；
- DEBUG：诊断；
- INFO：状态和生命周期；
- WARN：可恢复异常；
- ERROR：业务或组件失败；
- FATAL：无法继续运行。

日志字段包括时间、级别、组件、线程、trace_id、operation_id、device_id、task_id 和 error_code。密码、Token、连接串和原始图片路径按规则脱敏。

## 13.16 TraceContext

TraceContext 贯穿 API、service、Controller、Access、db 和事件。

~~~text
TraceContext {
    trace_id
    parent_span_id
    span_id
    sampled
}
~~~

跨线程提交任务时复制上下文；任务进入设备队列时保留 operation_id。第三方设备协议没有追踪字段时，在本地日志和数据库中关联。

## 13.17 Metrics

指标分为：

| 类别 | 示例 |
| --- | --- |
| 进程 | CPU、内存、线程数 |
| 通信 | 连接数、重连、延迟、丢包 |
| 控制 | 命令数、成功率、执行延迟 |
| 设备 | 在线、Permit、Fault、状态 |
| 任务 | 队列长度、等待、成功、失败 |
| 数据库 | 连接池、查询延迟、事务失败 |
| 安全 | 登录失败、权限拒绝、Token 撤销 |
| 队列 | 深度、满次数、丢弃数 |

指标名称和标签必须限制基数。user_id、trace_id 等高基数字段不能直接作为 Prometheus 标签。

## 13.18 序列化和校验

core 提供 JSON/YAML 边界适配，但不定义业务 DTO。序列化器必须：

1. 对未知字段按接口版本策略处理；
2. 检查字符串长度和嵌套深度；
3. 限制浮点 NaN 和 Infinity；
4. 明确枚举未知值；
5. 防止递归对象和超大数组；
6. 对二进制和路径字段进行安全校验。

## 13.19 文件与资源

core 的 FileService 只提供受控文件操作：

~~~text
Result<SafePath> resolveUnderRoot(root, relative)
Result<void> atomicWrite(path, bytes)
Result<FileInfo> stat(path)
Result<void> rotate(path, policy)
~~~

禁止将客户端传入的绝对路径直接用于读写。图片和日志目录由部署配置指定，文件写入采用临时文件 + fsync/rename 的原子策略。

## 13.20 安全工具边界

core 可以提供常量时间比较、随机数、敏感字段脱敏和安全字符串清理，但不实现用户授权和设备安全联锁。安全策略由 auth、service 和 SafetyService 使用。

## 13.21 关闭顺序

推荐顺序：

~~~text
停止接收新请求
  -> 标记 readiness=false
  -> 停止调度新任务
  -> 取消可取消业务任务
  -> 对运行设备发送安全停止
  -> 等待设备状态确认或超时
  -> flush 审计和关键事件
  -> 停止通信
  -> 停止数据库连接池
  -> 停止公共执行器
~~~

如果设备停止确认超时，记录风险并继续关闭进程，不能无限等待数据库或普通客户端。

## 13.22 测试设计

1. ID、时间和单位类型测试；
2. Result/Error 映射测试；
3. YAML 缺失、类型错误和脱敏测试；
4. Executor 优先级、停止和拒绝测试；
5. BoundedQueue 满策略和丢弃指标测试；
6. Timer 时间跳变和取消测试；
7. EventBus 重入和订阅生命周期测试；
8. Cancellation 并发测试；
9. Logger 敏感信息扫描；
10. FileService 路径穿越和原子写测试；
11. Component 生命周期和重复 stop 测试；
12. 线程泄漏、死锁和竞态检测。

## 13.23 设计检查项

- core 是否没有反向依赖业务模块；
- 是否没有隐藏全局数据库、设备或用户状态；
- 所有队列是否有容量和满策略；
- STOP 是否拥有独立的高优先级路径；
- 所有定时器是否使用单调时钟；
- 错误是否有稳定 code 和 trace_id；
- 日志和配置是否脱敏；
- 组件 stop 是否有超时；
- 高基数字段是否没有进入指标标签；
- 业务 DTO 是否没有污染 core。

## 13.24 本章小结

本章完成了 core 模块的公共基础设计，明确了类型、ID、时间、Result/Error、配置、依赖注入、执行器、有界队列、定时器、事件总线、取消、生命周期、日志、追踪、指标、序列化和文件安全边界。core 为整个 Backend 提供可复用基础能力，但不承载业务规则和设备协议。