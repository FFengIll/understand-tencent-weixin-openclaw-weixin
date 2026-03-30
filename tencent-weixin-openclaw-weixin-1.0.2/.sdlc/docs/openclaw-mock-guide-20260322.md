# OpenClaw 框架模拟完全指南

## 核心发现

插件与 OpenClaw 的交互主要通过 `PluginRuntime["channel"]` 接口，包含 5 个关键模块：

```
channelRuntime
├── routing    (路由解析)
├── session    (会话管理)
├── reply      (回复分发) ← 最核心
├── media      (媒体存储)
└── commands   (命令授权)
```

---

## 一、完整调用链路分析

### 消息处理完整流程 (process-message.ts)

```
┌─────────────────────────────────────────────────────────────────┐
│ 1. 收到微信消息 (WeixinMessage)                                   │
└─────────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│ 2. 转换为 MsgContext (weixinMessageToMsgContext)                │
│    - 提取文本、媒体                                              │
│    - 保存 context_token                                         │
└─────────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│ 3. 命令授权检查 (resolveSenderCommandAuthorizationWithRuntime)  │
│    channelRuntime.commands.resolveSenderCommandAuthorization    │
└─────────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│ 4. 路由解析 (resolveAgentRoute)                                  │
│    channelRuntime.routing.resolveAgentRoute({                   │
│      channel: "openclaw-weixin",                                 │
│      accountId: "xxx",                                           │
│      peer: { kind: "direct", id: userId }                        │
│    })                                                            │
│    → 返回: { agentId, sessionKey, mainSessionKey }              │
└─────────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│ 5. 会话存储路径解析                                              │
│    channelRuntime.session.resolveStorePath(                      │
│      config.session.store, { agentId }                          │
│    )                                                             │
└─────────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│ 6. 完善入站上下文                                                │
│    channelRuntime.reply.finalizeInboundContext(ctx)              │
└─────────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│ 7. 记录入站会话                                                  │
│    channelRuntime.session.recordInboundSession({                 │
│      storePath, sessionKey, ctx,                                 │
│      updateLastRoute: { sessionKey, channel, to, accountId }     │
│    })                                                            │
└─────────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│ 8. 创建回复分发器 (带 typing 指示器)                              │
│    channelRuntime.reply.createReplyDispatcherWithTyping({        │
│      humanDelay,                                                  │
│      typingCallbacks,  // 开始/停止输入状态                       │
│      deliver: async (payload) => { /* 发送到微信 */ },           │
│      onError: (err) => { /* 错误处理 */ }                        │
│    })                                                             │
│    → 返回: { dispatcher, replyOptions, markDispatchIdle }        │
└─────────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│ 9. 执行回复分发                                                  │
│    channelRuntime.reply.withReplyDispatcher({                    │
│      dispatcher,                                                  │
│      run: () =>                                                  │
│        channelRuntime.reply.dispatchReplyFromConfig({            │
│          ctx: finalized,                                         │
│          cfg: config,                                            │
│          dispatcher,                                             │
│          replyOptions                                            │
│        })                                                        │
│    })                                                             │
│                                                                  │
│    ↓ 这个函数内部会调用 AI，然后通过 dispatcher.deliver() 回调  │
│    ↓ 发送回复到微信                                               │
└─────────────────────────────────────────────────────────────────┘
```

---

## 二、最小模拟接口

### 核心接口定义

```typescript
// ========================================
// 类型定义 (基于实际使用推断)
// ========================================

interface MsgContext {
  Body?: string;              // 消息文本
  From: string;               // 发送者 ID
  To: string;                 // 接收者 ID (与 From 相同，私聊)
  AccountId: string;          // 账号 ID
  MessageSid: string;         // 消息唯一 ID
  Timestamp?: number;         // 时间戳
  SessionKey?: string;        // 会话键 (由路由设置)
  context_token?: string;     // 微信 context token
  MediaPath?: string;         // 本地媒体路径
  MediaType?: string;         // 媒体 MIME 类型
  CommandBody?: string;       // 命令原始文本
  CommandAuthorized?: boolean; // 命令授权状态
}

interface AgentRoute {
  agentId?: string;           // AI 代理 ID
  sessionKey?: string;        // 会话键 (如 "main:main")
  mainSessionKey?: string;    // 主会话键
}

interface DeliveryPayload {
  text?: string;              // 回复文本
  mediaUrl?: string;          // 媒体 URL
  mediaUrls?: string[];       // 多媒体 URL
}

interface ReplyDispatcher {
  deliver: (payload: DeliveryPayload) => Promise<void>;
}

interface HumanDelayConfig {
  mode: "auto" | "simulate" | "disabled";
  minMs?: number;
  maxMs?: number;
}

// ========================================
// 需要模拟的完整接口
// ========================================

interface MockChannelRuntime {
  // 1. 路由模块
  routing: {
    resolveAgentRoute(params: {
      cfg: any;
      channel: string;
      accountId: string;
      peer: { kind: "direct" | "group"; id: string };
    }): AgentRoute;
  };

  // 2. 会话模块
  session: {
    resolveStorePath(storeConfig: any, params: { agentId?: string }): string;
    recordInboundSession(params: {
      storePath: string;
      sessionKey?: string;
      ctx: MsgContext;
      updateLastRoute?: {
        sessionKey?: string;
        channel: string;
        to: string;
        accountId: string;
      };
      onRecordError?: (err: Error) => void;
    }): Promise<void>;
  };

  // 3. 回复模块 (最核心)
  reply: {
    finalizeInboundContext(ctx: MsgContext): MsgContext;

    resolveHumanDelayConfig(cfg: any, agentId?: string): HumanDelayConfig;

    createReplyDispatcherWithTyping(params: {
      humanDelay: HumanDelayConfig;
      typingCallbacks: {
        start: () => Promise<void>;
        stop: () => Promise<void>;
        onStartError?: (err: Error) => void;
        onStopError?: (err: Error) => void;
        keepaliveIntervalMs?: number;
      };
      deliver: (payload: DeliveryPayload) => Promise<void>;
      onError: (err: Error, info: { kind: string }) => void;
    }): {
      dispatcher: ReplyDispatcher;
      replyOptions: any;
      markDispatchIdle: () => void;
    };

    dispatchReplyFromConfig(params: {
      ctx: MsgContext;
      cfg: any;
      dispatcher: ReplyDispatcher;
      replyOptions: any;
    }): Promise<void>;

    withReplyDispatcher(params: {
      dispatcher: ReplyDispatcher;
      run: () => Promise<void>;
    }): Promise<void>;
  };

  // 4. 媒体模块
  media: {
    saveMediaBuffer(params: {
      buffer: Buffer;
      accountId: string;
      msgId: string;
      type: string;
    }): Promise<string>;  // 返回保存的文件路径
  };

  // 5. 命令模块
  commands: {
    resolveSenderCommandAuthorizationWithRuntime(params: {
      cfg: any;
      rawBody: string;
      isGroup: boolean;
      dmPolicy: string;
      configuredAllowFrom: string[];
      configuredGroupAllowFrom: string[];
      senderId: string;
      isSenderAllowed: (id: string, list: string[]) => boolean;
      readAllowFromStore: () => Promise<string[]>;
      runtime: any;
    }): Promise<{
      senderAllowedForCommands: boolean;
      commandAuthorized: boolean;
    }>;
  };
}
```

---

## 三、完整模拟实现

```typescript
// ========================================
// mock-openclaw.ts
// ========================================

import path from "node:path";
import fs from "node:fs";
import { fileURLToPath } from "node:url";

const __dirname = path.dirname(fileURLToPath(import.meta.url));

// ==================== 配置 ====================

export const mockConfig = {
  channels: {
    "openclaw-weixin": {
      enabled: true,
    },
  },
  session: {
    store: "~/.openclaw/sessions",
  },
  agents: {
    mode: "per-channel-per-peer",  // 或 "shared"
  },
};

// ==================== 会话存储 (JSON 文件) ====================

class SessionStore {
  private baseDir: string;

  constructor(baseDir = "~/.mock-openclaw/sessions") {
    this.baseDir = baseDir.replace("~", process.env.HOME || "");
  }

  resolvePath(agentId: string, sessionKey?: string): string {
    const dir = path.join(this.baseDir, agentId);
    return sessionKey
      ? path.join(dir, `${sessionKey}.json`)
      : path.join(dir, "main.json");
  }

  async load(agentId: string, sessionKey?: string): Promise<any> {
    const filePath = this.resolvePath(agentId, sessionKey);
    try {
      const raw = await fs.promises.readFile(filePath, "utf-8");
      return JSON.parse(raw);
    } catch {
      return { messages: [] };
    }
  }

  async save(agentId: string, sessionKey: string | undefined, data: any): Promise<void> {
    const filePath = this.resolvePath(agentId, sessionKey);
    const dir = path.dirname(filePath);
    await fs.promises.mkdir(dir, { recursive: true });
    await fs.promises.writeFile(filePath, JSON.stringify(data, null, 2));
  }

  async recordMessage(
    agentId: string,
    sessionKey: string | undefined,
    ctx: MsgContext,
    updateLastRoute?: { channel: string; to: string; accountId: string }
  ): Promise<void> {
    const data = await this.load(agentId, sessionKey);

    // 添加消息
    data.messages.push({
      role: "user",
      content: ctx.Body || "",
      from: ctx.From,
      to: ctx.To,
      timestamp: ctx.Timestamp || Date.now(),
      media: ctx.MediaPath,
    });

    // 更新最后路由
    if (updateLastRoute) {
      data.lastRoute = updateLastRoute;
    }

    await this.save(agentId, sessionKey, data);
  }
}

const sessionStore = new SessionStore();

// ==================== AI 调用 (你的自定义逻辑) ====================

interface AiResponse {
  text: string;
  mediaUrl?: string;
}

async function callYourAI(
  messages: Array<{ role: string; content: string }>,
  userId: string
): Promise<AiResponse> {
  // TODO: 在这里实现你的 AI 调用
  // - 可以调用 OpenAI API
  // - 可以调用本地模型
  // - 可以实现任何自定义逻辑

  console.log(`[AI] 收到 ${messages.length} 条消息，用户: ${userId}`);

  // 示例：简单 echo
  const lastMessage = messages[messages.length - 1]?.content || "";
  return { text: `你说: ${lastMessage}` };
}

// ==================== 完整的 Mock Runtime ====================

export const mockChannelRuntime: MockChannelRuntime = {
  // ------------------- 路由 -------------------
  routing: {
    resolveAgentRoute({ channel, accountId, peer }) {
      // 根据 agent 模式决定 sessionKey
      const mode = mockConfig.agents?.mode || "shared";

      if (mode === "per-channel-per-peer") {
        // 每个微信用户独立会话
        return {
          agentId: "default-agent",
          sessionKey: `main:${channel}:${accountId}:${peer.id}`,
          mainSessionKey: `main:${channel}:${accountId}`,
        };
      } else {
        // 共享会话
        return {
          agentId: "default-agent",
          sessionKey: "main:main",
          mainSessionKey: "main:main",
        };
      }
    },
  },

  // ------------------- 会话 -------------------
  session: {
    resolveStorePath(storeConfig, { agentId }) {
      return sessionStore.resolvePath(agentId || "default");
    },

    async recordInboundSession({ storePath, sessionKey, ctx, updateLastRoute, onRecordError }) {
      try {
        const agentId = "default-agent";
        await sessionStore.recordMessage(agentId, sessionKey, ctx, updateLastRoute);
      } catch (err) {
        onRecordError?.(err as Error);
      }
    },
  },

  // ------------------- 回复 (核心) -------------------
  reply: {
    finalizeInboundContext(ctx) {
      // 可以在这里添加额外的上下文信息
      return ctx;
    },

    resolveHumanDelayConfig(cfg, agentId) {
      // 配置回复延迟模拟
      return {
        mode: "auto",  // 或 "simulate" 模拟人类打字速度
        minMs: 1000,
        maxMs: 3000,
      };
    },

    createReplyDispatcherWithTyping({ humanDelay, typingCallbacks, deliver, onError }) {
      let isIdle = true;

      return {
        dispatcher: {
          async deliver(payload) {
            if (!isIdle) {
              console.warn("[dispatcher] 忙碌中，消息可能乱序");
            }

            isIdle = false;

            try {
              // 1. 开始输入指示
              await typingCallbacks.start();

              // 2. 人类延迟 (可选)
              if (humanDelay.mode === "simulate" && humanDelay.minMs && humanDelay.maxMs) {
                const delay = Math.random() * (humanDelay.maxMs - humanDelay.minMs) + humanDelay.minMs;
                await new Promise(r => setTimeout(r, delay));
              }

              // 3. 发送消息
              await deliver(payload);

            } catch (err) {
              onError(err as Error, { kind: "deliver" });
            } finally {
              // 4. 停止输入指示
              await typingCallbacks.stop();
              isIdle = true;
            }
          },
        },
        replyOptions: {},
        markDispatchIdle() {
          isIdle = true;
        },
      };
    },

    async dispatchReplyFromConfig({ ctx, cfg, dispatcher }) {
      // 这里是 AI 调用的入口点

      // 1. 加载历史会话
      const mode = cfg.agents?.mode || "shared";
      const sessionKey = ctx.SessionKey || "main:main";

      let sessionData: any;
      if (mode === "per-channel-per-peer") {
        sessionData = await sessionStore.load("default-agent", sessionKey);
      } else {
        sessionData = await sessionStore.load("default-agent", "main:main");
      }

      // 2. 构建消息列表
      const messages = (sessionData.messages || []).map((m: any) => ({
        role: m.role,
        content: m.content,
      }));

      // 添加当前消息
      if (ctx.Body) {
        messages.push({
          role: "user",
          content: ctx.Body,
        });
      }

      // 3. 调用 AI
      const response = await callYourAI(messages, ctx.From);

      // 4. 发送回复
      await dispatcher.deliver({
        text: response.text,
        mediaUrl: response.mediaUrl,
      });

      // 5. 保存助手的回复到会话
      if (mode === "per-channel-per-peer") {
        await sessionStore.recordMessage("default-agent", sessionKey, {
          ...ctx,
          Body: response.text,
        } as MsgContext);
      } else {
        await sessionStore.recordMessage("default-agent", "main:main", {
          ...ctx,
          Body: response.text,
        } as MsgContext);
      }
    },

    async withReplyDispatcher({ dispatcher, run }) {
      await run();
    },
  },

  // ------------------- 媒体 -------------------
  media: {
    async saveMediaBuffer({ buffer, accountId, msgId, type }) {
      const dir = path.join("~/.mock-openclaw/media", accountId);
      const realDir = dir.replace("~", process.env.HOME || "");
      await fs.promises.mkdir(realDir, { recursive: true });

      const ext = type === "image" ? "png" : type === "video" ? "mp4" : "bin";
      const fileName = `${msgId}.${ext}`;
      const filePath = path.join(realDir, fileName);

      await fs.promises.writeFile(filePath, buffer);
      return filePath;
    },
  },

  // ------------------- 命令授权 -------------------
  commands: {
    async resolveSenderCommandAuthorizationWithRuntime({
      senderId,
      readAllowFromStore,
    }) {
      // 简单实现：所有用户都授权
      // 你可以根据 readAllowFromStore() 的结果来实现白名单

      const allowFrom = await readAllowFromStore();
      const isAllowed = allowFrom.length === 0 || allowFrom.includes(senderId);

      return {
        senderAllowedForCommands: isAllowed,
        commandAuthorized: isAllowed,
      };
    },
  },
};

// ==================== 完整 Runtime ====================

export const mockRuntime = {
  channel: mockChannelRuntime,
  log: console.log,
  error: console.error,
};

// ==================== 导出设置函数 ====================

export function setupMockRuntime() {
  // 插件会调用 setWeixinRuntime
  return mockRuntime;
}
```

---

## 四、使用插件的 monitor 模块

```typescript
// ========================================
// main.ts
// ========================================

import { setWeixinRuntime } from "./tencent-weixin-openclaw-weixin-1.0.2/package/src/runtime.ts";
import { monitorWeixinProvider } from "./tencent-weixin-openclaw-weixin-1.0.2/package/src/monitor/monitor.ts";
import { mockRuntime, mockConfig } from "./mock-openclaw.js";
import { startWeixinLoginWithQr, waitForWeixinLogin } from "./tencent-weixin-openclaw-weixin-1.0.2/package/src/auth/login-qr.js";
import { saveWeixinAccount, registerWeixinAccountId, normalizeAccountId } from "./tencent-weixin-openclaw-weixin-1.0.2/package/src/auth/accounts.js";

async function main() {
  // 1. 设置 mock runtime
  setWeixinRuntime(mockRuntime);

  // 2. 登录微信
  console.log("开始登录...");
  const startResult = await startWeixinLoginWithQr({
    accountId: "my-account",
    apiBaseUrl: "https://ilinkai.weixin.qq.com",
    botType: "3",
  });

  console.log("扫码:", startResult.qrcodeUrl);

  const waitResult = await waitForWeixinLogin({
    sessionKey: startResult.sessionKey,
    apiBaseUrl: "https://ilinkai.weixin.qq.com",
  });

  console.log("登录成功!");
  console.log("  Token:", waitResult.botToken?.slice(0, 20) + "...");
  console.log("  AccountID:", waitResult.accountId);
  console.log("  UserID:", waitResult.userId);

  // 3. 保存账号
  const normalizedId = normalizeAccountId(waitResult.accountId!);
  saveWeixinAccount(normalizedId, {
    token: waitResult.botToken,
    baseUrl: waitResult.baseUrl,
    userId: waitResult.userId,
  });
  registerWeixinAccountId(normalizedId);

  // 4. 启动监控
  console.log("\n开始监听消息...\n");

  const abortCtrl = new AbortController();

  await monitorWeixinProvider({
    baseUrl: waitResult.baseUrl || "https://ilinkai.weixin.qq.com",
    cdnBaseUrl: "https://novac2c.cdn.weixin.qq.com/c2c",
    token: waitResult.botToken,
    accountId: normalizedId,
    config: mockConfig,
    runtime: mockRuntime,
    abortSignal: abortCtrl.signal,
    setStatus: (status) => {
      console.log("[Status]", status);
    },
  });
}

main().catch(console.error);
```

---

## 五、关键点总结

### 必须实现的接口

| 模块 | 方法 | 重要性 | 说明 |
|------|------|--------|------|
| `reply` | `dispatchReplyFromConfig` | ⭐⭐⭐⭐⭐ | AI 调用入口 |
| `reply` | `createReplyDispatcherWithTyping` | ⭐⭐⭐⭐⭐ | 创建消息发送器 |
| `routing` | `resolveAgentRoute` | ⭐⭐⭐⭐ | 决定会话键 |
| `session` | `recordInboundSession` | ⭐⭐⭐ | 保存会话历史 |
| `reply` | `finalizeInboundContext` | ⭐⭐ | 完善上下文 |
| `media` | `saveMediaBuffer` | ⭐⭐ | 保存媒体文件 |
| `commands` | `resolveSender...` | ⭐ | 命令授权 |

### 简化方案

如果不需要完整功能，可以最小化实现：

```typescript
export const minimalMockRuntime = {
  channel: {
    routing: {
      resolveAgentRoute: () => ({
        agentId: "default",
        sessionKey: "main:main",
      }),
    },
    session: {
      resolveStorePath: () => "/tmp/session.json",
      recordInboundSession: async () => {},
    },
    reply: {
      finalizeInboundContext: (ctx) => ctx,
      resolveHumanDelayConfig: () => ({ mode: "auto" }),
      createReplyDispatcherWithTyping: ({ deliver }) => ({
        dispatcher: { deliver },
        replyOptions: {},
        markDispatchIdle: () => {},
      }),
      dispatchReplyFromConfig: async ({ ctx, dispatcher }) => {
        // 直接在这里实现你的逻辑
        const response = await myAIHandler(ctx.Body);
        await dispatcher.deliver({ text: response });
      },
      withReplyDispatcher: async ({ run }) => run(),
    },
    media: {
      saveMediaBuffer: async () => "/tmp/media",
    },
    commands: {
      resolveSenderCommandAuthorizationWithRuntime: async () => ({
        senderAllowedForCommands: true,
        commandAuthorized: true,
      }),
    },
  },
};
```

### Context Token 处理

插件内部会自动管理 `context_token`，你不需要关心：

```typescript
// 插件在收到消息时自动保存
setContextToken(accountId, userId, contextToken);

// 在发送消息时自动获取
const contextToken = getContextToken(accountId, userId);
```

---

## 六、调试技巧

### 1. 查看调用日志

```typescript
export const debugRuntime = {
  channel: {
    ...mockChannelRuntime,
    reply: {
      ...mockChannelRuntime.reply,
      async dispatchReplyFromConfig(...args) {
        console.log("[dispatchReplyFromConfig] 调用参数:", args);
        return mockChannelRuntime.reply.dispatchReplyFromConfig(...args);
      },
    },
  },
};
```

### 2. 注入 AI 逻辑

```typescript
// 在 mock-openclaw.ts 中修改
async function callYourAI(messages, userId) {
  console.log("=".repeat(50));
  console.log("AI 调用:");
  console.log("  用户:", userId);
  console.log("  消息数:", messages.length);
  console.log("  上下文:");
  messages.slice(-5).forEach(m => {
    console.log(`    [${m.role}] ${m.content.slice(0, 50)}...`);
  });
  console.log("=".repeat(50));

  // 你的 AI 调用
  return { text: "这是 AI 的回复" };
}
```

---

## 七、完整项目结构

```
my-weixin-bot/
├── main.ts                          # 入口文件
├── mock-openclaw.ts                 # Mock 实现
├── package.json
└── tencent-weixin-openclaw-weixin/   # 插件源码
    └── package/src/
        ├── auth/
        ├── api/
        ├── monitor/
        └── ...
```

---

## 八、运行

```bash
npm install
node --loader ts-node/esm main.ts
```
