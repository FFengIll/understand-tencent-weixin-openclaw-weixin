# Golang 直接启动 Node.js 插件方案

## 核心发现

插件只需要两件事：
1. 调用 `setWeixinRuntime(runtime)` 注入模拟的 runtime
2. 调用 `monitorWeixinProvider(opts)` 启动监控

**不需要 HTTP 桥接！** 直接用 Golang 通过 stdin/stdout 与 Node.js 通信即可。

---

## 方案设计

```
┌─────────────────────────────────────────────────────────────┐
│                         Golang 主程序                        │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐      │
│  │ Runtime Mock │  │   AI Logic   │  │ Stdio Bridge │      │
│  │ (内存对象)    │  │  (业务逻辑)   │  │ (通信层)      │      │
│  └──────────────┘  └──────────────┘  └──────────────┘      │
└─────────────────────────────────────────────────────────────┘
                     ↕ stdin/stdout (JSON)
┌─────────────────────────────────────────────────────────────┐
│                   Node.js 独立脚本                           │
│  ┌──────────────┐  ┌──────────────┐                         │
│  │ Stdio Handler│  │ Plugin Core  │                         │
│  │ (协议处理)    │  │ (微信插件)    │                         │
│  └──────────────┘  └──────────────┘                         │
└─────────────────────────────────────────────────────────────┘
                     ↕ HTTP + Token
┌─────────────────────────────────────────────────────────────┐
│                      微信服务器                               │
└─────────────────────────────────────────────────────────────┘
```

---

## 通信协议

使用简单的 JSON 行协议：

```
Golang → Node.js:
{"type":"init","runtime":{...}}
{"type":"start","config":{...}}

Node.js → Golang:
{"type":"ready"}
{"type":"route","request":{...},"id":"1"}
{"type":"ai_dispatch","ctx":{...},"id":"2"}

Golang → Node.js:
{"type":"route_response","id":"1","result":{...}}
{"type":"deliver","text":"回复内容","id":"2"}
{"type":"error","id":"2","message":"..."}
```

---

## Node.js 启动脚本

创建 `standalone.js`（不依赖 OpenClaw SDK）：

```javascript
// standalone.js
import { setWeixinRuntime } from './tencent-weixin-openclaw-weixin-1.0.2/package/src/runtime.js';
import { monitorWeixinProvider } from './tencent-weixin-openclaw-weixin-1.0.2/package/src/monitor/monitor.js';
import { startWeixinLoginWithQr, waitForWeixinLogin } from './tencent-weixin-openclaw-weixin-1.0.2/package/src/auth/login-qr.js';
import { saveWeixinAccount, registerWeixinAccountId, normalizeAccountId } from './tencent-weixin-openclaw-weixin-1.0.2/package/src/auth/accounts.js';

// ==================== Stdio 通信 ====================

let messageId = 0;
const pendingCallbacks = new Map();

function sendToGolang(type, data = {}) {
  const message = JSON.stringify({ type, ...data, id: ++messageId });
  console.log(message);
  return messageId;
}

// 从 stdin 读取 Golang 的消息
process.stdin.setEncoding('utf8');
process.stdin.on('data', (chunk) => {
  const lines = chunk.toString().split('\n').filter(Boolean);
  for (const line of lines) {
    try {
      const msg = JSON.parse(line);
      handleGolangMessage(msg);
    } catch (err) {
      console.error(JSON.stringify({ type: 'error', message: err.message }));
    }
  }
});

function handleGolangMessage(msg) {
  const { type, id, ...data } = msg;

  switch (type) {
    case 'route_response':
      const callback = pendingCallbacks.get(id);
      if (callback) {
        callback(data.result);
        pendingCallbacks.delete(id);
      }
      break;

    case 'login_qr':
      // 显示二维码
      console.error(JSON.stringify({ type: 'qr', url: data.qrcodeUrl }));
      break;

    default:
      console.error(JSON.stringify({ type: 'unknown', originalType: type }));
  }
}

// ==================== 模拟 Runtime ====================

let currentConfig = null;
let currentAccountId = null;

function createMockRuntime() {
  return {
    channel: {
      routing: {
        resolveAgentRoute: async ({ channel, accountId, peer }) => {
          currentAccountId = accountId;

          // 向 Golang 请求路由决策
          const id = sendToGolang('route', {
            request: { channel, accountId, peerId: peer.id, peerKind: peer.kind }
          });

          // 等待 Golang 响应
          return new Promise((resolve) => {
            pendingCallbacks.set(id, resolve);
          });
        },
      },

      session: {
        resolveStorePath: (storeConfig, { agentId }) => {
          return `/tmp/weixin-sessions/${agentId}.json`;
        },

        async recordInboundSession({ storePath, sessionKey, ctx, updateLastRoute, onRecordError }) {
          try {
            // 通知 Golang 记录会话
            sendToGolang('session_record', {
              storePath,
              sessionKey,
              ctx: {
                body: ctx.Body,
                from: ctx.From,
                to: ctx.To,
                timestamp: ctx.Timestamp,
                mediaPath: ctx.MediaPath,
              },
              updateLastRoute,
            });
          } catch (err) {
            onRecordError?.(err);
          }
        },
      },

      reply: {
        finalizeInboundContext: (ctx) => {
          sendToGolang('context_finalize', { ctx });
          return ctx;
        },

        resolveHumanDelayConfig: (cfg, agentId) => ({ mode: 'auto' }),

        createReplyDispatcherWithTyping: ({ humanDelay, typingCallbacks, deliver, onError }) => {
          let isIdle = true;

          return {
            dispatcher: {
              async deliver(payload) {
                if (!isIdle) {
                  console.error('[Warning] Dispatcher not idle');
                }
                isIdle = false;

                try {
                  await typingCallbacks.start?.();

                  // 向 Golang 请求发送消息
                  const id = sendToGolang('deliver_request', {
                    payload: { text: payload.text, mediaUrl: payload.mediaUrl }
                  });

                  // 等待 Golang 处理完成
                  await new Promise((resolve, reject) => {
                    pendingCallbacks.set(id, resolve);
                    setTimeout(() => reject(new Error('Deliver timeout')), 30000);
                  });

                } catch (err) {
                  onError(err, { kind: 'deliver' });
                } finally {
                  await typingCallbacks.stop?.();
                  isIdle = true;
                }
              },
            },
            replyOptions: {},
            markDispatchIdle: () => { isIdle = true; },
          };
        },

        async dispatchReplyFromConfig({ ctx, cfg, dispatcher }) {
          // 通知 Golang 处理 AI
          sendToGolang('ai_dispatch', {
            ctx: {
              body: ctx.Body,
              from: ctx.From,
              to: ctx.To,
              accountId: ctx.AccountId,
              contextToken: ctx.context_token,
              mediaPath: ctx.MediaPath,
            }
          });

          // Golang 会通过 deliver_request 回调发送回复
        },

        async withReplyDispatcher({ dispatcher, run }) {
          await run();
        },
      },

      media: {
        async saveMediaBuffer({ buffer, accountId, msgId, type }) {
          // 保存媒体到本地
          const fs = await import('node:fs/promises');
          const path = await import('node:path');

          const mediaDir = `/tmp/weixin-media/${accountId}`;
          await fs.mkdir(mediaDir, { recursive: true });

          const ext = type === 'image' ? 'png' : type === 'video' ? 'mp4' : 'bin';
          const fileName = `${msgId}.${ext}`;
          const filePath = path.join(mediaDir, fileName);

          await fs.writeFile(filePath, Buffer.from(buffer));
          return filePath;
        },
      },

      commands: {
        async resolveSenderCommandAuthorizationWithRuntime({ senderId, readAllowFromStore }) {
          const allowFrom = await readAllowFromStore();
          const isAllowed = allowFrom.length === 0 || allowFrom.includes(senderId);
          return {
            senderAllowedForCommands: true,
            commandAuthorized: isAllowed,
          };
        },
      },
    },

    log: (msg) => console.error(JSON.stringify({ type: 'log', message: msg })),
    error: (msg) => console.error(JSON.stringify({ type: 'error', message: msg })),
  };
}

// ==================== 主流程 ====================

async function main() {
  console.error('[Standalone] Weixin plugin starting...');

  // 等待 Golang 初始化
  await new Promise(resolve => setTimeout(resolve, 1000));

  const runtime = createMockRuntime();
  setWeixinRuntime(runtime);

  // 通知 Golang 准备好了
  sendToGolang('ready');

  // 等待登录指令
  console.error('[Standalone] Waiting for login command...');
}

main().catch(err => {
  console.error(JSON.stringify({ type: 'fatal', message: err.message }));
  process.exit(1);
});

// 导出启动监控的函数（供 Golang 通过 eval 调用）
globalThis.startMonitor = async function(config) {
  currentConfig = config;

  try {
    await monitorWeixinProvider({
      baseUrl: config.baseUrl,
      cdnBaseUrl: config.cdnBaseUrl,
      token: config.token,
      accountId: config.accountId,
      config: config.openclawConfig,
      abortSignal: new AbortController().signal, // TODO: 支持 Golang 控制停止
      setStatus: (status) => {
        sendToGolang('status', { status });
      },
    });
  } catch (err) {
    sendToGolang('monitor_error', { message: err.message });
  }
};

globalThis.startLogin = async function(apiBaseUrl) {
  try {
    const startResult = await startWeixinLoginWithQr({
      accountId: 'default',
      apiBaseUrl: apiBaseUrl || 'https://ilinkai.weixin.qq.com',
      botType: '3',
    });

    sendToGolang('login_qr', { qrcodeUrl: startResult.qrcodeUrl });

    const waitResult = await waitForWeixinLogin({
      sessionKey: startResult.sessionKey,
      apiBaseUrl: apiBaseUrl || 'https://ilinkai.weixin.qq.com',
    });

    const normalizedId = normalizeAccountId(waitResult.accountId!);
    saveWeixinAccount(normalizedId, {
      token: waitResult.botToken,
      baseUrl: waitResult.baseUrl,
      userId: waitResult.userId,
    });
    registerWeixinAccountId(normalizedId);

    sendToGolang('login_success', {
      accountId: normalizedId,
      token: waitResult.botToken,
      baseUrl: waitResult.baseUrl,
      userId: waitResult.userId,
    });

    return waitResult;
  } catch (err) {
    sendToGolang('login_error', { message: err.message });
    throw err;
  }
};
```

---

## Golang 主程序

```go
// main.go
package main

import (
	"bufio"
	"encoding/json"
	"fmt"
	"log"
	"os"
	"os/exec"
	"strings"
	"sync"
)

type Message struct {
	Type    string                 `json:"type"`
	Id      int                    `json:"id,omitempty"`
	Request map[string]interface{} `json:"request,omitempty"`
	Result  map[string]interface{} `json:"result,omitempty"`
	Ctx     map[string]interface{} `json:"ctx,omitempty"`
	Payload map[string]interface{} `json:"payload,omitempty"`
	Status  map[string]interface{} `json:"status,omitempty"`
}

type PluginProcess struct {
	cmd     *exec.Cmd
 stdin   io.WriteCloser
	scanner *bufio.Scanner
	// 待处理回调
	pending map[int]chan map[string]interface{}
	mu      sync.RWMutex
}

func NewPluginProcess() *PluginProcess {
	return &PluginProcess{
		pending: make(map[int]chan map[string]interface{}),
	}
}

func (p *PluginProcess) Start() error {
	log.Println("Starting Node.js plugin...")

	p.cmd = exec.Command("node", "--input-type=module", "-e", `
		// 内联 standalone.js 的内容
		// 或者从文件加载
	`)

	// 或者直接运行文件
	// p.cmd = exec.Command("node", "standalone.js")

	stdin, err := p.cmd.StdinPipe()
	if err != nil {
		return err
	}
	p.stdin = stdin

	stdout, err := p.cmd.StdoutPipe()
	if err != nil {
		return err
	}

	p.scanner = bufio.NewScanner(stdout)

	if err := p.cmd.Start(); err != nil {
		return err
	}

	// 启动消息读取循环
	go p.readLoop()

	return nil
}

func (p *PluginProcess) readLoop() {
	for p.scanner.Scan() {
		line := p.scanner.Text()
		if strings.HasPrefix(line, "{") { // JSON 行
			var msg Message
			if err := json.Unmarshal([]byte(line), &msg); err != nil {
				log.Printf("[Error] Failed to parse: %s", err)
				continue
			}
			p.handleMessage(&msg)
		}
	}
}

func (p *PluginProcess) handleMessage(msg *Message) {
	switch msg.Type {
	case "ready":
		log.Println("[Plugin] Ready")

	case "qr":
		log.Printf("[QR] %s\n", msg.Payload["url"])

	case "route":
		// 处理路由请求
		route := p.resolveRoute(msg.Request)
		p.send("route_response", map[string]interface{}{
			"id":     msg.Id,
			"result": route,
		})

	case "ai_dispatch":
		// 处理 AI 调用
		go p.handleAiDispatch(msg)

	case "deliver_request":
		// 发送消息到微信
		p.handleDeliver(msg)

	case "session_record":
		// 记录会话
		p.handleSessionRecord(msg)

	case "context_finalize":
		// 上下文完善

	case "status":
		log.Printf("[Status] %+v", msg.Status)

	case "error":
		log.Printf("[Error] %s", msg.Payload["message"])

	case "log":
		log.Printf("[Plugin] %s", msg.Payload["message"])

	default:
		// 处理响应
		if msg.Id > 0 {
			p.mu.RLock()
			ch, ok := p.pending[msg.Id]
			p.mu.RUnlock()

			if ok {
				data := map[string]interface{}{
					"type":   msg.Type,
					"result": msg.Result,
					"ctx":    msg.Ctx,
					"payload": msg.Payload,
				}
				ch <- data
			}
		}
	}
}

func (p *PluginProcess) send(msgType string, data map[string]interface{}) error {
	msg := map[string]interface{}{"type": msgType}
	for k, v := range data {
		msg[k] = v
	}

	jsonBytes, err := json.Marshal(msg)
	if err != nil {
		return err
	}

	_, err = p.stdin.Write(append(jsonBytes, '\n'))
	return err
}

func (p *PluginProcess) SendAndWait(msgType string, data map[string]interface{}) (map[string]interface{}, error) {
	id := len(p.pending) + 1
	data["id"] = id

	ch := make(chan map[string]interface{}, 1)

	p.mu.Lock()
	p.pending[id] = ch
	p.mu.Unlock()

	defer func() {
		p.mu.Lock()
		delete(p.pending, id)
		p.mu.Unlock()
		close(ch)
	}()

	if err := p.send(msgType, data); err != nil {
		return nil, err
	}

	// 等待响应
	result := <-ch
	if errMsg, ok := result["error"].(string); ok {
		return nil, fmt.Errorf(errMsg)
	}

	return result, nil
}

// ==================== 回调处理 ====================

func (p *PluginProcess) resolveRoute(request map[string]interface{}) map[string]interface{} {
	channel := request["channel"].(string)
	accountId := request["accountId"].(string)
	peerId := request["peerId"].(string)

	log.Printf("[Route] channel=%s account=%s peer=%s", channel, accountId, peerId)

	// 决定路由
	return map[string]interface{}{
		"agentId":       "default-agent",
		"sessionKey":    fmt.Sprintf("main:%s:%s:%s", channel, accountId, peerId),
		"mainSessionKey": fmt.Sprintf("main:%s:%s", channel, accountId),
	}
}

func (p *PluginProcess) handleAiDispatch(msg *Message) {
	ctx := msg.Ctx
	body := ctx["body"].(string)
	from := ctx["from"].(string)

	log.Printf("[AI] Processing from %s: %s", from, body)

	// 调用你的 AI
	response := p.callAI(body, from)

	// 通知插件发送回复
	p.send("deliver_request", map[string]interface{}{
		"id": msg.Id,
		"payload": map[string]interface{}{
			"text": response,
		},
	})
}

func (p *PluginProcess) handleDeliver(msg *Message) {
	payload := msg.Payload
	text := payload["text"].(string)

	log.Printf("[Deliver] Sending: %s", text)

	// 插件会自动处理实际的发送
	// 我们只需要确认
	p.send("deliver_response", map[string]interface{}{
		"id": msg.Id,
	})
}

func (p *PluginProcess) handleSessionRecord(msg *Message) {
	sessionKey := msg.Ctx["sessionKey"]
	ctx := msg.Ctx

	log.Printf("[Session] Record session=%s from=%s", sessionKey, ctx["from"])

	// TODO: 保存到数据库
}

func (p *PluginProcess) callAI(body string, from string) string {
	// TODO: 实现你的 AI 调用
	return fmt.Sprintf("你说: %s", body)
}

// ==================== 主程序 ====================

func main() {
	plugin := NewPluginProcess()
	if err := plugin.Start(); err != nil {
		log.Fatalf("Failed to start plugin: %v", err)
	}
	defer plugin.cmd.Process.Kill()

	// 等待插件准备就绪
	// (readLoop 会收到 "ready" 消息)

	// 触发登录
	log.Println("Initiating login...")
	plugin.send("login_command", map[string]interface{}{
		"apiBaseUrl": "https://ilinkai.weixin.qq.com",
	})

	// 等待用户操作...

	// 保持运行
	select {}
}
```

---

## 更简化的方案：直接内联

如果想让部署更简单，可以把 Node.js 代码直接内联到 Go 程序中：

```go
const standaloneJS = `
import { setWeixinRuntime } from './src/runtime.js';
import { monitorWeixinProvider } from './src/monitor/monitor.js';
// ... 其他代码
`

func (p *PluginProcess) Start() error {
	p.cmd = exec.Command("node", "--input-type=module", "-e", standaloneJS)
	// ...
}
```

---

## 项目结构

```
weixin-plugin/
├── main.go              # Golang 主程序
├── standalone.js        # Node.js 启动脚本（可内联到 Go）
└── tencent-weixin-openclaw-weixin/  # 插件源码
    └── package/
```

---

## 优势

1. **无需 HTTP**：使用 stdin/stdout 通信
2. **单一进程**：Golang 直接管理 Node.js 子进程
3. **类型安全**：通过 JSON 通信，无需复杂的类型转换
4. **调试简单**：可以直接看到双方的通信日志
