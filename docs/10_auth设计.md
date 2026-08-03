---
title: MRSS 一期软件详细设计说明书
chapter: 第10章 auth 设计
version: v1.0
---

# 第10章 auth 设计

## 10.1 概述

auth 模块负责 MRSS 的身份认证、令牌管理、角色权限判定、会话状态、登录安全和认证审计。它位于 api、WebSocket 会话与 service 之间，为上层提供统一的身份上下文和授权决策。

auth 只回答两个问题：

1. 当前请求来自哪个已认证主体。
2. 该主体是否拥有执行当前操作所需的权限。

auth 不负责设备控制、任务调度、资源锁、数据库连接或厂商协议。权限判定通过后，具体业务规则仍由 service、Controller、Device 和安全联锁再次校验，不能把 JWT 中的角色当成设备安全许可。

一期支持本地账号认证。密码凭据存储在 PostgreSQL，密码只保存不可逆哈希；访问令牌采用 JWT，签名和密钥通过配置注入；WebSocket 在握手或首条认证消息中完成认证。后续可以扩展 LDAP、OAuth2 或统一身份平台，但上层业务接口不改变。

## 10.2 设计目标

1. 提供统一的登录、刷新、退出和当前用户查询接口。
2. 使用不可逆密码哈希，禁止明文或可逆加密保存密码。
3. 使用短时效 Access Token 降低令牌泄露风险。
4. 支持 Refresh Token 轮换、撤销和设备会话管理。
5. 支持超级管理员、系统管理员、运维经理、运维用户、只读用户五类基础角色。
6. 支持角色与权限解耦，允许后续增加细粒度权限。
7. 对 HTTP、WebSocket 和内部服务调用使用同一套 AuthContext。
8. 将登录、登出、权限拒绝、Token 撤销和安全配置变更写入审计。
9. 不把权限校验失败误报为设备故障或安全急停。
10. 在数据库暂时不可用时拒绝新登录，但不阻塞已进入安全停止流程的控制路径。
11. 支持仿真、真实和混合设备模式下相同的认证语义。
12. 所有认证失败结果使用统一错误码，不泄露用户是否存在、密码是否正确等敏感信息。

## 10.3 模块边界

| 模块 | auth 的职责 |
| --- | --- |
| api | 提取 HTTP Header、Cookie、WebSocket 握手信息并调用 auth |
| service | 根据 AuthContext 执行用户、任务和设备业务 |
| db | 保存用户、角色、权限、会话、Token 撤销和审计数据 |
| core | 提供时间、随机数、Hash、Result 和错误类型等公共能力 |
| Device/Controller | 在 AuthContext 通过后继续校验模式、OWNER、资源和安全许可 |
| Communication | 提供连接、握手、超时和消息传输，不决定业务权限 |
| auth | 认证、授权、会话和认证安全策略 |

auth 不得直接调用设备 Access，也不得在数据库中写入设备实时状态。

## 10.4 认证主体模型

### 10.4.1 User

User 表示可以登录系统的人员账号。

| 字段 | 说明 |
| --- | --- |
| user_id | 稳定用户标识 |
| username | 登录名，唯一且区分大小写规则固定 |
| display_name | 展示名称 |
| password_hash | 密码哈希 |
| status | ACTIVE、DISABLED、LOCKED、EXPIRED |
| failed_count | 连续失败次数 |
| locked_until | 临时锁定截止时间 |
| password_changed_at | 密码最近修改时间 |
| last_login_at | 最近成功登录时间 |
| created_at | 创建时间 |
| updated_at | 修改时间 |

### 10.4.2 Role

Role 是权限集合的业务名称。内置角色如下：

| 角色 | 定位 |
| --- | --- |
| SUPER_ADMIN | 系统最高管理角色，可管理账号、权限和系统配置 |
| SYSTEM_ADMIN | 系统配置、设备、通信和运行参数管理 |
| OPS_MANAGER | 任务、设备运行和运维操作管理 |
| OPS_USER | 按授权范围执行任务和设备操作 |
| READ_ONLY | 只读查看监控、历史、告警和审计摘要 |

角色不是安全许可。即使用户拥有设备操作权限，也必须同时满足 MODE、OWNER、RESOURCE、PERMIT、FAULT 和急停状态要求。

### 10.4.3 Permission

权限采用资源加动作表达：

~~~text
resource:action
user:read
user:write
role:manage
device:read
device:control
task:read
task:create
task:execute
task:cancel
alarm:read
alarm:ack
audit:read
system:config
safety:reset
safety:estop
~~~

权限名称使用小写 ASCII 和冒号分隔。未知权限默认拒绝。

### 10.4.4 Session

Session 表示某个用户在某台客户端上的登录会话，包含 session_id、user_id、client_id、登录时间、最后活动时间、IP 摘要、User-Agent 摘要、状态和撤销时间。IP 和 User-Agent 只用于安全审计，不能作为稳定身份标识。

## 10.5 AuthContext

AuthContext 是请求进入业务层时的认证上下文。

~~~text
AuthContext {
    subject_id
    username
    session_id
    token_id
    roles[]
    permissions[]
    issued_at
    expires_at
    client_type
    trace_id
}
~~~

AuthContext 不允许由客户端直接提交。它只能由 TokenVerifier 和 SessionStore 在服务端构造。内部调用如果使用服务账号，也必须明确 subject_type=SERVICE，并限制权限集合和有效期。

## 10.6 核心类设计

### 10.6.1 AuthService

职责：

- 登录和密码校验；
- 创建、刷新、撤销会话；
- 构造 AuthContext；
- 执行权限判断；
- 写入认证审计；
- 应用登录失败和账户锁定策略。

主要成员：

~~~text
UserRepository& user_repository_
RoleRepository& role_repository_
SessionRepository& session_repository_
AuditService& audit_service_
PasswordHasher& password_hasher_
TokenService& token_service_
AuthPolicy policy_
Clock& clock_
~~~

主要函数：

~~~text
Result<LoginResult> login(LoginRequest request, RequestMeta meta)
Result<TokenPair> refresh(RefreshRequest request, RequestMeta meta)
Result<void> logout(const AuthContext& context, LogoutReason reason)
Result<AuthContext> authenticate(const TokenCredential& credential,
                                 const RequestMeta& meta)
Result<void> require(const AuthContext& context,
                     std::string_view permission)
bool hasPermission(const AuthContext& context,
                   std::string_view permission) const
Result<UserProfile> currentUser(const AuthContext& context)
~~~

### 10.6.2 PasswordHasher

密码哈希参数必须由配置管理并在启动时打印非敏感摘要。默认使用 Argon2id；如果部署环境暂时不提供 Argon2id，允许使用经过评审的 bcrypt 兼容实现，但禁止使用 MD5、SHA-1、裸 SHA-256 或自定义算法。

~~~text
Result<PasswordHash> hash(std::string_view password)
Result<bool> verify(std::string_view password,
                    std::string_view encoded_hash)
bool needsRehash(std::string_view encoded_hash) const
~~~

密码策略至少包含最小长度、复杂度、历史密码数量、过期周期和禁止常见密码列表。

### 10.6.3 TokenService

TokenService 使用 jwt-cpp 或等价封装实现 JWT 编解码，但业务层不直接依赖 jwt-cpp 类型。

必需 Claims：

| Claim | 说明 |
| --- | --- |
| iss | 签发者 |
| sub | 用户或服务主体 |
| aud | 受众 |
| iat | 签发时间 |
| exp | 过期时间 |
| jti | 唯一令牌标识 |
| sid | 会话标识 |
| typ | access 或 refresh |
| roles | 登录时角色快照，可选 |
| ver | 用户 Token 版本 |

TokenService 主要函数：

~~~text
Result<AccessToken> issueAccessToken(const Session& session,
                                     const Subject& subject)
Result<RefreshToken> issueRefreshToken(const Session& session,
                                       const Subject& subject)
Result<VerifiedClaims> verify(std::string_view token,
                              TokenType expected_type)
Result<void> revoke(std::string_view token_id, TimePoint until)
~~~

Access Token 默认有效期较短，Refresh Token 有更长有效期但必须轮换。具体时间由部署配置决定，文档不把生产时长写死。

### 10.6.4 AuthorizationService

AuthorizationService 将权限判定与 Token 解析分离。

~~~text
Result<AuthorizationDecision> authorize(const AuthContext& context,
                                        const PermissionRequest& request)
bool match(std::string_view granted,
           std::string_view required) const
~~~

授权规则：

1. disabled、locked、expired 用户一律拒绝。
2. 过期 Token 一律拒绝。
3. 用户状态变化后，Token 版本不一致时拒绝。
4. 权限不存在时拒绝。
5. 资源范围不匹配时拒绝。
6. 管理员特权不绕过急停、硬件许可和设备故障保护。
7. 安全复位权限必须单独配置，不能由普通 device:control 推导。

## 10.7 密码登录流程

~~~mermaid
sequenceDiagram
    participant C as Client
    participant A as api
    participant S as AuthService
    participant U as UserRepository
    participant T as TokenService
    participant L as AuditService

    C->>A: POST /api/auth/login
    A->>S: login(credentials, meta)
    S->>U: findByUsername
    U-->>S: user
    S->>S: verify password and lock policy
    S->>T: issue access and refresh
    T-->>S: token pair
    S->>L: record login success
    S-->>A: LoginResult
    A-->>C: token pair and profile
~~~

无论用户名不存在还是密码错误，对外都返回 AUTH_INVALID_CREDENTIALS。内部审计可以保存分类后的失败原因，但不得保存密码、完整 Token 或认证 Header。

## 10.8 Refresh Token 轮换

Refresh Token 使用一次即失效。刷新过程必须在一个数据库事务中完成：

1. 校验签名、类型、有效期和会话状态。
2. 检查 refresh token 的 jti 是否已经使用或撤销。
3. 将旧 jti 标记为 USED。
4. 生成新的 refresh jti 和 access jti。
5. 更新会话最后刷新时间和 Token 版本。
6. 写入刷新审计。
7. 提交事务后返回新 Token。

如果同一个 Refresh Token 被重复使用，应撤销该会话下的全部 Token，并记录 REFRESH_REUSE_DETECTED。

## 10.9 WebSocket 认证

WebSocket 支持两种认证方式：

1. 握手 Header 携带 Authorization: Bearer Token。
2. 连接建立后，在规定时间内发送 auth 消息。

推荐使用握手 Header。首条消息认证模式用于无法设置 Header 的浏览器或特殊客户端。

连接状态：

~~~mermaid
stateDiagram-v2
    [*] --> CONNECTING
    CONNECTING --> AUTHENTICATING: handshake accepted
    AUTHENTICATING --> AUTHENTICATED: token valid
    AUTHENTICATING --> CLOSED: timeout or invalid
    AUTHENTICATED --> AUTHENTICATED: ping/pong and events
    AUTHENTICATED --> REAUTH_REQUIRED: token near expiry
    REAUTH_REQUIRED --> AUTHENTICATED: refresh token accepted
    AUTHENTICATED --> CLOSED: logout/revoke/error
    CLOSED --> [*]
~~~

未认证连接不得订阅设备状态、任务事件或告警事件。认证通过后还要针对每个订阅主题执行权限检查。

## 10.10 权限矩阵

| 操作 | SUPER_ADMIN | SYSTEM_ADMIN | OPS_MANAGER | OPS_USER | READ_ONLY |
| --- | --- | --- | --- | --- | --- |
| 查看监控 | 是 | 是 | 是 | 是 | 是 |
| 查看设备状态 | 是 | 是 | 是 | 是 | 是 |
| 创建设备配置 | 是 | 是 | 否 | 否 | 否 |
| 创建任务 | 是 | 是 | 是 | 按范围 | 否 |
| 执行任务 | 是 | 是 | 是 | 按范围 | 否 |
| 取消任务 | 是 | 是 | 是 | 按范围 | 否 |
| 设备手动控制 | 是 | 是 | 是 | 按范围 | 否 |
| 查看历史 | 是 | 是 | 是 | 是 | 是 |
| 确认告警 | 是 | 是 | 是 | 按范围 | 否 |
| 修改系统配置 | 是 | 是 | 否 | 否 | 否 |
| 管理用户角色 | 是 | 否 | 否 | 否 | 否 |
| 安全复位 | 是 | 按策略 | 按策略 | 否 | 否 |

按范围表示还需校验用户授权的 device_scope、task_scope 或 workspace_scope。

## 10.11 资源范围授权

角色只描述能力，范围描述能力作用对象。范围可以包括：

~~~text
device_scope = [R1, R2, HIK_FIX1]
task_scope = [inspection, sampling]
workspace_scope = [LEFT, RIGHT]
~~~

权限决策顺序：

1. 认证主体有效。
2. 拥有基础权限。
3. 操作对象属于授权范围。
4. 业务状态允许操作。
5. Controller 再执行 MODE、OWNER、RESOURCE、PERMIT 校验。

任何一步失败都返回拒绝，不能通过修改客户端请求中的 device_id 绕过范围检查。

## 10.12 登录失败与锁定

默认采用渐进式失败策略：

- 连续失败达到阈值后进入临时锁定；
- 重复失败延长锁定时间但设置最大上限；
- 成功登录清零失败计数；
- 管理员解锁会形成审计事件；
- 所有时间使用服务端 Clock；
- 不依据客户端时间决定锁定是否结束。

登录接口应增加 IP/客户端限流，但限流器不能把合法用户永久锁死。限流触发返回 AUTH_RATE_LIMITED，响应中不返回用户存在性信息。

## 10.13 注销和撤销

注销至少撤销当前 session。管理员可以撤销指定用户的全部 session。以下情况应触发撤销：

1. 用户被禁用或锁定。
2. 密码被修改。
3. 角色或权限发生重大变化。
4. 检测到 Refresh Token 重放。
5. 安全管理员执行强制下线。
6. 发生密钥轮换且策略要求重新登录。

Token 撤销表可以按 exp 自动清理，但清理不能早于所有相关 Token 的最大有效时间。

## 10.14 密钥管理

JWT 密钥不得写入代码、Git 仓库或普通日志。部署方式可以是受限配置文件、环境注入或密钥管理服务。配置中只保存 key_id 和引用信息，启动时检查：

- key_id 非空；
- 密钥长度满足算法要求；
- 当前密钥可验证；
- 过期密钥仍可按轮换窗口验证；
- 签发只使用 active key；
- 算法白名单不允许客户端指定。

密钥轮换采用双密钥窗口：新 Token 使用新密钥，验证端在窗口内同时信任旧密钥，窗口结束后撤销旧密钥。

## 10.15 错误模型

| 错误码 | 说明 | HTTP |
| --- | --- | --- |
| AUTH_INVALID_CREDENTIALS | 用户名或密码无效 | 401 |
| AUTH_TOKEN_MISSING | 缺少 Token | 401 |
| AUTH_TOKEN_INVALID | Token 无效 | 401 |
| AUTH_TOKEN_EXPIRED | Token 过期 | 401 |
| AUTH_SESSION_REVOKED | 会话已撤销 | 401 |
| AUTH_ACCOUNT_LOCKED | 账号被锁定 | 403 |
| AUTH_ACCOUNT_DISABLED | 账号被禁用 | 403 |
| AUTH_PERMISSION_DENIED | 权限不足 | 403 |
| AUTH_SCOPE_DENIED | 资源范围不允许 | 403 |
| AUTH_RATE_LIMITED | 认证请求过于频繁 | 429 |
| AUTH_SERVICE_UNAVAILABLE | 认证依赖不可用 | 503 |

错误消息不返回数据库异常、密码校验细节或内部堆栈。

## 10.16 与安全控制的关系

auth 通过只代表“用户有权发起操作”，不代表“设备现在允许运动”。运动命令必须依次经过：

~~~text
认证通过
  -> API 权限通过
  -> service 业务规则通过
  -> Controller 模式和任务规则通过
  -> OWNER/资源锁通过
  -> Device 状态允许
  -> Safety Permit 通过
  -> Access 下发
~~~

硬件急停、软件急停和设备自身保护优先于任何用户权限。任何管理员权限不能强行绕过硬件急停。

## 10.17 审计要求

以下事件必须记录：

- 登录成功和失败；
- 登出、强制下线；
- Token 刷新、撤销和重放；
- 用户创建、禁用、解锁、密码修改；
- 角色和权限变更；
- 认证配置、密钥引用和策略变更；
- 权限拒绝；
- 安全复位请求及结果。

审计中记录 actor_id、session_id、trace_id、operation_id、source、action、result、reason 和时间。密码、完整 Token、Authorization Header 和密钥内容禁止写入。

## 10.18 配置

~~~yaml
auth:
  access_token_ttl_seconds: 900
  refresh_token_ttl_seconds: 86400
  refresh_rotation: true
  password:
    algorithm: argon2id
    min_length: 12
    max_failed_attempts: 5
    lock_seconds: 300
  websocket:
    auth_timeout_ms: 5000
  key:
    active_key_id: mrss-primary
    accepted_key_ids: [mrss-primary, mrss-previous]
  rate_limit:
    login_per_minute: 10
~~~

示例仅说明配置结构，生产参数由安全评审和部署环境确定。

## 10.19 测试设计

| 测试类别 | 重点 |
| --- | --- |
| 密码单元测试 | 哈希、校验、重新哈希、错误密码 |
| Token 单元测试 | 签名、过期、aud、iss、jti、类型 |
| 会话测试 | 创建、刷新、轮换、撤销、过期 |
| RBAC 测试 | 五类角色和每项权限 |
| 范围测试 | 设备、任务和工作空间越权 |
| API 测试 | Header、Cookie、错误码和脱敏 |
| WebSocket 测试 | 超时、认证、订阅和重认证 |
| 安全测试 | 重放、暴力尝试、密钥轮换、算法混淆 |
| 故障测试 | DB 断开、审计失败、时钟异常 |
| 集成测试 | 登录后创建任务、控制命令和审计闭环 |

必须验证数据库不可用时新登录失败，但 STOP、急停和 Backend 关闭路径不被 auth 阻塞。

## 10.20 验收追踪

1. 五类内置角色能够创建并正确授权。
2. 密码数据库中不存在明文密码。
3. 过期、伪造、撤销和错误类型 Token 均被拒绝。
4. Refresh Token 重复使用会撤销会话。
5. WebSocket 未认证前不能订阅业务数据。
6. 权限通过后仍会执行 OWNER、PERMIT 和急停校验。
7. 权限拒绝、登录失败和管理员变更能够查询审计。
8. 认证错误不泄露用户存在性和内部异常。
9. 密钥轮换期间新旧 Token 行为符合配置。
10. 一期仿真、真实和混合模式使用相同 auth 流程。

## 10.21 扩展设计

后续接入 LDAP、OAuth2 或 OIDC 时，新增 IdentityProvider 接口，将外部身份映射到本地 User、Role 和 Scope。外部认证只替换身份来源，不改变 AuthContext、AuthorizationService、审计和业务权限接口。

## 10.22 本章小结

本章定义了 MRSS auth 模块的认证、授权、会话、Token、密码、WebSocket、范围权限、审计和安全策略。auth 负责用户身份与操作权限，不承担设备安全许可；所有控制操作仍必须经过模式、OWNER、资源锁、设备状态和三源急停联锁。