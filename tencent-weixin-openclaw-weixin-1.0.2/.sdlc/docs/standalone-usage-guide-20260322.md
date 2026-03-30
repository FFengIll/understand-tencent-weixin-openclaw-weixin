# 微信插件独立使用指南

## 问题：如何不使用 OpenClaw 框架直接使用微信插件？

## 答案：三种方案

---

## 方案 A：直接实现微信协议（推荐，完全独立）

### 优点
- 完全控制，无依赖
- 轻量级，只需 HTTP 请求
- 可自由定制

### 核心实现

```typescript
// weixin-client.ts

const DEFAULT_BASE_URL = "https://ilinkai.weixin.qq.com";

// ==================== 1. 扫码登录 ====================

interface LoginResult {
  token: string;
  accountId: string;
  baseUrl: string;
  userId: string;
}

async function loginWithQr(): Promise<LoginResult> {
  // 1. 获取二维码
  const qrResp = await fetch(`${DEFAULT_BASE_URL}/ilink/bot/get_bot_qrcode?bot_type=3`);
  const { qrcode, qrcode_img_content } = await qrResp.json();

  console.log("请扫描二维码:");
  console.log(qrcode_img_content);

  // 2. 长轮询等待扫码
  const deadline = Date.now() + 8 * 60 * 1000; // 8分钟超时

  while (Date.now() < deadline) {
    const statusResp = await fetch(
      `${DEFAULT_BASE_URL}/ilink/bot/get_qrcode_status?qrcode=${qrcode}`
    );
    const status = await statusResp.json();

    switch (status.status) {
      case "wait":
        console.log(".");
        break;
      case "scaned":
        console.log("已扫码，等待确认...");
        break;
      case "confirmed":
        return {
          token: status.bot_token,
          accountId: status.ilink_bot_id,
          baseUrl: status.baseurl,
          userId: status.ilink_user_id,
        };
      case "expired":
        throw new Error("二维码已过期");
    }

    await new Promise(r => setTimeout(r, 2000));
  }

  throw new Error("登录超时");
}

// ==================== 2. 长轮询接收消息 ====================

interface WeixinMessage {
  from_user_id?: string;
  to_user_id?: string;
  item_list?: MessageItem[];
  context_token?: string;
  create_time_ms?: number;
}

interface MessageItem {
  type?: number;
  text_item?: { text?: string };
}

interface ContextTokenStore {
  get(accountId: string, userId: string): string | undefined;
  set(accountId: string, userId: string, token: string): void;
}

// Context Token 内存存储
class MemoryContextTokenStore implements ContextTokenStore {
  private store = new Map<string, string>();

  private key(accountId: string, userId: string) {
    return `${accountId}:${userId}`;
  }

  get(accountId: string, userId: string): string | undefined {
    return this.store.get(this.key(accountId, userId));
  }

  set(accountId: string, userId: string, token: string): void {
    this.store.set(this.key(accountId, userId), token);
  }
}

async function receiveMessages(
  token: string,
  accountId: string,
  onMessage: (msg: WeixinMessage) => void,
  signal?: AbortSignal
): Promise<never> {
  let syncBuf = "";
  const contextStore = new MemoryContextTokenStore();

  while (!signal?.aborted) {
    try {
      const resp = await fetch(`${DEFAULT_BASE_URL}/ilink/bot/getupdates`, {
        method: "POST",
        headers: {
          "Content-Type": "application/json",
          "Authorization": `Bearer ${token}`,
          "AuthorizationType": "ilink_bot_token",
          "X-WECHAT-UIN": randomUin(),
        },
        body: JSON.stringify({
          get_updates_buf: syncBuf,
        }),
      });

      const data = await resp.json();

      // 检查错误
      if (data.ret !== 0) {
        if (data.errcode === -14) {
          console.error("会话过期，需要重新登录");
          throw new Error("Session expired");
        }
        console.error(`getUpdates error: ${data.errmsg}`);
        await new Promise(r => setTimeout(r, 5000));
        continue;
      }

      // 更新同步游标
      syncBuf = data.get_updates_buf || syncBuf;

      // 处理消息
      for (const msg of data.msgs || []) {
        // 保存 context_token
        if (msg.context_token && msg.from_user_id) {
          contextStore.set(accountId, msg.from_user_id, msg.context_token);
        }

        onMessage(msg);
      }

    } catch (err) {
      if (signal?.aborted) break;
      console.error(`接收消息失败: ${err}`);
      await new Promise(r => setTimeout(r, 5000));
    }
  }

  throw new Error("Aborted");
}

function randomUin(): string {
  const uint32 = Math.floor(Math.random() * 2**32);
  return Buffer.from(String(uint32)).toString("base64");
}

// ==================== 3. 发送消息 ====================

async function sendText(
  token: string,
  to: string,
  text: string,
  contextToken: string
): Promise<void> {
  await fetch(`${DEFAULT_BASE_URL}/ilink/bot/sendmessage`, {
    method: "POST",
    headers: {
      "Content-Type": "application/json",
      "Authorization": `Bearer ${token}`,
      "AuthorizationType": "ilink_bot_token",
    },
    body: JSON.stringify({
      msg: {
        to_user_id: to,
        context_token: contextToken,
        item_list: [
          { type: 1, text_item: { text } }
        ],
      },
    }),
  });
}

// ==================== 4. 完整使用示例 ====================

async function main() {
  // 1. 登录
  const { token, accountId, userId } = await loginWithQr();
  console.log(`登录成功! accountId=${accountId}`);

  // 2. 启动消息接收循环
  const abortCtrl = new AbortController();

  receiveMessages(token, accountId, async (msg) => {
    const from = msg.from_user_id;
    const text = msg.item_list?.[0]?.text_item?.text;

    console.log(`收到消息: ${from}: ${text}`);

    // 3. 回复消息
    if (text && from && msg.context_token) {
      await sendText(token, from, `你说的是: ${text}`, msg.context_token);
    }
  }, abortCtrl.signal);
}

// 运行
// main().catch(console.error);
```

---

## 方案 B：复用插件独立模块

### 可独立使用的模块

以下模块**不依赖 OpenClaw SDK**，可以直接引入：

```typescript
// 直接从插件源码引入
import {
  startWeixinLoginWithQr,
  waitForWeixinLogin,
} from "./tencent-weixin-openclaw-weixin-1.0.2/package/src/auth/login-qr.ts";

import {
  getUpdates,
  sendMessage,
  getUploadUrl,
} from "./tencent-weixin-openclaw-weixin-1.0.2/package/src/api/api.ts";

import {
  encryptAesEcb,
  decryptAesEcb,
} from "./tencent-weixin-openclaw-weixin-1.0.2/package/src/cdn/aes-ecb.ts";
```

### 示例代码

```typescript
import { startWeixinLoginWithQr, waitForWeixinLogin } from "./src/auth/login-qr.js";
import { getUpdates } from "./src/api/api.js";

async function standaloneWeixin() {
  // 1. 使用插件的登录模块
  const startResult = await startWeixinLoginWithQr({
    accountId: "my-account",
    apiBaseUrl: "https://ilinkai.weixin.qq.com",
    botType: "3",
  });

  console.log("扫码:", startResult.qrcodeUrl);

  // 2. 等待登录完成
  const waitResult = await waitForWeixinLogin({
    sessionKey: startResult.sessionKey,
    apiBaseUrl: "https://ilinkai.weixin.qq.com",
  });

  console.log("Token:", waitResult.botToken);

  // 3. 使用插件的 API 模块接收消息
  let syncBuf = "";
  while (true) {
    const resp = await getUpdates({
      baseUrl: "https://ilinkai.weixin.qq.com",
      token: waitResult.botToken,
      get_updates_buf: syncBuf,
    });

    syncBuf = resp.get_updates_buf;

    for (const msg of resp.msgs) {
      console.log("消息:", msg);
    }
  }
}
```

---

## 方案 C：模拟 OpenClaw 最小接口

如果需要使用插件的完整功能（如媒体上传、消息处理等），可以模拟 OpenClaw 的最小接口：

```typescript
// mock-openclaw.ts

// 模拟配置
export const mockConfig = {
  channels: {},
  session: { store: "~/.openclaw/sessions" },
};

// 模拟运行时
export const mockRuntime = {
  channel: {
    routing: {
      resolveAgentRoute: ({ channel, accountId, peer }) => ({
        agentId: "default-agent",
        sessionKey: "main:main",
      }),
    },
    session: {
      resolveStorePath: (store, { agentId }) => `~/.openclaw/sessions/${agentId}`,
      recordInboundSession: async ({ storePath, sessionKey, ctx }) => {},
    },
    reply: {
      finalizeInboundContext: (ctx) => ctx,
      createReplyDispatcherWithTyping: ({ humanDelay, typingCallbacks, deliver, onError }) => ({
        dispatcher: { deliver },
        replyOptions: {},
        markDispatchIdle: () => {},
      }),
      dispatchReplyFromConfig: async ({ ctx, cfg, dispatcher, replyOptions }) => {
        // 这里调用你的 AI 处理逻辑
        await handleAiResponse(ctx, dispatcher);
      },
      resolveHumanDelayConfig: (cfg, agentId) => ({ mode: "auto" }),
      withReplyDispatcher: async ({ dispatcher, run }) => run(),
    },
    media: {
      saveMediaBuffer: async (buf, { accountId, msgId, type }) => `/tmp/media/${msgId}`,
    },
    commands: {
      resolveSenderCommandAuthorizationWithRuntime: async () => ({
        senderAllowedForCommands: true,
        commandAuthorized: true,
      }),
    },
  },
};

async function handleAiResponse(ctx, dispatcher) {
  // 你的 AI 处理逻辑
  const response = await callYourAI(ctx.Body);

  await dispatcher.deliver({
    text: response,
    mediaUrl: null,
  });
}
```

然后使用插件的 monitor 模块：

```typescript
import { monitorWeixinProvider } from "./src/monitor/monitor.ts";
import { setWeixinRuntime } from "./src/runtime.ts";

// 设置模拟运行时
setWeixinRuntime(mockRuntime);

// 启动监控
monitorWeixinProvider({
  baseUrl: "https://ilinkai.weixin.qq.com",
  token: botToken,
  accountId: "my-account",
  config: mockConfig,
  runtime: mockRuntime,
});
```

---

## 方案对比

| 方案 | 优点 | 缺点 | 推荐度 |
|------|------|------|--------|
| **A: 直接实现** | 完全独立，轻量 | 需要自己处理细节 | ⭐⭐⭐⭐⭐ |
| **B: 复用模块** | 利用已有代码 | 仍需自己组装流程 | ⭐⭐⭐⭐ |
| **C: 模拟接口** | 功能完整 | 依赖复杂，容易出问题 | ⭐⭐ |

---

## 推荐方案

对于大多数场景，**推荐方案 A**：

1. **简单场景**（只需收发文本）：直接用方案 A 的代码
2. **需要媒体**：复用插件的 `cdn/` 模块处理加密和上传
3. **完整功能**：考虑直接使用 OpenClaw 框架

---

## 关键点

1. **Token 管理**：登录后获取的 token 需要持久化保存
2. **Context Token**：每条消息的 `context_token` 必须保存，回复时需要回传
3. **同步游标**：`get_updates_buf` 用于增量同步，需要持久化
4. **会话过期**：错误码 `-14` 表示会话过期，需要暂停一段时间后重试

---

## 完整独立客户端示例

见方案 A 中的完整代码，可以直接运行。
