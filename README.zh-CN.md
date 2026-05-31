# qclaw-wechat-client

[English](./README.md)

一个用于把个人 AI Agent 接入微信式消息通道的 TypeScript 客户端与协议实验项目，覆盖扫码登录、channel token、实时 Agent Gateway Protocol，以及消息流式回复。

这个仓库主要用于研究和验证：

- 基于用户扫码确认的授权登录流程
- 聊天用户与本地 / 云端 AI agent 之间的消息传递
- WebSocket 形式的 agent 网关协议
- Claude、Codex、Gemini、OpenClaw、自定义 OpenAI-compatible agent 等多 agent 路由
- 个人自动化和 agent 交互体验的本地开发工作流

## 项目目标

AI agent 只有进入用户已经在用的消息入口，才更容易成为真正可用的生产力工具。这个项目关注的是中间基础设施层：登录、会话状态、channel token、消息流、取消、重连、最终回复交付等。

当前实现面向 QClaw / OpenClaw 兼容的消息流程，定位是个人研究、原型验证和开发者评估。

## 当前能力

- 微信扫码登录流程辅助方法
- token 与 session 管理辅助方法
- 账号、设备、邀请码、channel token 等 API 客户端
- AGP WebSocket 客户端，用于实时 agent 消息
- agent 回复的流式发送辅助方法
- 自动重连和心跳检测
- prompt、cancel、content block、response 等 TypeScript 类型
- 一个简单 echo agent demo，用于验证完整链路

## 试用 / 评估申请说明

我会把这个仓库作为申请 AI agent 平台开发者预览或试用名额时的公开技术参考。它能说明我正在做的方向是：

- agent 消息基础设施
- 个人生产力 agent
- 本地到云端的 agent bridge
- 授权聊天入口
- TypeScript SDK 和协议封装

申请试用的目的不是跑玩具 prompt，而是在真实集成链路里评估新平台能力。

## 安装

```bash
npm install qclaw-wechat-client
# 或
pnpm add qclaw-wechat-client
```

## 开发

```bash
pnpm install
pnpm build
pnpm typecheck
pnpm demo
```

## 快速开始

```typescript
import { QClawClient } from "qclaw-wechat-client";
import type { WxLoginStateData, WxLoginData } from "qclaw-wechat-client";

const client = new QClawClient({ env: "production" });

const guid = "local-device-id";

// 1. 创建登录 state。
const stateRes = await client.getWxLoginState({ guid });
const state = QClawClient.unwrap<WxLoginStateData>(stateRes)?.state;

// 2. 向用户展示扫码登录 URL。
const qrUrl = client.buildWxLoginUrl(state!);
console.log("扫码授权:", qrUrl);

// 3. 用 OAuth 回调 code 换取会话。
const loginRes = await client.wxLogin({ guid, code: authCode, state: state! });

// 4. 构建 agent channel 配置补丁。
const channelToken = QClawClient.unwrap<WxLoginData>(loginRes)?.openclaw_channel_token;
const config = await client.buildPostLoginConfig(channelToken!);
```

## AGP WebSocket 客户端

包内包含 AGP 客户端，用于实时 agent 消息。服务端在用户发消息时下发 `session.prompt`，客户端通过 `session.update` 流式返回片段，再用 `session.promptResponse` 结束本轮回复。

```typescript
import { AGPClient } from "qclaw-wechat-client";

const client = new AGPClient(
  {
    url: "wss://mmgrcalltoken.3g.qq.com/agentwss",
    token: channelToken,
  },
  {
    onConnected() {
      console.log("已连接，等待消息...");
    },
    onPrompt(msg) {
      const { session_id, prompt_id, content } = msg.payload;
      const text = content.map((block) => block.text).join("");
      console.log("用户:", text);

      client.sendMessageChunk(session_id, prompt_id, "Hello ");
      client.sendMessageChunk(session_id, prompt_id, "from an agent bridge.");
      client.sendTextResponse(session_id, prompt_id, "Hello from an agent bridge.");
    },
    onCancel(msg) {
      const { session_id, prompt_id } = msg.payload;
      client.sendCancelledResponse(session_id, prompt_id);
    },
    onError(err) {
      console.error(err);
    },
  },
);

client.start();
```

## 主要 API

### 认证

| 方法 | 用途 |
|---|---|
| `getWxLoginState({ guid })` | 创建扫码登录所需的 CSRF/login state |
| `buildWxLoginUrl(state)` | 构建扫码登录 URL |
| `wxLogin({ guid, code, state })` | 用 OAuth 回调 code 换取会话 |
| `getUserInfo({ guid })` | 获取当前用户信息 |
| `wxLogout({ guid })` | 注销当前会话 |

### Channel 与 Agent 配置

| 方法 | 用途 |
|---|---|
| `createApiKey()` | 为账号创建模型供应商 API key |
| `refreshChannelToken()` | 刷新消息 channel token |
| `buildConfigPatch(channelToken, apiKey)` | 构建网关配置补丁 |
| `buildPostLoginConfig(channelToken)` | 便捷方法：创建 API key 并生成配置补丁 |

### AGP 客户端

| 方法 | 用途 |
|---|---|
| `start()` | 连接网关 |
| `stop()` | 断开连接并停止自动重连 |
| `sendMessageChunk(sessionId, promptId, text)` | 流式发送部分回复 |
| `sendTextResponse(sessionId, promptId, text)` | 发送最终文本回复 |
| `sendErrorResponse(sessionId, promptId, errorMessage)` | 发送错误回复 |
| `sendCancelledResponse(sessionId, promptId)` | 确认取消 |

## 安全说明

- 只使用你有权访问的账号和环境。
- 不要提交 token、JWT、channel token、API key 或用户标识。
- 扫码登录和 channel token 都应视为敏感凭证。
- 本项目用于个人研究、原型验证和开发者评估。

## 许可证

MIT
