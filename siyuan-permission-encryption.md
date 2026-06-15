# SiYuan 权限与文档加密代码结构分析

> 分析版本：基于当前代码库快照 | 分析日期：2026-06-15

---

## 目录

1. [系统架构总览](#1-系统架构总览)
2. [访问控制体系](#2-访问控制体系)
3. [密钥与口令状态管理](#3-密钥与口令状态管理)
4. [同步服务配置加密](#4-同步服务配置加密)
5. [解锁流程详解](#5-解锁流程详解)
6. [内容读写与访问控制联动](#6-内容读写与访问控制联动)
7. [界面提示联动机制](#7-界面提示联动机制)
8. [错误处理机制](#8-错误处理机制)
9. [备份恢复与数据仓库](#9-备份恢复与数据仓库)
10. [安全边界分析](#10-安全边界分析)
11. [协作模块关系图](#11-协作模块关系图)
12. [潜在风险分析](#12-潜在风险分析)
13. [后续验证清单](#13-后续验证清单)

---

## 1. 系统架构总览

SiYuan 的权限与加密体系由 **三层防护** 构成：

| 层级 | 功能 | 核心模块 |
|------|------|---------|
| **网络接入层** | HTTP 请求鉴权、来源校验、跨域防护 | [kernel/api/router.go](kernel/api/router.go), [kernel/model/session.go](kernel/model/session.go) |
| **业务逻辑层** | 角色权限控制、只读模式、发布访问过滤 | [kernel/model/role.go](kernel/model/role.go), [kernel/model/publish_access.go](kernel/model/publish_access.go) |
| **数据存储层** | 数据仓库加密、快照备份、云端同步加密 | [kernel/util/crypt.go](kernel/util/crypt.go), [kernel/model/repository.go](kernel/model/repository.go), [kernel/conf/repo.go](kernel/conf/repo.go) |

### 1.1 核心代码文件清单

| 文件路径 | 主要职责 |
|----------|---------|
| [kernel/util/crypt.go](kernel/util/crypt.go) | AES 加密解密、SHA256 哈希 |
| [kernel/model/auth.go](kernel/model/auth.go) | JWT 认证、发布服务账户、Basic Auth |
| [kernel/model/session.go](kernel/model/session.go) | 访问授权码登录、验证码、登出、角色权限检查中间件 |
| [kernel/util/session.go](kernel/util/session.go) | Session 数据结构、验证码计数 |
| [kernel/model/role.go](kernel/model/role.go) | 角色定义（Admin/Editor/Reader/Visitor）、角色校验函数 |
| [kernel/model/repository.go](kernel/model/repository.go) | 数据仓库密钥管理、快照索引、回滚、云端同步 |
| [kernel/model/cloud_service.go](kernel/model/cloud_service.go) | 云端用户数据加密、激活码、订阅刷新 |
| [kernel/model/publish_access.go](kernel/model/publish_access.go) | 发布访问控制（可见性/密码/禁止）、内容过滤 |
| [kernel/model/sync.go](kernel/model/sync.go) | WebDAV/S3/Local 同步配置管理 |
| [kernel/conf/repo.go](kernel/conf/repo.go) | 数据仓库配置结构（密钥/保留策略） |
| [kernel/conf/sync.go](kernel/conf/sync.go) | 同步服务配置结构（WebDAV/S3/Local） |
| [kernel/api/repo.go](kernel/api/repo.go) | 数据仓库 REST API 端点 |
| [kernel/api/sync.go](kernel/api/sync.go) | 同步配置导入导出 REST API 端点 |
| [kernel/server/proxy/publish.go](kernel/server/proxy/publish.go) | 发布服务反向代理、Basic Auth + Session 认证 |
| [kernel/api/router.go](kernel/api/router.go) | API 路由注册（中间件链：CheckAuth → CheckAdminRole → CheckReadonly） |
| [app/stage/auth.html](app/stage/auth.html) | 前端授权登录页面（验证码/记住我/退出） |
| [app/src/protyle/util/publishAccess.ts](app/src/protyle/util/publishAccess.ts) | 前端发布访问控制对话框（5种级别选择） |

---

## 2. 访问控制体系

### 2.1 角色模型

定义于 [kernel/model/role.go#L21-L32](kernel/model/role.go#L21-L32)：

```go
type Role uint

const (
    RoleAdministrator Role = iota  // 0 - 管理员（完全控制）
    RoleEditor                     // 1 - 编辑者（读写）
    RoleReader                     // 2 - 读者（只读）
    RoleVisitor                    // 3 - 匿名访客
)
```

**权限继承关系**：Admin > Editor > Reader > Visitor

| 能力 | Admin | Editor | Reader | Visitor |
|------|-------|--------|--------|---------|
| 修改系统配置 | ✅ | ❌ | ❌ | ❌ |
| 创建/修改/删除文档 | ✅ | ✅ | ❌ | ❌ |
| 读取文档内容 | ✅ | ✅ | ✅ | 受限 |
| 导出数据 | ✅ | ✅ | ❌ | ❌ |
| 管理数据仓库 | ✅ | ❌ | ❌ | ❌ |

### 2.2 认证方式矩阵

系统支持 **6 种认证途径**，定义于 [CheckAuth](kernel/model/session.go#L207-L384)：

| 认证方式 | 优先级 | 适用场景 | 授予角色 |
|----------|--------|---------|---------|
| JWT Token (`X-Auth-Token` Header) | 1 | 发布服务反向代理请求 | Reader / Editor / Admin |
| API Token (`Authorization: Token xxx` Header) | 2 | 第三方 API 集成 | Admin |
| API Token (`?token=xxx` Query) | 3 | URL 参数式调用 | Admin |
| Session Cookie | 4 | 浏览器登录用户 | Admin |
| Basic Auth | 5 | WebDAV / CalDAV / CardDAV | Admin |
| 来源白名单 (127.0.0.1 + 无授权码) | 6 | 本地无授权码场景 | Admin |

### 2.3 中间件执行链

在 [kernel/api/router.go](kernel/api/router.go) 中，每个受保护 API 按以下顺序注册中间件：

```
请求 → CheckAuth → CheckAdminRole → CheckReadonly → 实际 Handler
         ↓             ↓                ↓
    验证身份      验证管理员      验证非只读模式
    注入Role      拒绝非Admin     拦截写操作
```

**示例**：
```go
ginServer.Handle("POST", "/api/filetree/removeDoc", 
    model.CheckAuth,       // 第1关：必须已认证
    model.CheckAdminRole,  // 第2关：必须是管理员
    model.CheckReadonly,   // 第3关：不能是只读模式
    removeDoc)             // 实际处理函数
```

### 2.4 发布服务独立认证

发布服务通过独立的反向代理端口运行，认证逻辑在 [kernel/server/proxy/publish.go#L131-L196](kernel/server/proxy/publish.go#L131-L196)：

1. **Session Cookie 认证**：检查 `publish-visitor-session-id`，有效则直接注入 JWT
2. **Basic Auth 认证**：校验用户名密码，创建新 Session，返回 Cookie + JWT
3. **匿名访问**：未启用发布认证时，自动使用匿名账户 JWT（Role = Reader）

---

## 3. 密钥与口令状态管理

### 3.1 密钥/口令类型总览

| 类型 | 存储位置 | 用途 | 长度/格式 |
|------|---------|------|----------|
| **访问授权码** | `conf.json: accessAuthCode` | 工作区登录保护 | 任意字符串 |
| **API Token** | `conf.json: api.token` | REST API 认证 | 任意字符串 |
| **数据仓库密钥** | `conf.json: repo.key` | 快照/同步 AES 加密 | 32 字节 (AES-256) |
| **发布访问密码** | `publishAccess.json: password` | 单篇发布文档保护 | 任意字符串 |
| **发布服务账户** | `conf.json: publish.auth.accounts` | 发布站点全局登录 | 用户名 + 密码 |
| **JWT 签名密钥** | 内存（`jwtKey`） | 发布服务 Token 签名 | 32 字节（每次启动随机生成） |
| **静态 AES 密钥** | 代码硬编码 `SK` | 同步配置导入导出、云端用户数据 | 16 字节 (AES-128) |
| **WebDAV 密码** | `conf.json: sync.webdav.password` | WebDAV 同步服务认证 | 任意字符串（明文存储） |
| **S3 SecretKey** | `conf.json: sync.s3.secretKey` | S3 对象存储认证 | 任意字符串（明文存储） |

### 3.2 数据仓库密钥生命周期

密钥管理位于 [kernel/model/repository.go](kernel/model/repository.go)，有 **三种生成方式**：

#### 方式一：随机生成密钥 [InitRepoKey](kernel/model/repository.go#L789-L824)

```
流程：
  1. crypto/rand 生成 16 字节 password + 16 字节 salt
  2. encryption.KDF(password, salt) → 32 字节密钥
  3. 写入 Conf.Repo.Key
  4. 清空本地 repo 目录
  5. 触发首次索引 initDataRepo()
```

#### 方式二：通过口令派生密钥 [InitRepoKeyFromPassphrase](kernel/model/repository.go#L751-L787)

```
流程：
  输入 passphrase
    ↓
  判断是否为 Base64(32字节)?
    ├─ 是 → 直接解码作为密钥
    └─ 否 → SHA256(passphrase) 取前16字节作为 salt
            → encryption.KDF(passphrase, salt) → 32字节密钥
    ↓
  写入 Conf.Repo.Key
  清空 repo 目录
  触发首次索引
```

#### 方式三：导入现有密钥 [ImportRepoKey](kernel/model/repository.go#L647-L679)

```
流程：
  Base64 字符串输入
    ↓
  Base64 解码 → 验证长度 == 32 字节
    ↓
  写入 Conf.Repo.Key
  清空本地 repo 目录（需重新从云端拉取）
  触发首次索引
```

### 3.3 密钥状态判定

```go
// 密钥是否已初始化的关键检查
if 1 > len(Conf.Repo.Key) {
    // 返回错误："请先初始化数据仓库" (Language 26)
    return errors.New(Conf.Language(26))
}
```

该检查贯穿所有仓库操作：索引、回滚、差异对比、云端同步、快照浏览等。

### 3.4 ⚠️ 硬编码静态密钥用途与风险

在 [kernel/util/crypt.go#L29-L30](kernel/util/crypt.go#L29-L30) 中：

```go
var SK = []byte("696D897C9AA0611B")          // 硬编码密钥 (16字节)
var IV = []byte("RandomInitVector")            // 硬编码初始向量 (16字节)
```

**算法细节**：
- **算法**：AES-128-CBC + PKCS5Padding
- **编码链**：明文 → Hex 编码 → AES-CBC 加密 → Hex 编码（输出密文）
- **解码链**：密文 → Hex 解码 → AES-CBC 解密 → Hex 解码（输出明文）

**已确认的 4 个用途**：

| 用途 | 位置 | 说明 |
|------|------|------|
| 云端用户数据本地持久化 | [cloud_service.go#L390](kernel/model/cloud_service.go#L390) | `AESEncrypt(JSON(user))` → `Conf.UserData` → 存 conf.json |
| 云端用户数据本地加载 | [cloud_service.go#L403](kernel/model/cloud_service.go#L403) | `AESDecrypt(Conf.UserData)` → 恢复 user 对象到内存 |
| 云端 API 返回数据解密 | [cloud_service.go#L570](kernel/model/cloud_service.go#L570) | 云端 `/apis/siyuan/user` 返回加密的用户数据，本地解密 |
| 同步配置导入导出 | [api/sync.go#L144,L193,L337,L386](kernel/api/sync.go#L144) | WebDAV / S3 配置包导出加密、导入解密 |

**风险等级**：高（密钥公开于源码，所有加密数据可被解密）

---

## 4. 同步服务配置加密

### 4.1 同步服务配置结构

定义于 [kernel/conf/sync.go#L46-L78](kernel/conf/sync.go#L46-L78)：

```go
type WebDAV struct {
    Endpoint       string `json:"endpoint"`       // 服务端点 URL
    Username       string `json:"username"`       // 用户名 (明文)
    Password       string `json:"password"`       // 密码 (明文)
    SkipTlsVerify  bool   `json:"skipTlsVerify"`  // 跳过 TLS 验证
    Timeout        int    `json:"timeout"`        // 超时秒数
    ConcurrentReqs int    `json:"concurrentReqs"` // 并发请求数
}

type S3 struct {
    Endpoint       string `json:"endpoint"`       // 服务端点
    AccessKey      string `json:"accessKey"`      // Access Key (明文)
    SecretKey      string `json:"secretKey"`      // Secret Key (明文)
    Bucket         string `json:"bucket"`         // 存储空间
    Region         string `json:"region"`         // 存储区域
    PathStyle      bool   `json:"pathStyle"`      // 路径风格
    SkipTlsVerify  bool   `json:"skipTlsVerify"`  // 跳过 TLS 验证
    Timeout        int    `json:"timeout"`
    ConcurrentReqs int    `json:"concurrentReqs"`
}
```

**同步提供者枚举**：
```go
const (
    ProviderSiYuan = 0 // 思源官方云端
    ProviderS3     = 2 // S3 协议对象存储
    ProviderWebDAV = 3 // WebDAV 协议
    ProviderLocal  = 4 // 本地文件系统
)
```

### 4.2 配置存储方式

**运行时存储**：
- WebDAV 用户名/密码、S3 AccessKey/SecretKey **全部明文** 存储在 `conf.json` 的 `Sync` 节点下
- 设置函数：[kernel/model/sync.go#L434-L467](kernel/model/sync.go#L434-L467)
  - `SetSyncProviderWebDAV()` → 直接赋值 `Conf.Sync.WebDAV = webdav`
  - `SetSyncProviderS3()` → 直接赋值 `Conf.Sync.S3 = s3`

**脱敏处理**：
- `GetMaskedConf()` [kernel/model/conf.go#L1038-L1058](kernel/model/conf.go#L1038-L1058)：
  - ✅ `UserData` 置空
  - ✅ `AccessAuthCode` 掩码为 `*******`
  - ❌ **同步配置（含密码/密钥）原样返回**
- `HideConfSecret()` [kernel/model/conf.go#L1062-L1076](kernel/model/conf.go#L1062-L1076)：
  - ✅ 整个 `Sync` 对象重置为空
  - ✅ 同时清空 AI、Api、Repo、Publish 等敏感配置

### 4.3 WebDAV 配置导入导出加密流程

#### 导出流程 [exportSyncProviderWebDAV](kernel/api/sync.go#L167-L229)

```
用户点击"导出"
    ↓
[1] 序列化 JSON
    data = JSON.Marshal(Conf.Sync.WebDAV)
    包含明文: endpoint, username, password, skipTlsVerify...
    ↓
[2] 静态 AES 加密
    dataStr = util.AESEncrypt(string(data))
    → 明文 → Hex → AES-128-CBC(SK, IV) → Hex
    ↓
[3] 写入临时文件
    文件名: siyuan-webdav-YYYYMMDDHHMMSS.json
    路径: {TempDir}/export/{name}
    权限: 0644
    ↓
[4] 打包 ZIP
    创建 {name}.zip，包含 {name}.json
    ↓
[5] 返回下载路径
    /export/{name}.zip
```

#### 导入流程 [importSyncProviderWebDAV](kernel/api/sync.go#L38-L165)

```
用户上传 .zip 或 .json 文件
    ↓
[1] 读取上传文件到内存
    ↓
[2] 解压/提取
    .zip → 解压到 {TempDir}/import/webdav/
    .json → 直接复制
    ↓
[3] 验证包结构
    临时目录必须仅含 1 个文件
    ↓
[4] AES 解密
    data = util.AESDecrypt(string(fileContent))
    → Hex 解码 → AES 解密 → Hex 解码
    ↓
[5] 反序列化
    JSON.Unmarshal(data, &webdav)
    ↓
[6] 验证与保存
    SetSyncProviderWebDAV(webdav)
    → 检查坚果云屏蔽规则
    → 规范化 endpoint/timeout/concurrentReqs
    → Conf.Save() 持久化到 conf.json
    ↓
[7] 返回当前配置
    ret.Data = {"webdav": Conf.Sync.WebDAV}
```

### 4.4 S3 配置导入导出加密流程

S3 配置导入导出流程与 WebDAV **完全一致**，定义于：
- 导出：[exportSyncProviderS3](kernel/api/sync.go#L360-L422)
- 导入：[importSyncProviderS3](kernel/api/sync.go#L231-L358)

### 4.5 与内容读写、安全边界的关联

**数据流向图**：

```
┌─────────────┐   明文密码    ┌─────────────┐   AES-256-GCM    ┌────────────┐
│ conf.json   │────────────▶│  Sync 模块   │────────────────▶│ WebDAV/S3  │
│ (WebDAV    │  (运行时内存) │ (同步引擎)  │  (repo.key 加密) │  云端存储  │
│  Password) │               └─────────────┘                  └────────────┘
│             │                                                        ▲
│ (S3         │               ┌─────────────┐                          │
│  SecretKey) │────────────▶│ 数据仓库     │                          │
│             │  明文引用    │ (dejavu库)  │─── AES-256-GCM 对象 ────┘
└─────────────┘               └─────────────┘
        │
        │ 导入/导出 AES-128-CBC (SK硬编码)
        ▼
┌─────────────┐
│ 导出包      │  可被任意获取源码者解密
│ .zip/.json  │
└─────────────┘
```

**关键关联点**：

1. **同步认证与内容加密分层**：
   - WebDAV/S3 认证使用明文密码建立连接
   - 但传输的**数据内容**（仓库对象）已通过 `repo.key` 独立加密为 AES-256-GCM
   - 即使 WebDAV 服务被攻破，获取的对象仍需 repo.key 才能解密

2. **安全边界分层**：
   ```
   边界 1 (网络): HTTPS 传输 → 保护同步认证密码
   边界 2 (应用): 导入导出 AES → 保护配置包在文件传递中不被一眼看穿
   边界 3 (数据): repo.key AES-256 → 保护文档内容本身
   ```

3. **导入导出包的安全意义**：
   - 防止导出的 `.zip` 包在邮件/聊天传输中被直接阅读密码
   - **但不提供真正的安全性**（密钥在源码中公开）
   - 属于"混淆"而非"加密"级别保护

4. **API 权限边界**：
   - 导入导出端点均需 `CheckAuth + CheckAdminRole + CheckReadonly` 三重校验
   - 仅管理员可操作，防止低权限用户获取同步密码

---

## 5. 解锁流程详解

### 5.1 工作区访问授权码解锁

核心流程位于 [LoginAuth](kernel/model/session.go#L67-L163)：

```
用户输入 authCode + 可选 captcha + rememberMe
        ↓
[1] 验证码检查 (NeedCaptcha → WrongAuthCount > 3)
    ├─ 需要验证码 → 比对 session.Captcha (大小写不敏感)
    │   ├─ 错误 → 返回 code=1 (需刷新验证码), 重置 Captcha
    │   └─ 正确 → 继续
    └─ 不需要 → 跳过
        ↓
[2] 授权码比对 Conf.AccessAuthCode == input
    ├─ 错误 → WrongAuthCount++, 
    │         code=-1 + 错误消息
    │         若 NeedCaptcha → 额外返回 code=1 触发验证码UI
    │         重置 session.Captcha
    │         返回
    └─ 正确 → 继续
        ↓
[3] 认证通过
    - workspaceSession.AccessAuthCode = authCode
    - WrongAuthCount = 0
    - Captcha 重置
        ↓
[4] Session 过期设置
    ├─ rememberMe=true  → MaxAge = 30天 (2592000秒)
    └─ rememberMe=false → MaxAge = 0 (浏览器会话)
        ↓
[5] Session 持久化 + 广播 loginAuth 事件
    - WebSocket 通知 auth 通道的所有连接
    - 前端 auth.html 监听后跳转
```

### 5.2 验证码触发机制 [NeedCaptcha](kernel/util/session.go#L28-L30)

```go
func NeedCaptcha() bool {
    return 3 < WrongAuthCount  // 连续错误3次后触发
}
```

**验证码生成参数** ([GetCaptcha](kernel/model/session.go#L165-L193))：
- 尺寸：100×26 像素
- 字符集：`ABCDEFGHKLMNPQRSTUVWXYZ23456789`（去除易混淆字符）
- 噪声：0.5
- 曲线：0 条
- 存入 session，每次验证后无论成功失败都重置

### 5.3 发布文档密码解锁

文档级密码保护由 [CheckPublishAuthCookie](kernel/model/publish_access.go#L240-L243) 和 [SetPublishAuthCookie](kernel/model/publish_access.go#L228-L238) 实现：

```
访问受保护文档路径
    ↓
沿路径向上查找密码（GetPathPasswordByPublishAccess）
    文档 → 父文档 → ... → 笔记本
    ↓
找到 passwordID + password
    ↓
检查 Cookie: publish-auth-{passwordID}
    值 == SHA256(passwordID + password)?
    ├─ 是 → 通过，返回原文
    └─ 否 → 渲染密码输入框 HTML (🔒 图标)
```

**密码验证成功后**：
- 设置 `publish-auth-{ID}` Cookie
- 值 = `SHA256(ID + password)`
- MaxAge = 24 小时
- HttpOnly = true, Secure = SSL, Path = /

### 5.4 数据仓库密钥解锁（隐式）

数据仓库密钥 **不单独解锁**，而是：
1. 工作区启动时从 `conf.json` 加载到内存
2. 每个仓库操作前检查 `len(Conf.Repo.Key) > 0`
3. 如果为 nil/空 → 返回提示"请初始化数据仓库"
4. **重置仓库** ([ResetRepo](kernel/model/repository.go#L681-L703)) 会清空密钥 + 禁用同步

### 5.5 云端用户数据解锁（隐式）

云端用户数据通过 `AESEncrypt` 加密存储在 `Conf.UserData` 中：
- 启动时 [loadUserFromConf](kernel/model/cloud_service.go#L398-L410) 自动调用 `AESDecrypt` 解密
- 登录/刷新用户时 [RefreshUser](kernel/model/cloud_service.go#L330-L396) 重新加密保存
- 解密后 User 对象仅在内存中存在，不持久化

---

## 6. 内容读写与访问控制联动

### 6.1 发布访问控制过滤器链

在 [kernel/model/publish_access.go](kernel/model/publish_access.go) 中实现了 **多层内容过滤**：

```
原始内容
    ↓
[FilterContentByPublishAccess] 文档内容渲染过滤
    ├─ 检测密码保护 → 未登录替换为 🔒密码输入框
    └─ 检测禁止发布 → 替换为 🚫禁止提示
    ↓
[FilterBlocksByPublishAccess] 搜索结果/列表过滤
    → 去除不可见 + 密码未登录文档
    ↓
[FilterPathsByPublishAccess] 文档树路径过滤
    → 去除不可见 + 密码未登录路径
    ↓
[FilterViewByPublishAccess] 数据库视图过滤 (表格/画廊/看板)
    → 过滤行 + 替换封面为密码图标
    ↓
[FilterGraphByPublishIgnore] 关系图过滤
    → 移除不可见节点 + 相关边
    ↓
[FilterTagsByPublishIgnore] 标签云过滤
    → 重新计算标签计数，移除不可见文档贡献
    ↓
[FilterAssetContentByPublishAccess] 资源文件过滤
    → 仅保留可见文档引用的 assets
```

### 6.2 过滤核心函数分析

#### 密码路径继承 [GetPathPasswordByPublishAccess](kernel/model/publish_access.go#L191-L216)

```
算法：继承式密码查找
  当前文档路径 blockPath
    ↓
  while currentPath != "/" and password == "":
      currentID = basename(currentPath) 去掉 .sy
      遍历 publishAccess 找 ID 匹配项
      找到 → password = item.password, passwordID = item.ID
      currentPath = dirname(currentPath)
    ↓
  仍无密码 → 检查笔记本级别密码
    ↓
  返回 (passwordID, password)
```

**关键特性**：子文档继承父文档/笔记本的密码，无需每篇单独设置。

#### 内容级过滤 [FilterContentByPublishAccess](kernel/model/publish_access.go#L471-L518)

输出两种 HTML 占位：

**密码保护（未解锁）**：
```html
<div class="protyle-password" data-node-id="{passwordID}">
    <span class="protyle-password__logo">🔒</span>
    <label class="b3-form__icon protyle-password__content">
        <input type="text" placeholder="请输入访问密码"/>
        <svg class="protyle-password__button">→</svg>
    </label>
</div>
```

**禁止发布**：
```html
<div class="protyle-password protyle-password--forbidden" data-node-id="{ID}">
    <span class="protyle-password__logo">🚫</span>
    <div class="protyle-password__tip">该文档已被禁止发布</div>
</div>
```

### 6.3 只读模式联动

只读模式双重检查 ([CheckReadonly](kernel/model/session.go#L195-L205))：

```go
func CheckReadonly(c *gin.Context) {
    // 全局只读标志 OR 上下文角色为只读
    if util.ReadOnly || IsReadOnlyRoleContext(c) {
        result.Code = -1
        result.Msg = Conf.Language(34)  // "当前处于只读模式"
        result.Data = {"closeTimeout": 5000}
        c.Abort()  // 终止请求链
        return
    }
}
```

**全局只读触发源**：
- 启动参数 `--readonly`
- 容器环境变量
- 角色为 Reader / Visitor (通过 JWT 注入)

---

## 7. 界面提示联动机制

### 7.1 授权登录页面 [app/stage/auth.html](app/stage/auth.html)

**页面元素与后端联动**：

| UI 元素 | 触发条件 | 后端信号 |
|---------|---------|---------|
| 密码输入框 | 默认显示 | 总是存在 |
| 验证码图片+输入框 | `response.code === 1` | WrongAuthCount > 3 后后端返回 |
| "记住我" 复选框 | 用户自主选择 | rememberMe=true → MaxAge=30天 |
| "退出思源"按钮 | 仅 localhost + 主窗口 | 调用 /api/system/exit |
| 顶部错误提示 Snackbar | `response.code !== 0` | 返回 response.msg |

**WebSocket 实时联动**（[app/stage/auth.html#L584-L591](app/stage/auth.html#L584-L591)）：

```javascript
const ws = new WebSocket('ws://host/ws?...&type=auth');
ws.onmessage = (event) => {
    const data = JSON.parse(event.data);
    if (data.cmd === "loginAuth") {
        window.location.href = toPath;  // 别处登录成功，自动跳转
    }
};
```

**用途**：多标签页/多窗口登录同步——一个窗口登录，所有授权页自动跳转。

### 7.2 发布访问控制对话框 [app/src/protyle/util/publishAccess.ts](app/src/protyle/util/publishAccess.ts)

**五级访问控制模型** ([getPublishAccessLevel](app/src/protyle/util/publishAccess.ts#L50-L67))：

| 级别 | 图标 | visible | password | disable | 说明 |
|------|------|---------|----------|---------|------|
| `public` | 🌐 | true | "" | false | 公开，所有人可见 |
| `protected` | 🔒 | true | 非空 | false | 可见，需密码访问 |
| `hidden` | 👻 | false | "" | false | 隐藏（知道直接URL可访问） |
| `private` | 🤫 | false | 非空 | false | 隐藏 + 密码双重保护 |
| `forbidden` | 🚫 | false | "" | true | 完全禁止发布 |

对话框数据流：
```
openPublishAccessDialog(id)
    ↓ GET /api/filetree/getPublishAccess {ids:[id]}
    获取当前 (visible, password, disable)
    → 映射为 level → 高亮对应图标按钮
    ↓ 用户点击图标
    → 切换 level → 更新注释文字 + 密码输入框显隐
    ↓ 用户确认
    → level → 转回 (visible, password, disable)
    → callback({id, visible, password, disable, iconHTML})
    ↓ 调用方 POST /api/filetree/setPublishAccess
```

### 7.3 消息广播机制

关键广播事件（通过 WebSocket `BroadcastByType`）：

| 事件类型 | 频道 | 触发时机 | 前端响应 |
|---------|------|---------|---------|
| `loginAuth` | auth | 授权码登录成功 | 所有等待窗口跳转 |
| `logoutAuth` | main | 用户主动登出 | 前端跳转授权页 |
| `syncMergeResult` | main | 云端同步合并完成 | 刷新受影响文档 |
| `reloadUI` | 全局 | 仓库回滚/配置变更 | 整页刷新 |

---

## 8. 错误处理机制

### 8.1 认证错误处理矩阵

| 错误场景 | HTTP 状态码 | 返回 Code | 消息 (Language ID) | 附加操作 |
|---------|------------|-----------|-------------------|---------|
| 授权码错误 | 200 | -1 | "访问授权码错误" (83) | WrongAuthCount++ |
| 连续失败 ≥4 次 | 200 | 1 (叠加) | 同上 + 验证码字段 | 强制显示验证码 |
| 验证码错误 | 200 | 1 | "验证码错误" (22) | 生成新验证码 |
| 空授权码登出 | 200 | -1 | "当前未设置访问授权码" (86) | 忽略 |
| API Token 错误 | 401 | -1 | "Auth failed [header: Authorization]" | 无重试 |
| Query Token 错误 | 401 | -1 | "Auth failed [query: token]" | 无重试 |
| 非本地无授权码访问 | 401 | -1 | 中英双语长提示 | 建议设置授权码 |
| Session 已过期 | 302 | - | 重定向到 `/check-auth` | 保存原始 URL |
| 角色不足 (Admin) | 403 | - | (空响应体) | Forbidden |
| 角色不足 (Editor) | 403 | - | (空响应体) | Forbidden |
| 角色不足 (Reader) | 403 | - | (空响应体) | Forbidden |
| 只读模式写操作 | 200 | -1 | "当前处于只读模式" (34) | 5秒后自动关闭提示 |

### 8.2 同步配置导入导出错误处理

| 错误场景 | HTTP 状态码 | 返回 Code | 说明 |
|---------|------------|-----------|------|
| 上传文件数 ≠ 1 | 200 | -1 | "invalid upload file" |
| ZIP 解压失败 | 200 | -1 | "invalid WebDAV provider package" |
| 解密失败 | 200 | -1 | AESDecrypt 返回 nil + 日志错误 |
| JSON 反序列化失败 | 200 | -1 | "import WebDAV provider failed" |
| 坚果云 WebDAV 拦截 | 200 | -1 | "不支持配置坚果云 WebDAV 进行同步" (194) |
| 路径遍历攻击 | 200 | -1 | "import path is not sub path of import dir" |
| 导出 JSON 序列化失败 | 200 | -1 | "export WebDAV provider failed" |
| ZIP 打包失败 | 200 | -1 | "export WebDAV provider failed" |

### 8.3 数据仓库错误处理

| 错误场景 | 处理方式 | 用户提示 |
|---------|---------|---------|
| 密钥未初始化 | 提前返回，所有仓库操作短路 | "请先初始化数据仓库" (26) |
| 索引失败 | 写入日志 + 推送错误消息 | "创建数据仓库索引失败" + 错误详情 (140) |
| 云端空间不足 | 检测 `ErrCloudStorageSizeExceeded` | "云端存储空间已满 (已使用 X)" |
| 云端备份数超限 | 检测 `ErrCloudBackupCountExceeded` | "备份失败：云端备份数量超出限制" |
| 快照回滚失败 | 推送错误消息到状态栏 | "数据仓库快照回滚失败" (141) |
| 密钥导入格式错误 | Base64 解码失败 / 长度 ≠ 32 | "导入数据仓库密钥失败" (157) |
| 仓库致命错误 `ErrRepoFatal` | 自动重试间隔递增 | 同步状态写入错误信息 |

### 8.4 自动重试与降级

**同步重试策略** ([kernel/model/repository.go](kernel/model/repository.go))：

```
同步失败
    ↓
autoSyncErrCount++
    ↓
planSyncAfter( fixSyncInterval = 30s * 2^autoSyncErrCount )
    → 最大延迟约 8 分钟
    → 10 次相同后指数退避减缓
```

**冲突降级**：
- 开启 `GenerateConflictDoc` → 冲突文档重命名为 `xxx (Conflicted)` 并存
- 关闭 → 自动以云端为准/本地为准合并（根据同步模式）
- 冲突副本存放到 `/history/` 目录

---

## 9. 备份恢复与数据仓库

### 9.1 数据仓库架构

```
workspace/
├─ repo/                      # 本地仓库存储（dejavu 格式）
│  ├─ objects/                # 内容寻址块 (AES-256 加密)
│  ├─ indexes/                # 快照索引列表
│  └─ tags/                   # 命名快照标签
└─ data/.siyuan/conf.json     # repo.key, WebDAV/S3 密码存储处
```

**加密范围**：
- **仓库文件内容块**：使用 `Conf.Repo.Key` AES-256-GCM (dejavu/encryption 库)
- **工作区配置文件**：conf.json  **明文** 存储 (包含 repo.key + WebDAV/S3 密码!)
- **源文档 (.sy 文件)**：**明文** JSON 存储在 workspace/data/

> ⚠️ **重要理解**：数据仓库加密保护的是 **快照/备份/同步数据**，不是工作区的运行时文档文件。
> ⚠️ **同步配置加密分层**：导入导出包使用 AES-128 混淆保护，但 conf.json 中仍为明文。

### 9.2 快照创建流程 [IndexRepo](kernel/model/repository.go#L1162-L1205)

```
触发：用户手动 / 同步前自动 / 回滚前自动
    ↓
[1] 检查 Conf.Repo.Key 存在性
    ↓
[2] FlushTxQueue() → 所有未提交文档写入磁盘
    ↓
[3] repo.Index(memo, true, ctx)
    → 分块扫描 workspace/data/
    → 计算文件哈希去重 (SHA256)
    → 新文件内容 AES 加密写入 objects/
    → 生成索引 (文件清单 + 元数据)
    ↓
[4] 对比 latest 索引
    ├─ ID 相同 → "无变化，耗时 X 秒" (148)
    └─ 不同 → "已创建快照，耗时 X 秒" (147)
```

### 9.3 快照恢复流程 [checkoutRepo](kernel/model/repository.go#L839-L896)

```
触发：用户选择历史快照 → CheckoutRepo(id)
    ↓
[1] 安全措施
    ├─ 自动暂停云端同步 (Conf.Sync.Enabled=false)
    ├─ FlushTxQueue() 确保当前数据落盘
    ├─ 关闭资产/表情监听器 (避免恢复中文件变更)
    └─ 为 *当前* 数据创建备份快照 "Backup before checkout"
    ↓
[2] 执行 Checkout
    repo.Checkout(id, ctx)
    → AES-256 解密对象
    → 完整重写 workspace/data/ 目录
    ↓
[3] 索引重建
    FullReindex(true) → SQL 数据库完全重建
    ReloadFiletree / ReloadProtyle → 前端刷新
    ↓
[4] 恢复状态
    若原先是开启同步 → 7秒后推送提醒"记得开启同步"
```

### 9.4 单文件回滚 [RollbackRepoSnapshotFile](kernel/model/repository.go#L191-L297)

```
输入: fileID (快照文件引用)
    ↓
repo.GetFile(fileID) → 获取元数据 (Path/Size/Updated)
repo.OpenFile(file) → 解密获取字节流
    ↓
文件类型分支：
├─ .sy 文档 → 解析为 parse.Tree → 定位笔记本/路径
│   → 路径冲突处理 (同名文档是否覆盖/重命名)
│   → 写入新 .sy 文件 → 重建树索引 → 刷新前端
│   → 消息："已从快照回滚文档到 /笔记本名/路径"
└─ 其他文件 → 直接复制到 workspace/data/{Path}
    → 消息："已从快照恢复文件到路径"
    ↓
IncSync() → 标记需要同步
```

### 9.5 仓库清理策略

**自动清理** ([autoPurgeRepo](kernel/model/repository.go#L75-L168))：
- 每 12 小时执行一次（首次同步后启动）
- `IndexRetentionDays` (默认 180天)：超出此范围的索引可被清理
- `RetentionIndexesDaily` (默认 2个/天)：每天保留索引数量
- 算法：每天最后 1 个必保留 + 随机抽样至目标数量
- 手动清理 `PurgeRepo()` → 清理所有未引用的 objects

---

## 10. 安全边界分析

### 10.1 网络边界

#### 本地访问白名单 [CheckAuth](kernel/model/session.go#L258-L287)

```
条件 (所有条件满足 → 允许无授权码)：
1. Conf.AccessAuthCode == "" (未设置授权码)
2. util.SiYuanAccessAuthCodeBypass = true (桌面端内部标志)
3. OR 所有来源检查通过:
   ├─ RemoteAddr is localhost
   ├─ ClientIP is 127.0.0.1 / ::1 / localhost
   ├─ Host header is local
   ├─ Origin: local / chrome-extension://
   └─ X-Forwarded-Host is local
```

**边界穿透风险**：反向代理伪造 `X-Forwarded-Host: 127.0.0.1` → 被第3层 Host/Origin 检查拦截。

#### 发布服务网络隔离 [kernel/server/proxy/publish.go](kernel/server/proxy/publish.go)

```
外部请求 → 发布端口 (独立 listener)
    ↓ PublishServiceTransport.RoundTrip
[1] 发布服务认证 (Basic Auth / Session)
[2] 注入 JWT (X-Auth-Token: role=Reader)
    ↓
转发到内部 Kernel Server URL (util.ServerURL)
    ↓
[3] CheckAuth 识别 JWT → 注入 Role=Reader
[4] 后续 CheckAdminRole → 写操作全部 403
```

**关键点**：发布服务的 JWT 密钥每次启动重新生成（`crypto/rand 32字节`），重启后所有发布服务会话失效。

### 10.2 数据边界

| 数据区域 | 加密状态 | 访问控制 |
|---------|---------|---------|
| workspace/data/*.sy | 明文 | 系统文件权限 + API 鉴权 |
| workspace/repo/objects/* | AES-256-GCM (repo.key) | repo.key 持有者可解密 |
| 云端同步对象 | AES-256-GCM (repo.key) | 同上 + 云端账户认证 |
| workspace/conf.json | 明文 (含 repo.key + WebDAV/S3 密码!) | 系统文件权限 |
| 同步配置导出包 (.zip) | AES-128-CBC (硬编码 SK) | 混淆级保护，可被源码读者解密 |
| Conf.UserData (云端用户) | AES-128-CBC (硬编码 SK) | 混淆级保护 |
| Session Cookie (浏览器) | HttpOnly + Secure(SSL时) | 浏览器同源策略 |
| 发布密码 Cookie | SHA256 哈希 + HttpOnly | 24小时过期 |

### 10.3 同步配置安全边界

```
导出流程边界：
  内存 (明文 WebDAV 密码)
      ↓ AESEncrypt(SK)
  导出包 (.zip/.json) → AES-128 混淆
      ↓ 网络传输 (HTTPS)
  用户存储 (U盘/邮件)
      ↓ 导入时 AESDecrypt(SK)
  内存 → conf.json (明文持久化)

安全漏洞链：
  源码公开 → SK 已知 → 所有历史导出包可被解密 → 获取 WebDAV/S3 密码
  ↓
  conf.json 明文 → 直接获取密码
  ↓
  结合 HTTPS 抓包 → 获取同步数据 → 需 repo.key 才能解密内容
```

### 10.4 并发安全边界

- **Session 写入**：`sessionLock sync.Mutex` 保护 sessionsMap
- **发布访问控制缓存**：`publishAccessLock sync.Mutex` + 30秒 TTL
- **API 并发控制**：写操作路径全局串行化 ([ControlConcurrency](kernel/model/session.go#L456-L507))
- **仓库操作**：dejavu 库内部文件锁 `filelock`
- **导入导出临时目录**：使用 `TempDir` 隔离 + 路径遍历检查 (`IsSubPath`)

---

## 11. 协作模块关系图

```
┌───────────────────────────────────────────────────────────────────────┐
│                         前端层 (Electron/Browser)                       │
│  ┌──────────────┐     ┌───────────────────┐    ┌────────────────────┐  │
│  │  auth.html   │     │ publishAccess.ts  │    │ syncConfigDialog   │  │
│  │ (登录/验证码)│────▶│ (5级发布权限对话框)│    │ (WebDAV/S3导入导出)│  │
│  └──────┬───────┘     └─────────┬─────────┘    └─────────┬──────────┘  │
│         │                      │                          │             │
│         │ POST /api/system/*   │ GET/POST filetree       │ POST /api/sync │
└─────────┼──────────────────────┼──────────────────────────┼─────────────┘
          │                      │                          │
┌─────────▼──────────────────────▼──────────────────────────▼─────────────┐
│                        API 路由层 (kernel/api/)                          │
│  ┌───────────────────────────────────────────────────────────────────┐  │
│  │  中间件链: Recover → Timing → ControlConcurrency → CheckAuth        │  │
│  │             → CheckAdminRole → CheckReadonly → Handler             │  │
│  └───────────────────────────────────────────────────────────────────┘  │
└─────────┬──────────────────────┬──────────────────────────┬─────────────┘
          │                      │                          │
┌─────────▼──────────────────────▼──────────────────────────▼─────────────┐
│                        业务模型层 (kernel/model/)                        │
│  ┌───────────────┐  ┌────────────────┐  ┌──────────────────────────────┐ │
│  │ session.go    │  │publish_access. │  │     repository.go            │ │
│  │ ·登录/登出     │  │go              │  │ ·密钥管理(3种方式)           │ │
│  │ ·验证码       │  │ ·5级过滤链      │  │ ·快照索引/回滚                │ │
│  │ ·CheckAuth    │  │ ·密码Cookie     │  │ ·云端同步(上传/下载/合并)    │ │
│  │ ·角色检查     │  │ ·内容/视图/图   │  │ ·冲突处理(副本/历史)         │ │
│  └───────┬───────┘  │   过滤         │  └─────────────┬────────────────┘ │
│          │          └────────┬───────┘                │                  │
│  ┌───────▼───────┐  ┌───────▼────────┐  ┌─────────────▼────────────────┐ │
│  │   auth.go     │  │   role.go      │  │     cloud_service.go         │ │
│  │ ·JWT生成/解析 │  │ ·4级角色定义   │  │ ·用户数据 AESEncrypt/Decrypt  │ │
│  │ ·发布账户管理 │  │ ·权限校验函数  │  │ ·激活码/订阅刷新              │ │
│  └───────────────┘  └────────────────┘  └─────────────┬────────────────┘ │
│                                                        │                  │
│  ┌──────────────────────────────┐  ┌───────────────────▼──────────────┐ │
│  │     sync.go                  │  │          crypt.go                │ │
│  │ ·WebDAV/S3 配置管理          │  │ ·AES-128-CBC (硬编码 SK/IV)      │ │
│  │ ·坚果云拦截/参数规范化        │  │ ·SHA256 哈希                    │ │
│  │ ·同步模式/间隔设置            │  │                                 │ │
│  └──────────────────────────────┘  └──────────────────────────────────┘ │
└─────────┬──────────────────────────────────────────────────────────────┘
          │
┌─────────▼──────────────────────────────────────────────────────────────┐
│                        配置/持久化层 (kernel/conf/)                      │
│  ┌───────────────────┐  ┌──────────────────┐  ┌──────────────────────┐ │
│  │ conf.go           │  │ repo.go          │  │ sync.go              │ │
│  │ ·AccessAuthCode   │  │ ·Key [32]byte    │  │ ·WebDAV (明文密码)   │ │
│  │ ·Api.Token        │  │ ·保留天数/数量   │  │ ·S3 (明文SecretKey)  │ │
│  │ ·UserData (AES)   │  │ ·本地仓库路径    │  │ ·Local/Provider枚举  │ │
│  │ ·ReadOnly         │  │                  │  │                      │ │
│  └───────────────────┘  └──────────────────┘  └──────────────────────┘ │
└─────────┬──────────────────────────────────────────────────────────────┘
          │
┌─────────▼──────────────────────────────────────────────────────────────┐
│                        外部服务/存储层                                    │
│  ┌──────────────────┐  ┌─────────────────┐  ┌────────────────────────┐ │
│  │  dejavu 库       │  │  SiYuan Cloud   │  │ WebDAV / S3 / Local    │ │
│  │ (AES-GCM加密     │  │ (HTTPS + API    │  │ (第三方同步目标)       │ │
│  │  内容寻址存储)   │  │  签名 Token)    │  │                        │ │
│  └──────────────────┘  └─────────────────┘  └────────────────────────┘ │
└────────────────────────────────────────────────────────────────────────┘
```

---

## 12. 潜在风险分析

### 12.1 高风险项

| 风险 | 位置 | 影响 | 建议 |
|------|------|------|------|
| **conf.json 明文存储密钥** | [kernel/model/conf.go](kernel/model/conf.go) | 攻击者获取工作区目录即可拿到 repo.key、WebDAV 密码、S3 SecretKey，解密所有备份/快照 + 登录第三方同步服务 | 考虑使用 OS Keychain/DPAPI 加密存储敏感配置 |
| **硬编码 AES 密钥** | [kernel/util/crypt.go#L29](kernel/util/crypt.go#L29) | `SK="696D897C9AA0611B"` 和 `IV` 公开于源码，所有同步配置导出包、UserData 可被解密 | 评估是否仍需该加密，或改为动态生成密钥绑定设备 |
| **同步配置导入导出伪加密** | [kernel/api/sync.go#L144,L193](kernel/api/sync.go#L144) | 导出包使用硬编码密钥加密，形同虚设，仅提供混淆级别保护 | 改用用户提供的导出密码 + PBKDF2 派生密钥，或直接移除加密改为警告 |
| **发布密码 SHA256 无盐** | [kernel/model/publish_access.go#L229](kernel/model/publish_access.go#L229) | Cookie 值 = SHA256(ID+password)，可彩虹表破解弱密码 | 改为使用 bcrypt/argon2 + 随机盐 |
| **运行时 .sy 文档明文** | workspace/data/ | 服务器被攻破时，所有文档直接可读 | 可考虑提供"工作区级透明加密"选项 |
| **JWT 无过期时间** | [kernel/model/auth.go#L107-L117](kernel/model/auth.go#L107-L117) | 发布服务 JWT 未设置 exp 声明，理论上永久有效 | 添加合理的 exp (如 24h) + 滑动刷新 |

### 12.2 中风险项

| 风险 | 位置 | 影响 | 建议 |
|------|------|------|------|
| **GetMaskedConf 泄露同步密码** | [kernel/model/conf.go#L1038-L1058](kernel/model/conf.go#L1038-L1058) | 该 API 仅掩码 AccessAuthCode 和 UserData，但完整返回 Sync.WebDAV.Password 和 Sync.S3.SecretKey | 将同步敏感字段也加入掩码 |
| **验证码 session 存储** | [kernel/model/session.go#L180](kernel/model/session.go#L180) | 验证码明文存 session，理论上可从 Cookie 侧推断 | 哈希后存储，或一次性验证后立即删除 |
| **错误计数全局共享** | [kernel/util/session.go#L26](kernel/util/session.go#L26) | `WrongAuthCount` 是进程级变量，攻击者 A 触发验证码会影响所有用户 B 的登录 | 改为按 IP + UserAgent 维度计数 |
| **授权码明文比对** | [kernel/model/session.go#L116](kernel/model/session.go#L116) | `Conf.AccessAuthCode != authCode` 使用 Go 字符串 `!=`，理论可被时序攻击 | 改用 `subtle.ConstantTimeCompare` |
| **Cookie 缺少 SameSite** | 多处 Cookie 设置 | 可能易受 CSRF 攻击 | 添加 `SameSite=Lax` 或 `Strict` |
| **Checkout 前备份覆盖** | [kernel/model/repository.go#L874](kernel/model/repository.go#L874) | 连续快速回滚会覆盖"回滚前备份"，丢失真正的原始数据 | 备份索引添加时间戳标签防止覆盖 |

### 12.3 低风险项

| 风险 | 位置 | 影响 | 建议 |
|------|------|------|------|
| 30秒发布访问缓存 | [kernel/model/publish_access.go#L61-L62](kernel/model/publish_access.go#L61-L62) | 修改权限后最多 30 秒才生效 | 提供主动失效接口 |
| 验证码字符集有限 | [kernel/model/session.go#L167](kernel/model/session.go#L167) | 仅大写+数字，7 位字符熵约 41 bit | 考虑添加小写字母或增加长度 |
| 仓库自动清理随机性 | [kernel/model/repository.go#L149-L150](kernel/model/repository.go#L149-L150) | `mathRand` 选择保留的索引，非密码学安全 | 如需可审计性改用 crypto/rand |
| 导入导出临时目录权限 | [kernel/api/sync.go#L76,L173](kernel/api/sync.go#L76) | 临时文件权限为 0644，同机用户可读取 | 改为 0600 限制访问 |
| 坚果云拦截仅检查域名 | [kernel/model/sync.go#L454](kernel/model/sync.go#L454) | 仅检查 `dav.jianguoyun.com` 字符串，可通过 IP/反向代理绕过 | 如需严格限制，需额外检测机制 |

---

## 13. 后续验证清单

### 13.1 功能验证

#### A. 解锁流程测试
- [ ] **A1** 空授权码 + 本地访问 → 自动通过，不跳登录页
- [ ] **A2** 空授权码 + 远程 IP 访问 → 拦截，提示设置授权码
- [ ] **A3** 连续输错 3 次授权码 → 第 4 次起要求验证码
- [ ] **A4** 验证码输入错误 → 提示错误 + 自动刷新图片
- [ ] **A5** 验证码输入正确 + 授权码正确 → 登录成功
- [ ] **A6** 勾选"记住我" → 关闭浏览器后重新打开仍登录 (30天)
- [ ] **A7** 不勾选"记住我" → 关闭浏览器后需重新登录
- [ ] **A8** 多窗口打开登录页 → 一个窗口成功，所有窗口自动跳转
- [ ] **A9** 登出后 → 回到登录页 + 清除 session

#### B. 数据仓库密钥测试
- [ ] **B1** 随机生成密钥 → 索引成功 → 导出 Base64 密钥备份
- [ ] **B2** 通过密码短语生成密钥 → 相同密码每次生成的密钥应一致 (确定性派生)
- [ ] **B3** 导入正确 Base64 密钥 → 原仓库数据可读取
- [ ] **B4** 导入错误密钥 → 解密失败但不应崩溃
- [ ] **B5** 导入长度 ≠ 32 字节 → 返回"密钥格式错误" (157)
- [ ] **B6** 清空密钥 → 所有仓库操作返回"请初始化"
- [ ] **B7** 重置仓库 → Key 清空 + Sync 禁用 + 用户需重新初始化

#### C. 同步配置导入导出测试
- [ ] **C1** WebDAV 配置导出 → 生成 .zip 包含加密 .json
- [ ] **C2** WebDAV 配置导入 → 正确解密并保存密码到 conf.json
- [ ] **C3** S3 配置导出 → 生成 .zip 包含加密 .json
- [ ] **C4** S3 配置导入 → 正确解密并保存 SecretKey 到 conf.json
- [ ] **C5** 导入损坏的 zip → 返回"invalid WebDAV provider package"
- [ ] **C6** 导入非加密 json → AESDecrypt 失败，返回错误
- [ ] **C7** 坚果云 WebDAV 端点 → 被拦截，返回 Language 194
- [ ] **C8** 导入路径遍历攻击 (../) → 被 IsSubPath 拦截
- [ ] **C9** GetMaskedConf API → 检查是否返回明文 WebDAV 密码

#### D. 发布访问控制测试
- [ ] **D1** 5 级权限 (public/protected/hidden/private/forbidden) 各自正确渲染
- [ ] **D2** protected 级别 → 输入正确密码 → 24h 内无需再输
- [ ] **D3** 父文档设置密码 → 子文档自动继承密码
- [ ] **D4** 笔记本设置密码 → 所有下属文档继承
- [ ] **D5** forbidden 文档 → 搜索/列表/图/标签中均不出现
- [ ] **D6** 数据库视图 (Table/Gallery/Kanban) → 行按权限过滤
- [ ] **D7** 引用/嵌入文档 → 按目标文档权限过滤显示
- [ ] **D8** 资源文件 → 仅可见文档引用的 assets 可访问

#### E. 云端同步 & 备份恢复测试
- [ ] **E1** Checkout 快照 → 自动创建"回滚前备份"快照
- [ ] **E2** Checkout 期间 → 云端同步自动暂停
- [ ] **E3** 单文件回滚 → 文档正确恢复 + 不影响其他文件
- [ ] **E4** 云端空间已满 → 明确提示已用空间
- [ ] **E5** 备份数量超限 → 明确提示错误 (84+154)
- [ ] **E6** 同步中断后重试 → 指数退避正常
- [ ] **E7** 自动清理 → 超过 180 天的索引被清理，每天保留 2 个
- [ ] **E8** 冲突文档生成 → 命名正确 + 路径正确

### 13.2 安全验证

#### F. 认证边界测试
- [ ] **F1** 伪造 `X-Forwarded-Host: 127.0.0.1` → 无法绕过授权码检查
- [ ] **F2** 伪造 Origin: `chrome-extension://malicious` → 仍需检查其他条件
- [ ] **F3** API Token 在 Header 和 Query 两种方式 → 均可正常工作
- [ ] **F4** 错误 API Token → 返回 401，不泄露 Token 是否存在
- [ ] **F5** JWT 篡改/过期 → ParseJWT 失败 → 退回到其他认证方式
- [ ] **F6** 发布服务 JWT 脱离发布代理 → 在 Kernel 主端口无法直接使用
- [ ] **F7** Basic Auth WebDAV → 正确凭据通过，错误返回 401

#### G. 漏洞与健壮性测试
- [ ] **G1** 授权码登录 → 测量不同长度输入响应时间，检测时序攻击可能性
- [ ] **G2** 暴力尝试验证码 → 3 次错误后验证码是否每次变化
- [ ] **G3** 超长授权码 (10k 字符) → 不应崩溃/DoS
- [ ] **G4** 特殊字符/SQL 注入在密码字段 → 均被安全处理
- [ ] **G5** 并发大量 CheckAuth 请求 → 无竞态、无死锁
- [ ] **G6** conf.json 设置为只读 → 优雅降级不崩溃
- [ ] **G7** repo 目录损坏 → 检测到错误并提示用户，不丢失源数据

#### H. 加密验证
- [ ] **H1** 导出仓库对象 → 确认不可直接读取（AES-GCM 密文特征）
- [ ] **H2** 两个相同内容文件 → 去重后只存一份 object (内容寻址)
- [ ] **H3** 修改 repo.key → 旧 repo 数据不可读 (正确行为)
- [ ] **H4** 快照 diff → 不泄露任何需要密钥的元信息
- [ ] **H5** 同步配置导出包 → 确认使用 AESEncrypt 且可用 SK 解密
- [ ] **H6** UserData → 确认使用 AESEncrypt 且可用 SK 解密
- [ ] **H7** 发布 Cookie → 抓包验证 HttpOnly + Secure(HTTPS时)

### 13.3 非功能验证

#### I. 性能与资源
- [ ] **I1** 大数据集 (10万文档) Checkout → FullReindex 时间可接受
- [ ] **I2** 1000 次认证失败 → 内存不持续增长 (验证码 GC)
- [ ] **I3** 仓库目录 50GB → Purge/Index 操作不应 OOM
- [ ] **I4** ControlConcurrency 高并发下 → 无请求饿死现象

#### J. 跨平台测试
- [ ] **J1** Windows/Linux/macOS → 文件权限 (repo 目录 0700) 一致
- [ ] **J2** iOS/Android → 移动端访问授权码流程正常
- [ ] **J3** ARM64 / x86_64 → AES-NI 可用性不影响正确性
- [ ] **J4** 同步配置导入导出 → 跨平台兼容 (zip 格式)

---

## 附录：关键语言 ID 速查表 (Language)

| ID | 中文文本 | 用途 |
|----|---------|------|
| 21 | 请输入验证码 | 验证码空提示 |
| 22 | 验证码错误 | 验证码错误提示 |
| 26 | 请先初始化数据仓库 | 密钥未初始化 |
| 29 | 该功能需要订阅付费版 | 订阅检查 |
| 34 | 当前处于只读模式 | 只读拦截 |
| 80 | 数据同步失败：%s | 同步通用失败 |
| 83 | 访问授权码错误 | 授权码错误 |
| 84 | 备份失败：%s | 云端备份失败 |
| 86 | 当前未设置访问授权码 | 空授权码登出 |
| 108 | 数据同步发生冲突 | 冲突提醒 |
| 134 | 数据同步已自动恢复 | 同步恢复提示 |
| 136-148 | 仓库初始化/索引流程消息组 | 仓库操作 |
| 150 | 同步完成 (Ux/Dx 文件...) | 同步成功统计 |
| 151 | 名称无效 | 标签命名错误 |
| 153/152 | 云快照下载/上传完成 | 流量统计 |
| 154 | 云端备份数量超出限制 | 备份数超限 |
| 156 | 请重新登录 | Session 过期 |
| 157 | 导入数据仓库密钥失败 | 密钥格式错误 |
| 194 | 不支持配置坚果云 WebDAV 进行同步 | 坚果云拦截 |
| 202/203 | 清理本地仓库 | 清理进度 |
| 258 | Session 保存失败 | 内部错误 |
| 283 | 请输入访问密码 | 发布文档密码框占位符 |
| 284 | 该文档已被禁止发布 | 禁止文档提示 |
| 286 | 已从快照回滚文件 | 回滚成功提示 |
