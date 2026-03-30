# Golang 模拟 OpenClaw 完整方案

## 架构设计

```
┌─────────────────────────────────────────────────────────────────┐
│                        Golang 主程序                            │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐          │
│  │  HTTP Bridge │  │   Runtime    │  │   AI Logic   │          │
│  │  (通信桥接)   │  │  (模拟接口)   │  │  (业务逻辑)   │          │
│  └──────────────┘  └──────────────┘  └──────────────┘          │
└─────────────────────────────────────────────────────────────────┘
         │                                 ▲
         │ HTTP/JSON                       │ HTTP/JSON
         ▼                                 │
┌─────────────────────────────────────────────────────────────────┐
│              Node.js 插件进程 (独立运行)                         │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐          │
│  │ HTTP Server  │  │ Runtime      │  │ Plugin Core  │          │
│  │ (桥接层)      │  │ (适配器)     │  │ (微信插件)    │          │
│  └──────────────┘  └──────────────┘  └──────────────┘          │
└─────────────────────────────────────────────────────────────────┘
         │
         │ HTTP + Token
         ▼
┌─────────────────────────────────────────────────────────────────┐
│                      微信服务器                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 方案选择

有两种集成方式：

### 方案 A：HTTP 桥接（推荐）

Golang 和 Node.js 通过 HTTP 通信，各自运行在独立进程。

**优点**：
- 语言隔离，稳定性好
- 独立部署，易于维护
- 可以单独重启插件

**缺点**：
- 需要额外的 HTTP 通信

### 方案 B：直接嵌入 Node.js

Golang 启动 Node.js 子进程。

**优点**：
- 部署简单

**缺点**：
- 进程管理复杂
- 调试困难

---

## 方案 A：HTTP 桥接完整实现

### 第一步：Node.js 桥接层

创建 `bridge-server.js`：

```javascript
// bridge-server.js
import express from 'express';
import cors from 'cors';
import { setWeixinRuntime } from './src/runtime.js';
import { monitorWeixinProvider } from './src/monitor/monitor.js';
import { startWeixinLoginWithQr, waitForWeixinLogin } from './src/auth/login-qr.js';
import { saveWeixinAccount, registerWeixinAccountId, normalizeAccountId } from './src/auth/accounts.js';

const app = express();
app.use(cors());
app.use(express.json());

const PORT = 3456;
const SECRET = 'your-secret-key'; // 用于认证

// 存储运行时状态
const runtimes = new Map();
let currentMonitor = null;

// ==================== 认证中间件 ====================

function checkAuth(req, res, next) {
  const token = req.headers['x-bridge-token'];
  if (token !== SECRET) {
    return res.status(401).json({ error: 'Unauthorized' });
  }
  next();
}

// ==================== Mock Runtime ====================

function createMockRuntime(config, callbacks) {
  return {
    channel: {
      routing: {
        resolveAgentRoute: ({ channel, accountId, peer }) => {
          // 通知 Golang 获取路由
          const route = callbacks.onRouteResolve?.({
            channel,
            accountId,
            peerId: peer.id,
            peerKind: peer.kind,
          }) || {
            agentId: 'default',
            sessionKey: 'main:main',
            mainSessionKey: 'main:main',
          };
          return route;
        },
      },

      session: {
        resolveStorePath: (storeConfig, { agentId }) => {
          return callbacks.onStorePathResolve?.(agentId) || `/tmp/${agentId}.json`;
        },

        async recordInboundSession({ storePath, sessionKey, ctx, updateLastRoute, onRecordError }) {
          try {
            await callbacks.onSessionRecord?.({
              storePath,
              sessionKey,
              ctx,
              updateLastRoute,
            });
          } catch (err) {
            onRecordError?.(err);
          }
        },
      },

      reply: {
        finalizeInboundContext: (ctx) => {
          callbacks.onContextFinalize?.(ctx);
          return ctx;
        },

        resolveHumanDelayConfig: (cfg, agentId) => {
          return callbacks.onHumanDelayResolve?.(agentId) || {
            mode: 'auto',
          };
        },

        createReplyDispatcherWithTyping: ({ humanDelay, typingCallbacks, deliver, onError }) => {
          let isIdle = true;

          return {
            dispatcher: {
              async deliver(payload) {
                if (!isIdle) {
                  console.warn('[dispatcher] Busy, message may be out of order');
                }

                isIdle = false;

                try {
                  // 开始输入指示
                  await typingCallbacks.start?.();

                  // 通知 Golang 发送消息
                  await callbacks.onDeliver?.(payload, {
                    onStart: () => typingCallbacks.start?.(),
                    onStop: () => typingCallbacks.stop?.(),
                  });

                } catch (err) {
                  onError(err, { kind: 'deliver' });
                } finally {
                  // 停止输入指示
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
          // 通知 Golang 处理 AI 逻辑
          await callbacks.onAiDispatch?.({
            ctx,
            config: cfg,
            dispatcher: {
              async deliver(payload) {
                await dispatcher.deliver(payload);
              },
            },
          });
        },

        async withReplyDispatcher({ dispatcher, run }) {
          await run();
        },
      },

      media: {
        async saveMediaBuffer({ buffer, accountId, msgId, type }) {
          return callbacks.onMediaSave?.({ buffer, accountId, msgId, type })
            || `/tmp/media/${msgId}`;
        },
      },

      commands: {
        async resolveSenderCommandAuthorizationWithRuntime({ senderId, readAllowFromStore }) {
          const allowFrom = await readAllowFromStore();
          const isAllowed = allowFrom.length === 0 || allowFrom.includes(senderId);

          return {
            senderAllowedForCommands: isAllowed,
            commandAuthorized: isAllowed,
          };
        },
      },
    },

    log: (msg) => console.log('[Runtime]', msg),
    error: (msg) => console.error('[Runtime]', msg),
  };
}

// ==================== API 路由 ====================

// 1. 启动监控
app.post('/api/monitor/start', checkAuth, async (req, res) => {
  try {
    const { accountId, baseUrl, token, cdnBaseUrl, config } = req.body;

    if (currentMonitor) {
      return res.status(400).json({ error: 'Monitor already running' });
    }

    // 创建回调函数
    const callbacks = {
      onRouteResolve: async (params) => {
        // 调用 Golang 获取路由
        const response = await fetch('http://localhost:8080/callback/route', {
          method: 'POST',
          headers: {
            'Content-Type': 'application/json',
            'X-Bridge-Token': SECRET,
          },
          body: JSON.stringify(params),
        });
        return response.json();
      },

      onContextFinalize: async (ctx) => {
        // 通知 Golang 上下文已完善
        await fetch('http://localhost:8080/callback/context/finalize', {
          method: 'POST',
          headers: {
            'Content-Type': 'application/json',
            'X-Bridge-Token': SECRET,
          },
          body: JSON.stringify(ctx),
        });
      },

      onSessionRecord: async (params) => {
        // 通知 Golang 记录会话
        await fetch('http://localhost:8080/callback/session/record', {
          method: 'POST',
          headers: {
            'Content-Type': 'application/json',
            'X-Bridge-Token': SECRET,
          },
          body: JSON.stringify(params),
        });
      },

      onAiDispatch: async (params) => {
        // 通知 Golang 执行 AI 调用
        await fetch('http://localhost:8080/callback/ai/dispatch', {
          method: 'POST',
          headers: {
            'Content-Type': 'application/json',
            'X-Bridge-Token': SECRET,
          },
          body: JSON.stringify({
            ctx: params.ctx,
            // dispatcher 是一个本地函数，不需要传递
          }),
        });
      },

      onDeliver: async (payload, typingCallbacks) => {
        // 通知 Golang 发送消息
        const response = await fetch('http://localhost:8080/callback/deliver', {
          method: 'POST',
          headers: {
            'Content-Type': 'application/json',
            'X-Bridge-Token': SECRET,
          },
          body: JSON.stringify(payload),
        });

        if (!response.ok) {
          throw new Error(`Deliver failed: ${response.statusText}`);
        }

        // Golang 可能要求我们先执行 typing
        const result = await response.json();
        if (result.typingStart) {
          await typingCallbacks.onStart?.();
        }
      },

      onStorePathResolve: async (agentId) => {
        const response = await fetch(`http://localhost:8080/callback/store/path?agentId=${agentId}`, {
          headers: { 'X-Bridge-Token': SECRET },
        });
        const data = await response.json();
        return data.path;
      },

      onMediaSave: async (params) => {
        // 保存媒体到本地，返回路径
        const fs = await import('node:fs/promises');
        const path = await import('node:path');
        const crypto = await import('node:crypto');

        const mediaDir = `/tmp/weixin-media/${params.accountId}`;
        await fs.mkdir(mediaDir, { recursive: true });

        const ext = params.type === 'image' ? 'png' : 'bin';
        const fileName = `${params.msgId}.${ext}`;
        const filePath = path.join(mediaDir, fileName);

        await fs.writeFile(filePath, Buffer.from(params.buffer, 'base64'));
        return filePath;
      },
    };

    // 创建 mock runtime
    const mockRuntime = createMockRuntime(config, callbacks);
    setWeixinRuntime(mockRuntime);

    // 启动监控
    const abortController = new AbortController();

    currentMonitor = monitorWeixinProvider({
      baseUrl,
      cdnBaseUrl,
      token,
      accountId,
      config,
      abortSignal: abortController.signal,
      setStatus: (status) => {
        // 通知 Golang 状态更新
        fetch('http://localhost:8080/callback/status', {
          method: 'POST',
          headers: {
            'Content-Type': 'application/json',
            'X-Bridge-Token': SECRET,
          },
          body: JSON.stringify(status),
        }).catch(() => {});
      },
    });

    runtimes.set(accountId, {
      mockRuntime,
      abortController,
    });

    res.json({ success: true, message: 'Monitor started' });

  } catch (error) {
    console.error('Failed to start monitor:', error);
    res.status(500).json({ error: error.message });
  }
});

// 2. 停止监控
app.post('/api/monitor/stop', checkAuth, async (req, res) => {
  const { accountId } = req.body;

  const runtime = runtimes.get(accountId);
  if (runtime?.abortController) {
    runtime.abortController.abort();
    runtimes.delete(accountId);
    currentMonitor = null;
  }

  res.json({ success: true });
});

// 3. 发送消息（从 Golang 调用）
app.post('/api/send', checkAuth, async (req, res) => {
  try {
    const { accountId, to, text, contextToken } = req.body;

    const runtime = runtimes.get(accountId);
    if (!runtime) {
      return res.status(404).json({ error: 'Account not found' });
    }

    // 动态导入 sendMessage
    const { sendMessageWeixin } = await import('./src/messaging/send.js');
    const { loadWeixinAccount } = await import('./src/auth/accounts.js');

    const account = loadWeixinAccount(accountId);
    await sendMessageWeixin({
      to,
      text,
      opts: {
        baseUrl: account?.baseUrl || 'https://ilinkai.weixin.qq.com',
        token: account?.token,
        contextToken,
      },
    });

    res.json({ success: true });
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// 4. 健康检查
app.get('/health', (req, res) => {
  res.json({ status: 'ok', monitorRunning: currentMonitor !== null });
});

// ==================== 启动服务器 ====================

app.listen(PORT, () => {
  console.log(`Weixin Bridge Server listening on port ${PORT}`);
  console.log(`Secret: ${SECRET}`);
});
```

---

### 第二步：Golang 主程序

```go
// main.go
package main

import (
	"bytes"
	"context"
	"encoding/json"
	"fmt"
	"io"
	"log"
	"net/http"
	"os"
	"os/exec"
	"time"
)

const (
	BridgeURL      = "http://localhost:3456"
	BridgeSecret   = "your-secret-key"
	BridgeNodePath = "./bridge-server.js"
)

// ==================== 数据结构 ====================

type MsgContext struct {
	Body         string `json:"Body"`
	From         string `json:"From"`
	To           string `json:"To"`
	AccountId    string `json:"AccountId"`
	MessageSid   string `json:"MessageSid"`
	Timestamp    int64  `json:"Timestamp,omitempty"`
	SessionKey   string `json:"SessionKey,omitempty"`
	ContextToken string `json:"context_token,omitempty"`
	MediaPath    string `json:"MediaPath,omitempty"`
	MediaType    string `json:"MediaType,omitempty"`
}

type AgentRoute struct {
	AgentId       string `json:"agentId"`
	SessionKey    string `json:"sessionKey"`
	MainSessionKey string `json:"mainSessionKey"`
}

type DeliveryPayload struct {
	Text      string   `json:"text"`
	MediaUrl  string   `json:"mediaUrl,omitempty"`
	MediaUrls []string `json:"mediaUrls,omitempty"`
}

type BridgeServer struct {
	cmd    *exec.Cmd
	client *http.Client
}

// ==================== 桥接服务器管理 ====================

func NewBridgeServer() *BridgeServer {
	return &BridgeServer{
		client: &http.Client{Timeout: 30 * time.Second},
	}
}

func (b *BridgeServer) Start() error {
	log.Println("Starting Node.js bridge server...")

	b.cmd = exec.Command("node", BridgeNodePath)
	b.cmd.Stdout = os.Stdout
	b.cmd.Stderr = os.Stderr

	if err := b.cmd.Start(); err != nil {
		return fmt.Errorf("failed to start bridge: %w", err)
	}

	// 等待服务器启动
	for i := 0; i < 30; i++ {
		ctx, cancel := context.WithTimeout(context.Background(), 2*time.Second)
		req, _ := http.NewRequestWithContext(ctx, "GET", BridgeURL+"/health", nil)
		req.Header.Set("X-Bridge-Token", BridgeSecret)

		resp, err := b.client.Do(req)
		cancel()

		if err == nil && resp.StatusCode == 200 {
			log.Println("Bridge server is ready")
			return nil
		}
		time.Sleep(500 * time.Millisecond)
	}

	return fmt.Errorf("bridge server failed to start")
}

func (b *BridgeServer) Stop() {
	if b.cmd != nil && b.cmd.Process != nil {
		log.Println("Stopping bridge server...")
		b.cmd.Process.Kill()
		b.cmd.Wait()
	}
}

func (b *BridgeServer) request(method, path string, body interface{}) ([]byte, error) {
	var bodyReader io.Reader

	if body != nil {
		jsonData, err := json.Marshal(body)
		if err != nil {
			return nil, err
		}
		bodyReader = bytes.NewReader(jsonData)
	}

	ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
	defer cancel()

	req, err := http.NewRequestWithContext(ctx, method, BridgeURL+path, bodyReader)
	if err != nil {
		return nil, err
	}

	req.Header.Set("Content-Type", "application/json")
	req.Header.Set("X-Bridge-Token", BridgeSecret)

	resp, err := b.client.Do(req)
	if err != nil {
		return nil, err
	}
	defer resp.Body.Close()

	return io.ReadAll(resp.Body)
}

// ==================== 回调处理器 ====================

type CallbackHandler struct {
	bridge *BridgeServer
}

func (h *CallbackHandler) HandleRouteResolve(w http.ResponseWriter, r *http.Request) {
	var params struct {
		Channel   string `json:"channel"`
		AccountId string `json:"accountId"`
		PeerId    string `json:"peerId"`
		PeerKind  string `json:"peerKind"`
	}

	if err := json.NewDecoder(r.Body).Decode(&params); err != nil {
		http.Error(w, err.Error(), 400)
		return
	}

	log.Printf("[Route] Channel=%s Account=%s Peer=%s", params.Channel, params.AccountId, params.PeerId)

	// 决定使用哪个 agent 和 session
	route := AgentRoute{
		AgentId:       "default-agent",
		SessionKey:    fmt.Sprintf("main:%s:%s:%s", params.Channel, params.AccountId, params.PeerId),
		MainSessionKey: fmt.Sprintf("main:%s:%s", params.Channel, params.AccountId),
	}

	json.NewEncoder(w).Encode(route)
}

func (h *CallbackHandler) HandleContextFinalize(w http.ResponseWriter, r *http.Request) {
	var ctx MsgContext
	if err := json.NewDecoder(r.Body).Decode(&ctx); err != nil {
		http.Error(w, err.Error(), 400)
		return
	}

	log.Printf("[ContextFinalize] From=%s Body=%s", ctx.From, ctx.Body)
	w.WriteHeader(200)
}

func (h *CallbackHandler) HandleSessionRecord(w http.ResponseWriter, r *http.Request) {
	var params struct {
		SessionKey string     `json:"sessionKey"`
		Ctx        MsgContext `json:"ctx"`
	}

	if err := json.NewDecoder(r.Body).Decode(&params); err != nil {
		http.Error(w, err.Error(), 400)
		return
	}

	log.Printf("[SessionRecord] SessionKey=%s From=%s", params.SessionKey, params.Ctx.From)

	// TODO: 保存到数据库或文件
	// sessionStore.Save(params.SessionKey, params.Ctx)

	w.WriteHeader(200)
}

func (h *CallbackHandler) HandleAiDispatch(w http.ResponseWriter, r *http.Request) {
	var params struct {
		Ctx MsgContext `json:"ctx"`
	}

	if err := json.NewDecoder(r.Body).Decode(&params); err != nil {
		http.Error(w, err.Error(), 400)
		return
	}

	log.Printf("[AiDispatch] From=%s Body=%s", params.Ctx.From, params.Ctx.Body)

	// 调用你的 AI 逻辑
	response := h.callAI(params.Ctx)

	// 通知 Node.js 发送回复
	body, _ := json.Marshal(DeliveryPayload{
		Text: response,
	})

	h.bridge.request("POST", "/api/send", map[string]interface{}{
		"accountId":    params.Ctx.AccountId,
		"to":          params.Ctx.To,
		"text":        response,
		"contextToken": params.Ctx.ContextToken,
	})

	w.WriteHeader(200)
}

func (h *CallbackHandler) callAI(ctx MsgContext) string {
	// TODO: 实现你的 AI 调用逻辑
	// - 调用 OpenAI API
	// - 调用本地模型
	// - 任何自定义逻辑

	log.Printf("[AI] Processing message from %s: %s", ctx.From, ctx.Body)

	// 简单示例：echo
	return fmt.Sprintf("你说: %s", ctx.Body)
}

func (h *CallbackHandler) HandleDeliver(w http.ResponseWriter, r *http.Request) {
	var payload DeliveryPayload
	if err := json.NewDecoder(r.Body).Decode(&payload); err != nil {
		http.Error(w, err.Error(), 400)
		return
	}

	log.Printf("[Deliver] Text=%s MediaUrl=%s", payload.Text, payload.MediaUrl)

	// 这里消息已经通过 Node.js 发送到微信了
	// 如果需要额外的处理，可以在这里做

	json.NewEncoder(w).Encode(map[string]bool{
		"typingStart": false, // 是否需要 typing 指示
	})
}

func (h *CallbackHandler) HandleStorePath(w http.ResponseWriter, r *http.Request) {
	agentId := r.URL.Query().Get("agentId")
	path := fmt.Sprintf("/tmp/sessions/%s.json", agentId)

	json.NewEncoder(w).Encode(map[string]string{
		"path": path,
	})
}

func (h *CallbackHandler) HandleStatus(w http.ResponseWriter, r *http.Request) {
	var status map[string]interface{}
	json.NewDecoder(r.Body).Decode(&status)

	log.Printf("[Status] %+v", status)
	w.WriteHeader(200)
}

// ==================== 主程序 ====================

func main() {
	// 启动桥接服务器
	bridge := NewBridgeServer()
	if err := bridge.Start(); err != nil {
		log.Fatalf("Failed to start bridge: %v", err)
	}
	defer bridge.Stop()

	// 设置回调处理器
	handler := &CallbackHandler{bridge: bridge}

	http.HandleFunc("/callback/route", handler.HandleRouteResolve)
	http.HandleFunc("/callback/context/finalize", handler.HandleContextFinalize)
	http.HandleFunc("/callback/session/record", handler.HandleSessionRecord)
	http.HandleFunc("/callback/ai/dispatch", handler.HandleAiDispatch)
	http.HandleFunc("/callback/deliver", handler.HandleDeliver)
	http.HandleFunc("/callback/store/path", handler.HandleStorePath)
	http.HandleFunc("/callback/status", handler.HandleStatus)

	// 启动 HTTP 服务器
	go func() {
		log.Println("Golang callback server listening on :8080")
		log.Fatal(http.ListenAndServe(":8080", nil))
	}()

	// 等待信号
	sigChan := make(chan os.Signal, 1)
	signal.Notify(sigChan, os.Interrupt)
	<-sigChan

	log.Println("Shutting down...")
}
```

---

### 第三步：登录和启动

```go
// login.go
package main

import (
	"bytes"
	"encoding/json"
	"fmt"
	"io"
	"log"
	"net/http"
)

type LoginResult struct {
	Token     string `json:"token"`
	AccountId string `json:"accountId"`
	BaseUrl   string `json:"baseUrl"`
	UserId    string `json:"userId"`
}

func LoginWithQr() (*LoginResult, error) {
	// 1. 获取二维码
	resp, err := http.Get("https://ilinkai.weixin.qq.com/ilink/bot/get_bot_qrcode?bot_type=3")
	if err != nil {
		return nil, err
	}
	defer resp.Body.Close()

	body, _ := io.ReadAll(resp.Body)
	var qrResp struct {
		Qrcode          string `json:"qrcode"`
		QrcodeImgContent string `json:"qrcode_img_content"`
	}
	json.Unmarshal(body, &qrResp)

	fmt.Println("请扫描二维码:")
	fmt.Println(qrResp.QrcodeImgContent)

	// 2. 等待扫码
	deadline := time.Now().Add(8 * time.Minute)
	for time.Now().Before(deadline) {
		resp, err := http.Get(
			fmt.Sprintf("https://ilinkai.weixin.qq.com/ilink/bot/get_qrcode_status?qrcode=%s", qrResp.Qrcode),
		)
		if err != nil {
			continue
		}

		body, _ := io.ReadAll(resp.Body)
		resp.Body.Close()

		var statusResp struct {
			Status      string `json:"status"`
			BotToken    string `json:"bot_token"`
			IlinkBotId  string `json:"ilink_bot_id"`
			BaseUrl     string `json:"baseurl"`
			IlinkUserId string `json:"ilink_user_id"`
		}
		json.Unmarshal(body, &statusResp)

		switch statusResp.Status {
		case "wait":
			fmt.Print(".")
		case "scaned":
			fmt.Println("\n已扫码，等待确认...")
		case "confirmed":
			return &LoginResult{
				Token:     statusResp.BotToken,
				AccountId: statusResp.IlinkBotId,
				BaseUrl:   statusResp.BaseUrl,
				UserId:    statusResp.IlinkUserId,
			}, nil
		case "expired":
			return nil, fmt.Errorf("二维码已过期")
		}

		time.Sleep(2 * time.Second)
	}

	return nil, fmt.Errorf("登录超时")
}

func StartMonitor(bridge *BridgeServer, loginResult *LoginResult) error {
	payload := map[string]interface{}{
		"accountId":  loginResult.AccountId,
		"baseUrl":    loginResult.BaseUrl,
		"token":      loginResult.Token,
		"cdnBaseUrl":  "https://novac2c.cdn.weixin.qq.com/c2c",
		"config": map[string]interface{}{
			"channels": map[string]interface{}{
				"openclaw-weixin": map[string]interface{}{
					"enabled": true,
				},
			},
			"session": map[string]interface{}{
				"store": "~/.openclaw/sessions",
			},
		},
	}

	body, err := json.Marshal(payload)
	if err != nil {
		return err
	}

	resp, err := bridge.client.Post(
		BridgeURL+"/api/monitor/start",
		"application/json",
		bytes.NewReader(body),
	)
	if err != nil {
		return err
	}
	defer resp.Body.Close()

	if resp.StatusCode != 200 {
		respBody, _ := io.ReadAll(resp.Body)
		return fmt.Errorf("failed to start monitor: %s", string(respBody))
	}

	log.Println("Monitor started successfully")
	return nil
}
```

---

## 项目结构

```
weixin-golang-bridge/
├── main.go                 # Golang 主程序
├── login.go                # 登录逻辑
├── go.mod
├── package.json
├── bridge-server.js        # Node.js 桥接服务器
└── tencent-weixin-openclaw-weixin/  # 插件源码
    └── package/
        └── src/
```

---

## 启动流程

1. **启动 Golang 程序**：
   ```bash
   go run main.go login.go
   ```

2. **扫码登录**：
   - 程序会显示二维码
   - 用微信扫码确认

3. **自动启动监控**：
   - Golang 启动 Node.js 桥接服务器
   - 桥接服务器启动微信插件监控
   - 开始接收消息

---

## 通信流程

```
微信消息 → Node.js 插件 → Golang Callback → AI 处理 → Golang 通知 → Node.js 发送 → 微信
```

---

## 调试技巧

1. **查看桥接服务器日志**：
   ```
   tail -f bridge-server.log
   ```

2. **查看 Golang 日志**：
   ```
   # Golang 直接输出到 stdout
   ```

3. **测试 API**：
   ```bash
   curl http://localhost:3456/health
   ```
