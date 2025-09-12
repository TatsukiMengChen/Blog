---
title: OSS 直传：如何构建一套安全高效的文件上传/访问方案
published: 2025-09-12
description: 深度解析“客户端预签名 URL 直传”模式，从零开始构建一套完整的文件上传与管理方案，为你的应用带来极致的上传以及访问体验。
image: https://oss.mengchen.xyz/2025/09/12/20250912144810088.avif
tags: ["全栈", "后端", "前端", "OSS", "实战"]
category: 全栈
draft: false
---

> 当你需要在 Web 应用中处理大文件上传时，一个经典的问题是：我们应该让文件经过应用服务器中转吗？答案通常是：不。本文将为你揭示一种更优解法——利用 **预签名 URL** 实现客户端文件直传至对象存储服务（OSS），并提供一套完整的技术流程和管理接口设计。

:::note[关于 OSS]

本文中的 OSS 泛指所有对象存储服务，如阿里云 OSS、腾讯云 COS、AWS S3 等。尽管服务商不同，但核心的**预签名 URL 直传**模式是通用的。

:::

## 核心概念

在我们深入技术细节之前，有几个核心概念必须先理清楚：

* **Key**：文件在 OSS 存储桶中的唯一身份标识，类似于文件在文件系统中的完整路径。为了便于管理，我们通常会采用 `年/月/日/UUID.扩展名` 的格式。
* **Meta**：一个灵活的 JSON 对象，用于存储文件的附加信息。例如，你可以用它来记录图片的尺寸、文件来源等业务相关数据，这比在数据库中为每个字段创建列要灵活得多。
* **预签名 URL (Pre-signed URL)**：一个由后端服务器通过 OSS 提供的 SDK 生成的、有时效性的 URL。前端拿到这个 URL 后，可以直接用它来执行特定的操作（如上传、下载），而不需要知道后端敏感的访问密钥。

## 方案优势

为什么这种“客户端直传”模式如此受欢迎？因为它完美地解决了传统中转模式的痛点：

* **减轻服务器负担**：文件数据不经过你的应用服务器，大文件的上传带宽和处理压力完全由 OSS 承担。
* **高安全性和高可用性**：直接利用 OSS 服务商提供的成熟、稳定的基础设施，通过预签名 URL 精确控制权限和时效，确保安全。
* **高扩展性**：上传并发能力直接由 OSS 支持，具备极强的水平扩展能力。

## 文件上传流程 (Create)

整个上传流程可以概括为“授权 -> 直传 -> 确认”三步走。

### 1. 前端：请求预签名接口

前端在用户选择文件后，不直接上传，而是先向你的后端发送一个轻量级的请求，提供文件的基本信息和元数据。

```json
POST /api/v1/uploads/presign
Headers:
Authorization: Bearer <JWT>
Body:
{
  "filename": "my-photo.jpg",
  "contentType": "image/jpeg",
  "size": 204800,
  "meta": {
    "source": "user\_profile\_avatar"
  }
}

````

### 2. 后端：生成预签名 URL 并返回

后端接收到请求后，会进行必要的验证，然后生成一个用于上传的预签名 URL。

* **验证**：检查用户权限、文件类型、大小等是否合规。
* **生成 Key**：根据预设规则（例如 `2025/09/12/UUID.jpg`）生成一个唯一的 Key。
* **调用 OSS SDK**：使用 Key 和操作类型（`PUT`）生成一个有时效的预签名 URL。
* **返回**：将生成的 `url` 和 `key` 返回给前端。

```json
HTTP/1.1 200 OK
{
  "url": "[https://your-bucket.oss-region.aliyuncs.com/2025/09/12/...?Signature=](https://your-bucket.oss-region.aliyuncs.com/2025/09/12/...?Signature=)...",
  "key": "/2025/09/12/a1b2c3d4-e5f6-7890-1234-567890abcdef.jpg"
}
````

### 3\. 前端：使用预签名 URL 直传

前端拿到 URL 后，直接用 `PUT` 方法将文件二进制数据上传到 OSS。这一步**完全绕开了你的应用服务器**。

```
PUT https://your-bucket.oss-region.aliyuncs.com/
Headers:
  Content-Type: image/jpeg
Body:
  <文件二进制数据>
```

### 4\. 前端：上传成功确认接口

文件成功上传到 OSS 后，前端需要调用后端的确认接口，通知后端将文件记录保存到数据库中。

```json
POST /api/v1/uploads/confirm
Headers:
  Authorization: Bearer <JWT>
Body:
{
  "key": "/2025/09/12/a1b2c3d4-e5f6-7890-1234-567890abcdef.jpg",
  "meta": { ... }
}
```

后端接收到请求后，会通过 OSS SDK 校验文件是否存在，如果存在，则将文件信息（Key、元数据、用户ID等）存入数据库。

```json
HTTP/1.1 201 Created
{
  "id": "db-uuid-abcdef-123456",
  "key": "/2025/09/12/a1b2c3d4-e5f6-7890-1234-567890abcdef.jpg",
  "accessUrl": "https://cdn.yourdomain.com/2025/09/12/a1b2c3d4-e5f6-7890-1234-567890abcdef.jpg"
}
```

## 文件管理接口 (CRUD)

除了上传，一个完整的文件服务还需要支持读取、更新和删除操作。

### 1\. 读取 (Read)

要获取文件的信息或访问 URL，可以设计一个通用的读取接口。

```
GET /api/v1/files/{id}
```

对于私有文件，后端可以返回一个**有时效性的只读预签名 URL**。然而，如果你的前端代码中已经写死了 OSS 的静态 URL，并且你不想或不能修改前端代码，那么后端直接返回预签名 URL 就不太实用了。这时，可以考虑以下两种方案：

#### 方案一：后端代理 / 中转

所有文件读取请求都先发到你的后端服务器，由后端验证权限后再去 OSS 获取文件数据，最后返回给浏览器。

**优点**：实现简单，安全性高。
**缺点**：所有流量都经过你的服务器，会增加服务器的带宽和处理压力，可能产生性能瓶颈。

#### 方案二：Service Worker 无感认证

这是最优雅的解决方案，尤其适用于前端 URL 是静态的场景。**Service Worker** 作为浏览器中的一个代理，能拦截所有网络请求。

**工作原理**：

1.  浏览器发起对静态 OSS URL（如 `https://oss.my-domain.com/user-photo.jpg`）的请求。
2.  Service Worker 捕获该请求，但**不直接放行**。
3.  Service Worker 携带用户的 **JWT** 向后端请求该文件的预签名 URL。
4.  后端验证 JWT，生成预签名 URL 并返回给 Service Worker。
5.  Service Worker 再用这个预签名 URL 去 OSS 真正获取图片数据。
6.  Service Worker 将获取到的图片数据作为原始请求的响应返回给浏览器。

**优点**：无需修改前端代码，用户对整个认证过程完全无感知，同时利用了 OSS 的高速 CDN。
**缺点**：实现复杂度相对较高，需要管理 Service Worker 的生命周期和缓存策略。

| 方案 | 优点 | 缺点 | 适用场景 |
| :--- | :--- | :--- | :--- |
| **后端代理** | 实现简单；安全性高；所有逻辑在后端。 | 增加服务器带宽和延迟；所有请求都要经过你的服务器。 | 不在意服务器开销和轻微延迟；不具备 Service Worker 开发能力。 |
| **Service Worker** | 无需修改前端代码；用户无感；利用 OSS CDN，性能好。 | 实现相对复杂；需要管理 Service Worker 的生命周期和缓存。 | **前端 URL 静态写死**；追求高性能和无缝用户体验。 |

### 2\. 更新 (Update)

OSS 中的文件内容是不可变的。因此，“更新”通常意味着两种情况：

  * **更新元数据**：通过一个 `PATCH /api/v1/files/{id}` 接口，只更新数据库中存储的 `meta` 字段。
  * **替换文件**：这不应该有独立的接口。正确的做法是**复用完整的上传流程**，上传一个新文件，然后你的业务逻辑（例如更新用户头像）使用新文件的 `id` 和 `key` 替换旧的。旧文件可以根据业务策略决定是否异步删除。

### 3\. 删除 (Delete)

删除文件需要同步处理 OSS 和数据库记录。

```
DELETE /api/v1/files/{id}
```

后端通过 `id` 找到 `key`，先调用 OSS SDK 删除文件，然后从数据库中删除记录。为了保证数据一致性，这两个操作应该放在一个数据库事务中。

## 数据库设计

一个简单的 `files` 表足以存储文件信息，我们可以在此基础上进行扩展。

```sql
CREATE TABLE files (
    id UUID PRIMARY KEY,
    user_id UUID NOT NULL,
    key VARCHAR(512) UNIQUE NOT NULL,
    original_filename VARCHAR(255),
    content_type VARCHAR(128),
    size BIGINT,
    meta JSONB,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

## 总结

采用“客户端预签名 URL 直传”模式，我们构建了一套安全、高效、可扩展的文件上传与管理方案。它不仅减轻了服务器的压力，还为应用带来了流畅的用户体验。通过前后端的分工协作，我们利用了各自的优势：后端负责安全的授权，而前端则直接利用 OSS 的强大能力完成数据传输。希望这份文档能帮助你的团队高效地实现文件上传功能，为未来的业务发展打下坚实的基础。
