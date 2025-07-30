---
title: 认证模块演进式设计：从 JWT 会话控制到分布式架构
published: 2025-07-29
description: 旨在提供一个全面、可演进的认证鉴权系统设计方案。它从一个基于JWT和数据库的基础会话管理系统开始，逐步引入性能优化策略，最终扩展为一个适用于异地分布式 K8s 环境的高性能、高可用的统一认证平台。
image: https://d988089.webp.li/2025/07/29/20250729024019711.jpg
tags: ["后端", "分布式", "JWT", "鉴权"]
category: 后端
draft: false
---

## JWT 会话管理基础设计

### 1. 引言

本设计旨在解决基于 JWT 的无状态认证在实际应用中遇到的挑战：无法方便地踢出用户使其离线，以及难以限制用户登录的设备数量。我们将通过引入刷新令牌（Refresh Token）和会话管理机制，在保持 JWT 访问令牌轻量、可伸缩优势的同时，实现对用户会话的精细化控制。

实现目标：

* 方便地踢出特定用户或特定设备，使其当前会话立即失效。
* 限制用户同时登录的设备数量（例如，最多3台）。
* 实现同类型设备登录时的自动踢出：当用户在新的同类型设备（如另一台手机）上登录时，自动使最旧的同类型设备会话失效。

### 2. 核心概念

#### 2.1 访问令牌 (Access Token - JWT)

* **性质**：短寿命的 JSON Web Token。
* **用途**：用于访问受保护的 API 资源。
* **特点**：无状态，一旦签发，服务器不存储其状态。过期后自动失效。

#### 2.2 刷新令牌 (Refresh Token)

* **性质**：长寿命的、唯一的字符串，存储在数据库中。
* **用途**：当访问令牌过期时，用于向认证服务请求新的访问令牌。
* **特点**：有状态，与用户 ID、设备信息、过期时间等关联。服务器可以对其进行管理（撤销、查询）。

#### 2.3 设备类型 (Device Type)

为了实现同类型设备踢出，我们需要在登录时识别设备类型。

* **枚举值**：`PC` (电脑), `MOBILE` (手机), `TABLET` (平板)。
* **获取方式**：通常由客户端在登录请求中提供。

#### 2.4 会话管理

我们将通过在数据库中维护 `RefreshToken` 记录来实现会话管理。每条 `RefreshToken` 记录代表一个活跃的设备会话。

### 3. 数据库 Schema 设计 (Prisma)

在现有的 `User` 模型基础上，我们将新增一个 `RefreshToken` 模型来存储刷新令牌及其相关信息。

```prisma
// prisma/schema.prisma

// 用户模型 (User Model)
model User {
  id            Int            @id @default(autoincrement())
  username      String         @unique
  email         String         @unique
  password      String
  createdAt     DateTime       @default(now())
  updatedAt     DateTime       @updatedAt
  // 关联刷新令牌
  refreshTokens RefreshToken[]
}

// 刷新令牌模型 (RefreshToken Model)
model RefreshToken {
  id         Int      @id @default(autoincrement())
  token      String   @unique // 存储唯一的刷新令牌字符串
  userId     Int      // 关联的用户ID
  user       User     @relation(fields: [userId], references: [id], onDelete: Cascade) // 当用户被删除时，其所有刷新令牌也删除
  deviceType String   // 设备类型: PC, MOBILE, TABLET
  deviceId   String   @unique // 设备的唯一标识符，例如 UUID，由客户端生成并提供
  deviceName String?  // 新增：设备的名称，方便用户识别
  expiresAt  DateTime // 刷新令牌的过期时间
  isValid    Boolean  @default(true) // 软删除标记，用于立即失效
  createdAt  DateTime @default(now())
  updatedAt  DateTime @updatedAt

  @@index([userId]) // 为 userId 字段添加索引，提高查询效率
  @@unique([userId, deviceId]) // 确保同一用户同一设备只有一个活跃令牌
}
````

**字段解释**：

  * **`RefreshToken.isValid`**: 一个布尔标志，用于“软删除”或立即使刷新令牌失效，而无需真正从数据库中删除记录。
  * **`RefreshToken.deviceId`**: 客户端提供的设备唯一标识，用于区分同一用户的不同设备。
  * **`RefreshToken.deviceName`**: 用于存储用户可读的设备名称（例如“我的iPhone 15”），方便用户在会话管理界面识别。

### 4\. API 端点设计

  * `POST /auth/register`: 用户注册。
  * `POST /auth/login`: 用户登录。
  * `POST /auth/refresh`: 使用刷新令牌获取新的访问令牌。
  * `POST /auth/logout`: 注销当前设备会话。
  * `POST /auth/logout-all-devices`: 注销用户在所有设备上的会话。
  * `GET /auth/active-sessions`: 获取用户当前所有活跃的设备会话列表。
  * `DELETE /auth/active-sessions/:id`: 注销指定 ID 的设备会话（踢出某个设备）。

### 5\. 认证流程设计

#### 5.1 登录流程 (`/auth/login`)

1.  **客户端**：发送 `POST /auth/login` 请求，包含 `username`, `password`, `deviceType`, `deviceId`, `deviceName`。
2.  **服务端**：
    a. 验证用户名和密码。
    b. **检查设备限制**：查询数据库中该用户 `isValid = true` 的 `RefreshToken` 数量。
    c. **执行踢出策略**：如果已达到最大限制（例如3台），查找该用户最旧的同类型设备会话，将其 `isValid` 字段设置为 `false`。如果无同类型设备可踢，则踢出最旧的一个会话。
    d. **生成并保存令牌**：生成新的 `RefreshToken` 并存入数据库，同时生成短寿命的 `AccessToken`。
    e. **返回令牌**：将 `AccessToken` 和 `RefreshToken` 返回给客户端。

#### 5.2 访问令牌刷新流程 (`/auth/refresh`)

1.  **客户端**：当 `AccessToken` 过期时，发送 `POST /auth/refresh` 请求，包含 `RefreshToken`。
2.  **服务端**：
    a. 在数据库中查找该 `RefreshToken`，校验其是否存在、是否有效 (`isValid = true`)、是否过期。
    b. 若验证失败，返回 401，客户端应强制用户重新登录。
    c. 若验证成功，**为了安全，生成一个新的 `RefreshToken` 替换旧的**（一次性使用原则），并签发一个新的 `AccessToken`。
    d. 返回新的双令牌。

#### 5.3 受保护资源访问流程

1.  **客户端**：发送包含 `AccessToken` (Bearer Token) 的请求到受保护的 API。
2.  **服务端 (JWT 策略)**：
    a. 验证 `AccessToken` 的签名和过期时间。
    b. \*\*（关键步骤）\*\*从 `AccessToken` 的 payload 中提取 `userId` 和 `deviceId`。
    c. **查询 `RefreshToken` 表，检查是否存在 `userId` 和 `deviceId` 匹配且 `isValid` 为 `true` 的记录。**
    d. 如果不存在或 `isValid` 为 `false`，则抛出 `UnauthorizedException`，即使 `AccessToken` 本身未过期。
    e. 验证通过，请求继续处理。

#### 5.4 注销流程

  * **注销当前设备 (`/auth/logout`)**: 将数据库中当前用户和设备对应的 `RefreshToken` 记录的 `isValid` 字段设置为 `false`。
  * **注销所有设备 (`/auth/logout-all-devices`)**: 将该用户所有 `RefreshToken` 记录的 `isValid` 设为 `false`。
  * **注销指定设备 (`DELETE /auth/active-sessions/:id`)**: 将指定 ID 的 `RefreshToken` 记录的 `isValid` 设为 `false`。

### 6\. 客户端注意事项

  * **生成 `deviceId`**：客户端（前端）需在首次启动时生成一个唯一 `deviceId` (UUID) 并持久化存储（如 `localStorage`）。
  * **存储 `RefreshToken`**：应存储在安全的地方，例如 **HttpOnly Cookie** (Web端) 或移动应用的安全存储区。绝不能存储在 `localStorage`。
  * **`AccessToken` 管理**：通常存储在内存中，通过 `Authorization` 请求头发送。

## JWT 方案性能优化策略

上述基础方案虽然功能完备，但在“受保护资源访问流程”的 5.3.c 步骤中，**每次 API 请求都需要查询数据库**，这在高并发场景下会成为性能瓶颈。

**核心思想**：用高速缓存（如 Redis）替代慢速的关系型数据库（如 PostgreSQL）来进行高频的“有效性检查”。

### 1\. 核心优化：引入 Redis 作为“吊销列表 (Revocation List)”

我们不再在每次请求时都去查询数据库，而是查询速度快几个数量级的 Redis。

**工作原理**：

1. **吊销时（踢出/登出）**：当一个会话需要被终止时（用户登出、被踢出），我们执行两个操作：

     * **更新数据库**：将 PostgreSQL 中对应 `RefreshToken` 的 `isValid` 设为 `false`（保持事实来源的准确性）。
     * **写入 Redis 吊销列表**：将被吊销的 `AccessToken` 的一个唯一标识（使用其 `jti` 声明）存入 Redis，并**设置一个等于该 `AccessToken` 剩余生命周期的 TTL**。

   *Redis 操作示例 (假设 Access Token 还有 840 秒过期)*：

   ```sh
   # 'jti' (JWT ID) 是 JWT payload 中的一个标准字段，非常适合做唯一标识
   SET revoked_tokens:<jti_of_the_token> true EX 840
   ```

   这样做的好处是，一旦令牌自然过期，其在 Redis 中的吊销记录也会自动清理，不会造成内存无限增长。

2. **验证时（访问受保护 API）**：`JwtStrategy` 的验证逻辑变为：
   a. 验证 JWT 的签名和基础过期时间。
   b. 从 JWT 的 payload 中提取 `jti`。
   c. **查询 Redis**：检查 `revoked_tokens:<jti>` 这个键是否存在。
   d. 如果键**存在**，说明该令牌已被吊销，立即拒绝访问 (401 Unauthorized)。
   e. 如果键**不存在**，说明令牌有效，允许访问。

### 2\. 改良后的 JWT 认证流程

  * **登录 (`/auth/login`)**: 流程不变，依旧操作数据库，生成含 `jti` 的 `AccessToken`。
  * **访问受保护API**:
    1.  客户端携带 `AccessToken`。
    2.  认证中间件验证JWT签名和`exp`。
    3.  从Payload中提取 `jti`。
    4.  查询 Redis `EXISTS revoked_tokens:<jti>`。
    5.  若存在，拒绝 (401)；若不存在，放行。
  * **登出/踢出**:
    1.  服务端找到要吊销的会话。
    2.  从其 `AccessToken` 解析出 `jti` 和剩余过期时间 `ttl`。
    3.  执行 `SET revoked_tokens:<jti> true EX <ttl>` 写入 Redis。
    4.  执行 `UPDATE "RefreshToken" SET "isValid" = false ...` 更新数据库。

### 3\. 总结对比

| 特性             | 原始 JWT 方案                | 优化后的 JWT 方案                   |
| :--------------- | :--------------------------- | :---------------------------------- |
| **API 验证开销** | 高: JWT解密 + **数据库查询** | 低: JWT解密 + **Redis 查询**        |
| **性能瓶颈**     | 关系型数据库的 I/O           | Redis 的网络延迟 (通常极低)         |
| **实现复杂度**   | 中等                         | 稍高: 需管理数据库和 Redis 的同步   |
| **即时撤销能力** | 具备                         | 具备，且性能更高                    |
| **架构**         | 应用服务器 \<-\> 数据库      | 应用服务器 \<-\> Redis \<-\> 数据库 |

**结论**：通过引入 Redis 作为“吊销列表”，我们实现了一个“两全其美”的方案：对外保留了 JWT 的优点（跨域、微服务友好），对内通过高速缓存实现了高性能、有状态的会话控制。这种 **“JWT + Redis 吊销列表”** 模式是现代分布式应用中非常成熟和流行的行业实践。

## 异地分布式 Kubernetes 统一认证鉴权平台技术设计

当业务部署在多个地理区域的 Kubernetes 集群上时，我们需要解决跨地域延迟和全局实时会话同步的问题。本设计将前述方案扩展为一个分布式架构。

### 1\. 设计目标

  * **高性能**: 核心API的鉴权延迟（P99）应在10毫秒以内，不受限于跨地域网络。
  * **高可用**: 无单点故障，单个地域的故障不影响其他地域。
  * **实时控制**: 会话吊销指令应在1秒内全局生效。
  * **强安全性**: 遵循业界最佳安全实践。

### 2\. 整体架构

本方案采用一种**以JWT为载体、以本地缓存加速、以消息总线同步**的混合模式。

#### 2.1 组件职责

  * **API Gateway / Ingress**: 流量入口。将写操作（登录/刷新/登出）路由到中央认证服务，将读操作（业务API请求）路由到各地域的业务服务。
  * **认证服务 (Auth Service)**: **中心化**的服务。处理用户登录、令牌刷新、会话吊销等**写密集型**操作。是唯一有权限写入 PostgreSQL 和发布吊销事件的服务。
  * **业务服务 (Business Service)**: 部署在各地域K8s集群中。包含一个认证中间件，负责所有API请求的鉴权。
  * **PostgreSQL (事实来源 - Source of Truth)**: **中心化**部署的高可用数据库集群，持久化存储用户及 `RefreshToken` 表。
  * **本地Redis缓存 (Local Read Cache)**: 通过 `DaemonSet` 部署在**每个K8s节点**上。核心职责是缓存已吊销的 `jti` 列表，为本节点的业务服务提供近乎零延迟的鉴权检查。
  * **消息总线 (Message Bus - NATS / Redis Pub/Sub)**: **全局部署**的高性能消息系统。核心职责是当认证服务吊销会话时，立即向全局发布一条“吊销事件”消息。

### 3\. 核心流程

![分布式鉴权流程图](https://d988089.webp.li/2025/07/29/20250729023145715.png)

#### 3.1 API请求鉴权 (高性能读路径)

1.  用户携带 `AccessToken` 访问任意地域的业务服务。
2.  该服务的认证中间件开始鉴权：
    a. **JWT基础验证**: 验证签名和`exp`。
    b. **本地缓存检查**: 从Token解析 `jti`，查询**节点本地的Redis**，检查 `revoked_jti:<jti>` 是否存在。
    c. 如果存在，立即拒绝(401)。
    d. 如果不存在，鉴权通过，处理业务逻辑。

#### 3.2 会话吊销 (实时写路径)

1.  用户（或管理员）发起登出/踢出请求，路由到**中央认证服务**。
2.  认证服务执行以下操作：
    a. **更新事实来源**: 更新 PostgreSQL 中 `RefreshToken` 的 `isValid = false`。
    b. **发布吊销事件**: 构造一条包含被吊销Token的 `jti` 和 `exp` 的消息。
    c. 通过 **NATS** 将此消息发布到全局 Topic (例如 `auth:revocation`)。
3.  **全局同步**:
    a. **所有地域**的订阅者（一个轻量级服务）收到消息。
    b. 订阅者解析消息，计算出 `ttl = exp - now()`。
    c. 立即将 `jti` 写入其**所在节点的本地Redis**，并设置TTL：`SET revoked_jti:<jti> 1 EX <ttl>`。
4.  至此，该 `AccessToken` 在全局所有节点都已即时失效。

### 4\. 高可用与容错设计

  * **本地Redis缓存**: 通过 `DaemonSet` 部署，单个Pod故障由K8s自愈。即使整个节点Redis故障，鉴权可降级为查询中央数据库（缓存回源），服务仍可用。
  * **消息总线 (NATS)**: 应以高可用集群模式部署。短暂中断会延迟吊销，但系统具备自愈能力（Redis的TTL会自动清理过期项）。
  * **中央数据库 (PostgreSQL)**: 是系统的核心依赖，必须采用主从复制、自动故障转移等高可用方案。它的故障将影响所有写操作（登录、刷新）。