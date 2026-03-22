# OpenClaw 微信插件技术架构深度解析

**分析日期**: 2026-03-22
**插件版本**: 1.0.2
**分析范围**: 完整代码架构与实现原理

---

## 1. 项目概述

### 1.1 核心定位

这是一个 **OpenClaw 框架的微信渠道插件**，本质上是一个**协议适配器**，将微信的私有协议转换为 OpenClaw 的标准 HTTP JSON API。

```
┌─────────────────────────────────────────────────────────────────┐
│                      OpenClaw 框架层                            │
├─────────────────────────────────────────────────────────────────┤
│         本插件 (协议适配层)                                      │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐          │
│  │   认证模块    │  │  消息处理     │  │   媒体处理    │          │
│  └──────────────┘  └──────────────┘  └──────────────┘          │
├─────────────────────────────────────────────────────────────────┤
│              微信后端网关 API (HTTP JSON)                       │
└─────────────────────────────────────────────────────────────────┘
```

> 补充
> export const DEFAULT_BASE_URL = "https://ilinkai.weixin.qq.com";
> export const CDN_BASE_URL = "https://novac2c.cdn.weixin.qq.com/c2c";

### 1.2 技术栈

| 层级     | 技术                        |
| -------- | --------------------------- |
| 运行时   | Node.js                     |
| 模块系统 | ES Modules                  |
| 插件接口 | OpenClaw Plugin SDK         |
| 通信协议 | HTTP POST + JSON            |
| 加密算法 | AES-128-ECB                 |
| 同步机制 | 长轮询 (Long Polling)       |
| 认证方式 | Bearer Token + X-WECHAT-UIN |

---

## 2. 目录结构

```
package/
├── index.ts                      # 插件入口，注册到 OpenClaw
├── openclaw.plugin.json          # 插件元数据配置
├── package.json                  # NPM 包配置
│
├── src/
│   ├── channel.ts                # 核心渠道实现 (ChannelPlugin 接口)
│   ├── runtime.ts                # 运行时状态管理
│   │
│   ├── auth/                     # 认证与账号管理
│   │   ├── login-qr.ts          # 二维码登录流程
│   │   ├── accounts.ts          # 多账号存储与管理
│   │   └── pairing.ts           # 设备配对逻辑
│   │
│   ├── api/                      # 后端 API 通信层
│   │   ├── types.ts             # 完整的类型定义
│   │   ├── api.ts               # HTTP 请求封装
│   │   ├── config-cache.ts      # 配置缓存 (typing ticket)
│   │   └── session-guard.ts     # 会话状态保护 (过期处理)
│   │
│   ├── messaging/                # 消息处理
│   │   ├── inbound.ts           # 入站消息转换
│   │   ├── send.ts              # 出站文本消息
│   │   ├── send-media.ts        # 出站媒体消息
│   │   ├── process-message.ts   # 消息处理流程
│   │   ├── slash-commands.ts    # 斜杠命令处理
│   │   ├── debug-mode.ts        # 调试模式
│   │   └── error-notice.ts      # 错误通知
│   │
│   ├── monitor/                  # 消息监控循环
│   │   └── monitor.ts           # 长轮询 getUpdates 主循环
│   │
│   ├── cdn/                      # CDN 媒体处理
│   │   ├── aes-ecb.ts           # AES-128-ECB 加密/解密
│   │   ├── cdn-url.ts           # CDN URL 构造
│   │   ├── cdn-upload.ts        # CDN 上传流程
│   │   ├── pic-decrypt.ts       # 图片解密
│   │   └── upload.ts            # 统一上传接口
│   │
│   ├── media/                    # 媒体转码
│   │   ├── media-download.ts    # 媒体下载
│   │   ├── silk-transcode.ts    # SILK 语音转码
│   │   └── mime.ts              # MIME 类型检测
│   │
│   ├── storage/                  # 本地存储
│   │   ├── state-dir.ts         # 状态目录管理
│   │   └── sync-buf.ts          # 同步游标持久化
│   │
│   ├── util/                     # 工具函数
│   │   ├── logger.ts            # 日志系统
│   │   ├── random.ts            # 随机 ID 生成
│   │   └── redact.ts            # 敏感信息脱敏
│   │
│   ├── config/                   # 配置
│   │   └── config-schema.ts     # 配置 Schema 定义
│   │
│   └── log-upload.ts             # 日志上传 CLI
```

---

## 3. 核心架构设计

### 3.1 插件注册流程

**文件**: [`index.ts`](package/index.ts)

```typescript
const plugin = {
  id: "openclaw-weixin",
  name: "Weixin",
  description: "Weixin channel (getUpdates long-poll + sendMessage)",

  register(api: OpenClawPluginApi) {
    // 1. 设置运行时环境
    setWeixinRuntime(api.runtime);

    // 2. 注册渠道插件
    api.registerChannel({ plugin: weixinPlugin });

    // 3. 注册 CLI 命令
    api.registerCli(
      ({ program, config }) => registerWeixinCli({ program, config }),
      { commands: ["openclaw-weixin"] }
    );
  },
};
```

### 3.2 ChannelPlugin 接口实现

**文件**: [`src/channel.ts`](package/src/channel.ts)

```typescript
export const weixinPlugin: ChannelPlugin<ResolvedWeixinAccount> = {
  // === 元数据 ===
  id: "openclaw-weixin",
  meta: {
    selectionLabel: "openclaw-weixin (long-poll)",
    blurb: "getUpdates long-poll upstream, sendMessage downstream; token auth.",
  },

  // === 能力声明 ===
  capabilities: {
    chatTypes: ["direct"],    // 仅支持私聊
    media: true,              // 支持媒体
  },

  // === 配置管理 ===
  config: {
    listAccountIds,           // 列出所有账号
    resolveAccount,           // 解析账号配置
    isConfigured,             // 检查是否已配置
    describeAccount,          // 账号描述
  },

  // === 出站消息 ===
  outbound: {
    deliveryMode: "direct",
    textChunkLimit: 4000,     // 文本分块限制
    sendText,                 // 发送文本
    sendMedia,                // 发送媒体
  },

  // === 认证登录 ===
  auth: {
    login: async ({ cfg, accountId, verbose, runtime }) => {
      // 1. 获取二维码
      const startResult = await startWeixinLoginWithQr(...);

      // 2. 显示二维码
      qrcode-terminal.generate(startResult.qrcodeUrl);

      // 3. 等待扫码确认
      const waitResult = await waitForWeixinLogin(...);

      // 4. 保存凭证
      saveWeixinAccount(normalizedId, {
        token: waitResult.botToken,
        baseUrl: waitResult.baseUrl,
        userId: waitResult.userId,
      });
    },
  },

  // === Gateway 集成 ===
  gateway: {
    startAccount: async (ctx) => {
      // 启动长轮询监控循环
      return monitorWeixinProvider({...});
    },
    loginWithQrStart,   // CLI 登录 - 第一步
    loginWithQrWait,    // CLI 登录 - 第二步
  },
};
```

---

## 4. 认证机制

### 4.1 二维码登录流程

**文件**: [`src/auth/login-qr.ts`](package/src/auth/login-qr.ts)

```typescript
// === 状态机 ===
type LoginStatus = "wait" | "scaned" | "confirmed" | "expired";

// === 登录流程 ===
async function loginFlow() {
  // 1. 请求二维码
  const qrResponse = await fetchQRCode(apiBaseUrl, botType);
  // 返回: { qrcode, qrcode_img_content }

  // 2. 长轮询等待扫码 (35秒超时)
  while (Date.now() < deadline) {
    const status = await pollQRStatus(apiBaseUrl, qrcode);

    switch (status.status) {
      case "wait":
        // 未扫码，继续等待
        break;

      case "scaned":
        // 已扫码，等待确认
        break;

      case "expired":
        // 二维码过期，自动刷新 (最多3次)
        qrResponse = await fetchQRCode(...);
        break;

      case "confirmed":
        // 确认成功，返回凭证
        return {
          connected: true,
          botToken: status.bot_token,
          accountId: status.ilink_bot_id,
          baseUrl: status.baseurl,
          userId: status.ilink_user_id,
        };
    }
  }
}
```

### 4.2 多账号管理

**文件**: [`src/auth/accounts.ts`](package/src/auth/accounts.ts)

```typescript
// === 账号存储结构 ===
~/.openclaw/openclaw-weixin/accounts/
├── {accountId}.json          # 账号凭证
└── {accountId}.sync.json     # 同步游标

// === 账号数据结构 ===
interface WeixinAccountData {
  token: string;        // Bearer Token
  baseUrl: string;      // API Base URL
  userId: string;       // 用户 ID
}

// === 账号 ID 规范化 ===
// 原始 ID: "b0f5860fdecb@im.bot"
// 规范化: "b0f5860fdecb-im-bot" (文件系统安全)
function normalizeAccountId(rawId: string): string {
  return rawId.replace(/[^a-zA-Z0-9-]/g, "-");
}
```

---

## 5. 消息同步机制

### 5.1 长轮询 (Long Polling)

**文件**: [`src/monitor/monitor.ts`](package/src/monitor/monitor.ts)

```typescript
// === 主循环 ===
export async function monitorWeixinProvider(opts: MonitorWeixinOpts) {
  const syncFilePath = getSyncBufFilePath(accountId);
  let getUpdatesBuf = loadGetUpdatesBuf(syncFilePath) ?? "";
  let nextTimeoutMs = 35_000;  // 默认 35 秒

  while (!abortSignal?.aborted) {
    // 1. 发起长轮询
    const resp = await getUpdates({
      baseUrl,
      token,
      get_updates_buf: getUpdatesBuf,
      timeoutMs: nextTimeoutMs,
    });

    // 2. 处理响应
    if (resp.ret === 0) {
      // 成功：更新游标
      getUpdatesBuf = resp.get_updates_buf;
      saveGetUpdatesBuf(syncFilePath, getUpdatesBuf);

      // 处理新消息
      for (const msg of resp.msgs) {
        await processOneMessage(msg, {...});
      }
    } else if (resp.errcode === -14) {
      // 会话过期：暂停 30 分钟
      pauseSession(accountId);
      await sleep(30 * 60_000);
    }

    // 3. 更新超时时间 (服务端建议)
    if (resp.longpolling_timeout_ms) {
      nextTimeoutMs = resp.longpolling_timeout_ms;
    }
  }
}
```

### 5.2 同步游标 (Sync Buffer)

**文件**: [`src/storage/sync-buf.ts`](package/src/storage/sync-buf.ts)

```typescript
// === 游标作用 ===
// get_updates_buf 是一个服务端维护的同步状态标识符
// - 首次请求: 传空字符串 ""
// - 后续请求: 传上次响应返回的 get_updates_buf
// - 作用: 服务端根据此游标返回增量消息

// === 持久化路径 ===
~/.openclaw/openclaw-weixin/accounts/{accountId}.sync.json

// === 数据结构 ===
{
  "get_updates_buf": "base64_encoded_sync_state"
}
```

---

## 6. API 通信层

### 6.1 统一请求封装

**文件**: [`src/api/api.ts`](package/src/api/api.ts)

```typescript
// === 请求头构造 ===
function buildHeaders(opts: { token?: string; body: string }) {
  return {
    "Content-Type": "application/json",
    "AuthorizationType": "ilink_bot_token",
    "Authorization": `Bearer ${token}`,
    "X-WECHAT-UIN": randomWechatUin(),  // 随机 uint32 base64
    "SKRouteTag": loadConfigRouteTag(), // 可选路由标签
  };
}

// === 通用 Fetch ===
async function apiFetch(params) {
  const controller = new AbortController();
  const timer = setTimeout(() => controller.abort(), timeoutMs);

  const res = await fetch(url, {
    method: "POST",
    headers: buildHeaders(...),
    body: JSON.stringify(body),
    signal: controller.signal,
  });

  clearTimeout(timer);
  return res.text();
}
```

### 6.2 核心 API 端点

| 端点                     | 超时 | 用途           |
| ------------------------ | ---- | -------------- |
| `ilink/bot/getupdates`   | 35s  | 长轮询获取消息 |
| `ilink/bot/sendmessage`  | 15s  | 发送消息       |
| `ilink/bot/getuploadurl` | 15s  | 获取上传 URL   |
| `ilink/bot/getconfig`    | 10s  | 获取配置       |
| `ilink/bot/sendtyping`   | 10s  | 发送输入状态   |

---

## 7. 媒体处理

### 7.1 AES-128-ECB 加密

**文件**: [`src/cdn/aes-ecb.ts`](package/src/cdn/aes-ecb.ts)

```typescript
// === 加密 ===
export function encryptAesEcb(plaintext: Buffer, key: Buffer): Buffer {
  const cipher = createCipheriv("aes-128-ecb", key, null);
  return Buffer.concat([cipher.update(plaintext), cipher.final()]);
}

// === 解密 ===
export function decryptAesEcb(ciphertext: Buffer, key: Buffer): Buffer {
  const decipher = createDecipheriv("aes-128-ecb", key, null);
  return Buffer.concat([decipher.update(ciphertext), decipher.final()]);
}

// === PKCS7 填充计算 ===
export function aesEcbPaddedSize(plaintextSize: number): number {
  return Math.ceil((plaintextSize + 1) / 16) * 16;
}
```

### 7.2 媒体上传流程

**文件**: [`src/cdn/upload.ts`](package/src/cdn/upload.ts)

```typescript
// === 完整流程 ===
async function uploadMedia() {
  // 1. 计算文件参数
  const rawMd5 = md5File(filePath);
  const rawSize = fs.statSync(filePath).size;
  const aesKey = randomBytes(16);
  const ciphertextSize = aesEcbPaddedSize(rawSize);

  // 2. 获取上传 URL
  const uploadResp = await getUploadUrl({
    filekey,
    media_type: 1,  // IMAGE
    rawsize: rawSize,
    rawfilemd5: rawMd5,
    filesize: ciphertextSize,
    aeskey: aesKey.toString('base64'),
  });

  // 3. 加密文件
  const plaintext = fs.readFileSync(filePath);
  const ciphertext = encryptAesEcb(plaintext, aesKey);

  // 4. 上传到 CDN
  await fetch(uploadResp.upload_param, {
    method: "PUT",
    body: ciphertext,
  });

  // 5. 返回 CDN 引用
  return {
    filekey,
    fileSize: rawSize,
    aesKey,
    encryptQueryParam: uploadResp.upload_param,
  };
}
```

### 7.3 媒体类型路由

**文件**: [`src/messaging/send-media.ts`](package/src/messaging/send-media.ts)

```typescript
// === MIME 类型路由 ===
export async function sendWeixinMediaFile(params) {
  const mime = getMimeFromFilename(filePath);

  if (mime.startsWith("video/")) {
    return uploadVideoToWeixin(...) + sendVideoMessageWeixin(...);
  }

  if (mime.startsWith("image/")) {
    return uploadFileToWeixin(...) + sendImageMessageWeixin(...);
  }

  // 其他类型：文件附件
  return uploadFileAttachmentToWeixin(...) + sendFileMessageWeixin(...);
}
```

---

## 8. 消息类型系统

### 8.1 消息结构

**文件**: [`src/api/types.ts`](package/src/api/types.ts)

```typescript
// === 消息类型 ===
const MessageType = {
  USER: 1,    // 用户消息
  BOT: 2,     // 机器人消息
};

const MessageItemType = {
  TEXT: 1,    // 文本
  IMAGE: 2,   // 图片
  VOICE: 3,   // 语音
  FILE: 4,    // 文件
  VIDEO: 5,   // 视频
};

// === 消息状态 ===
const MessageState = {
  NEW: 0,        // 新消息
  GENERATING: 1, // 生成中
  FINISH: 2,     // 已完成
};

// === 统一消息结构 ===
interface WeixinMessage {
  seq?: number;                // 序列号
  message_id?: number;         // 消息 ID
  from_user_id?: string;       // 发送者 ID
  to_user_id?: string;         // 接收者 ID
  create_time_ms?: number;     // 创建时间
  session_id?: string;         // 会话 ID
  message_type?: number;       // 消息类型
  message_state?: number;      // 消息状态
  item_list?: MessageItem[];   // 内容列表
  context_token?: string;      // 会话令牌 (回复时必须回传)
}

// === 消息项 ===
interface MessageItem {
  type?: number;
  text_item?: TextItem;
  image_item?: ImageItem;
  voice_item?: VoiceItem;
  file_item?: FileItem;
  video_item?: VideoItem;
  ref_msg?: RefMessage;  // 引用消息
}
```

### 8.2 Context Token 机制

**文件**: [`src/messaging/inbound.ts`](package/src/messaging/inbound.ts)

```typescript
// === Context Token 作用 ===
// 1. 服务端为每条消息生成唯一的 context_token
// 2. 回复时必须回传此 token
// 3. 用于服务端维护会话上下文

// === 内存存储 ===
const contextTokenStore = new Map<string, string>();

function contextTokenKey(accountId: string, userId: string) {
  return `${accountId}:${userId}`;
}

// === 入站：保存 token ===
export function setContextToken(accountId: string, userId: string, token: string) {
  contextTokenStore.set(contextTokenKey(accountId, userId), token);
}

// === 出站：获取 token ===
export function getContextToken(accountId: string, userId: string) {
  return contextTokenStore.get(contextTokenKey(accountId, userId));
}
```

---

## 9. 会话保护机制

### 9.1 会话过期处理

**文件**: [`src/api/session-guard.ts`](package/src/api/session-guard.ts)

```typescript
const SESSION_EXPIRED_ERRCODE = -14;
const PAUSE_DURATION_MS = 30 * 60_000;  // 30 分钟

const sessionPauseTimers = new Map<string, number>();

// === 暂停会话 ===
export function pauseSession(accountId: string) {
  const expiresAt = Date.now() + PAUSE_DURATION_MS;
  sessionPauseTimers.set(accountId, expiresAt);
}

// === 检查是否暂停中 ===
export function isSessionPaused(accountId: string): boolean {
  const expiresAt = sessionPauseTimers.get(accountId);
  if (!expiresAt) return false;

  if (Date.now() > expiresAt) {
    sessionPauseTimers.delete(accountId);
    return false;
  }

  return true;
}

// === 在 monitor.ts 中的使用 ===
if (resp.errcode === SESSION_EXPIRED_ERRCODE) {
  pauseSession(accountId);
  const pauseMs = getRemainingPauseMs(accountId);
  errLog(`Session expired, pausing for ${pauseMs / 60_000} min`);
  await sleep(pauseMs);
}
```

---

## 10. 错误处理与重试

### 10.1 指数退避

**文件**: [`src/monitor/monitor.ts`](package/src/monitor/monitor.ts)

```typescript
const MAX_CONSECUTIVE_FAILURES = 3;
const RETRY_DELAY_MS = 2_000;
const BACKOFF_DELAY_MS = 30_000;

let consecutiveFailures = 0;

while (!abortSignal?.aborted) {
  try {
    const resp = await getUpdates(...);
    consecutiveFailures = 0;  // 成功，重置计数
  } catch (err) {
    consecutiveFailures++;

    if (consecutiveFailures >= MAX_CONSECUTIVE_FAILURES) {
      // 连续失败，退避 30 秒
      await sleep(BACKOFF_DELAY_MS);
      consecutiveFailures = 0;
    } else {
      // 短暂重试
      await sleep(RETRY_DELAY_MS);
    }
  }
}
```

---

## 11. 配置缓存

### 11.1 Typing Ticket 缓存

**文件**: [`src/api/config-cache.ts`](package/src/api/config-cache.ts)

```typescript
// === Config Manager ===
export class WeixinConfigManager {
  private cache = new Map<string, CachedConfig>();

  async getForUser(
    userId: string,
    contextToken?: string
  ): Promise<CachedConfig> {
    const key = contextToken || userId;
    let cached = this.cache.get(key);

    if (!cached || this.isExpired(cached)) {
      // 调用 getConfig 获取新配置
      const resp = await getConfig({
        baseUrl: this.opts.baseUrl,
        token: this.opts.token,
        ilinkUserId: userId,
        contextToken,
      });

      cached = {
        typingTicket: resp.typing_ticket,
        expiresAt: Date.now() + this.CACHE_TTL_MS,
      };
      this.cache.set(key, cached);
    }

    return cached;
  }
}
```

---

## 12. 日志系统

### 12.1 结构化日志

**文件**: [`src/util/logger.ts`](package/src/util/logger.ts)

```typescript
// === 日志级别 ===
enum LogLevel {
  DEBUG = 0,
  INFO = 1,
  WARN = 2,
  ERROR = 3,
}

// === 日志文件路径 ===
~/.openclaw/openclaw-weixin/logs/{accountId}.log

// === 账号日志上下文 ===
class Logger {
  withAccount(accountId: string): Logger {
    // 返回带有账号上下文的新 Logger
    // 所有日志自动包含 accountId 标签
  }
}
```

### 12.2 敏感信息脱敏

**文件**: [`src/util/redact.ts`](package/src/util/redact.ts)

```typescript
// === Token 脱敏 ===
export function redactToken(token?: string): string {
  if (!token) return "(empty)";
  if (token.length <= 8) return "***";
  return `${token.slice(0, 4)}...${token.slice(-4)}`;
}

// === URL 脱敏 ===
export function redactUrl(url: string): string {
  return url.replace(/token=[^&]+/g, "token=***");
}

// === 请求体脱敏 ===
export function redactBody(body: string): string {
  return body
    .replace(/"token":"[^"]+"/g, '"token":"***"')
    .replace(/"typing_ticket":"[^"]+"/g, '"typing_ticket":"***"');
}
```

---

## 13. 数据流图

### 13.1 入站消息流

```
┌─────────────┐    getUpdates    ┌──────────────┐
│   微信服务器│ ←──────────────── │ Gateway 进程 │
└─────────────┘    (long-poll)   └──────────────┘
       │                              │
       │ 新消息                        │
       ├─────────────────────────────>│
       │                              │
       │  WeixinMessage {             │
       │    from_user_id,             │
       │    item_list [...],          │
       │    context_token,            │
       │  }                           │
       │                              │
       │                              │ 1. 解析消息
       │                              │
       │                              │ 2. 下载媒体 (如有)
       │                              │    - CDN 下载
       │                              │    - AES 解密
       │                              │
       │                              │ 3. 转换为 MsgContext
       │                              │
       │                              │ 4. 保存 context_token
       │                              │
       │                              │ 5. 提交给 AI 处理
       │                              │
```

### 13.2 出站消息流

```
┌──────────────┐                     ┌─────────────┐
│ Gateway 进程 │                     │  微信服务器  │
└──────────────┘                     └─────────────┘
       │                                    │
       │ AI 生成回复                         │
       │                                    │
       │ 1. 获取 context_token              │
       │    (从内存缓存)                    │
       │                                    │
       │ 2. 文本消息 → sendMessage          │
       │                                    │
       │ 3. 媒体消息:                       │
       │    - 读取本地文件                  │
       │    - AES-128-ECB 加密             │
       │    - getUploadUrl 获取 URL        │
       │    - PUT 上传到 CDN               │
       │    - sendMessage 发送引用         │
       │                                    │
       ├──────────────────────────────────>│
       │                                    │
```

---

## 14. 关键设计模式

### 14.1 游标模式 (Cursor Pattern)

```typescript
// === 问题 ===
// 如何在不重复获取消息的情况下实现增量同步？

// === 解决方案 ===
// 服务端维护一个同步状态 (get_updates_buf)
// 客户端每次请求带上上次返回的游标
// 服务端返回该游标之后的新消息

let cursor = loadCursor();
while (true) {
  const resp = await getUpdates({ cursor });
  cursor = resp.next_cursor;
  saveCursor(cursor);

  for (const msg of resp.messages) {
    process(msg);
  }
}
```

### 14.2 令牌桶模式 (Token Bucket)

```typescript
// === 问题 ===
// 如何防止会话过期后频繁重试造成服务端压力？

// === 解决方案 ===
// 使用暂停令牌，会话过期后设置过期时间
// 在过期前跳过所有请求

const pauseTokens = new Map<string, number>();

if (isSessionExpired) {
  pauseTokens.set(accountId, Date.now() + 30 * 60_000);
}

function canRequest(accountId: string) {
  const expiresAt = pauseTokens.get(accountId);
  return !expiresAt || Date.now() > expiresAt;
}
```

### 14.3 适配器模式 (Adapter Pattern)

```typescript
// === 问题 ===
// 微信协议与 OpenClaw 框架接口不兼容

// === 解决方案 ===
// ChannelPlugin 接口作为适配器
// 将微信特有的协议转换为框架标准接口

export const weixinPlugin: ChannelPlugin = {
  // 框架接口
  outbound: {
    sendText: async (ctx) => {
      // 转换为微信协议
      return sendMessageWeixin({...});
    },
  },

  // 框架接口
  inbound: {
    // 微信协议 → 框架格式
    weixinMessageToMsgContext(msg, accountId) {
      return { Body, From, To, ... };
    },
  },
};
```

---

## 15. 安全机制

### 15.1 认证链

```
┌─────────────────────────────────────────────────────────────┐
│ 1. 扫码登录                                                 │
│    - 生成临时二维码                                          │
│    - 用户扫码确认                                           │
│    - 获取 bot_token 和 ilink_bot_id                         │
├─────────────────────────────────────────────────────────────┤
│ 2. Token 存储                                               │
│    - 本地文件加密存储 (~/.openclaw/openclaw-weixin/)         │
│    - 文件权限 0600                                          │
├─────────────────────────────────────────────────────────────┤
│ 3. API 请求认证                                             │
│    - Authorization: Bearer <token>                          │
│    - X-WECHAT-UIN: 随机标识符                               │
│    - AuthorizationType: ilink_bot_token                     │
├─────────────────────────────────────────────────────────────┤
│ 4. 媒体加密                                                 │
│    - AES-128-ECB 加密所有上传媒体                           │
│    - 每个文件使用独立密钥                                   │
│    - CDN 仅存储加密内容                                     │
└─────────────────────────────────────────────────────────────┘
```

### 15.2 会话过期保护

```typescript
// === 问题 ===
// 当会话过期时，频繁重试可能导致账号被封禁

// === 解决方案 ===
// 检测到过期错误 (errcode = -14) 后，暂停所有请求 30 分钟

const SESSION_EXPIRED_ERRCODE = -14;
const PAUSE_DURATION_MS = 30 * 60_000;

if (resp.errcode === SESSION_EXPIRED_ERRCODE) {
  pauseSession(accountId);
  // 等待 30 分钟后自动恢复
}
```

---

## 16. 多账号隔离

### 16.1 配置隔离

```bash
# 每个账号独立配置文件
~/.openclaw/openclaw-weixin/accounts/
├── account1.json          # 账号1 凭证
├── account1.sync.json     # 账号1 同步游标
├── account2.json          # 账号2 凭证
└── account2.sync.json     # 账号2 同步游标
```

### 16.2 运行时隔离

```typescript
// === 每个账号独立的监控循环 ===
for (const account of accounts) {
  monitorWeixinProvider({
    accountId: account.accountId,
    baseUrl: account.baseUrl,
    token: account.token,
    abortSignal: account.abortSignal,
  });
}

// === 每个账号独立的 context_token 缓存 ===
const contextTokenStore = new Map<string, string>();
// key: `${accountId}:${userId}`
```

---

## 17. 性能优化

### 17.1 长轮询超时动态调整

```typescript
// === 服务端建议的超时时间 ===
const resp = await getUpdates({
  timeoutMs: nextTimeoutMs,
});

// 更新下次轮询的超时时间
if (resp.longpolling_timeout_ms) {
  nextTimeoutMs = resp.longpolling_timeout_ms;
  // 服务端可以根据网络状况调整此值
  // 减少无效请求，节省资源
}
```

### 17.2 配置缓存

```typescript
// === Typing Ticket 缓存 ===
// 有效期：5 分钟
// 避免每次发送消息都调用 getConfig

class WeixinConfigManager {
  private CACHE_TTL_MS = 5 * 60_000;
  private cache = new Map<string, CachedConfig>();

  async getForUser(userId, contextToken) {
    const cached = this.cache.get(key);
    if (cached && !this.isExpired(cached)) {
      return cached;
    }
    // 缓存过期，重新获取
    return await this.fetchAndCache(userId, contextToken);
  }
}
```

---

## 18. 总结

### 18.1 架构优势

| 特性     | 实现方式                         |
| -------- | -------------------------------- |
| 可靠性   | 长轮询 + 游标持久化 + 断线重连   |
| 可扩展性 | 多账号支持 + 独立监控循环        |
| 安全性   | Token 认证 + AES 加密 + 会话保护 |
| 可维护性 | 模块化设计 + 清晰的类型定义      |
| 可观测性 | 结构化日志 + 状态上报            |

### 18.2 关键技术点

1. **长轮询**: 服务端推送消息的高效方式，避免频繁轮询
2. **游标同步**: 增量获取消息，避免重复处理
3. **Context Token**: 会话上下文令牌，确保回复与对话关联
4. **AES-128-ECB**: 媒体文件加密，保护用户隐私
5. **会话保护**: 过期暂停机制，防止账号被封禁
6. **多账号隔离**: 独立配置、独立运行时、独立状态

### 18.3 适用场景

- ✅ 私人助手：个人微信接入 AI
- ✅ 客服机器人：企业微信客服自动化
- ✅ 消息推送：系统通知通过微信发送
- ❌ 群聊操作：当前版本仅支持私聊
- ❌ 高频发送：有速率限制，需控制发送频率

---

## 附录：文件索引

| 文件                                                     | 行数 | 功能         |
| -------------------------------------------------------- | ---- | ------------ |
| [index.ts](package/index.ts)                             | 28   | 插件入口     |
| [channel.ts](package/src/channel.ts)                     | 381  | 核心渠道实现 |
| [api/types.ts](package/src/api/types.ts)                 | 223  | 类型定义     |
| [api/api.ts](package/src/api/api.ts)                     | 241  | HTTP 通信    |
| [auth/login-qr.ts](package/src/auth/login-qr.ts)         | 334  | 二维码登录   |
| [monitor/monitor.ts](package/src/monitor/monitor.ts)     | 222  | 长轮询循环   |
| [messaging/inbound.ts](package/src/messaging/inbound.ts) | 172  | 入站消息     |
| [cdn/aes-ecb.ts](package/src/cdn/aes-ecb.ts)             | 22   | 加密解密     |
