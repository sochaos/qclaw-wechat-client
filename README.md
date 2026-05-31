# qclaw-wechat-client

[中文文档](./README.zh-CN.md)

A TypeScript client and protocol lab for connecting personal AI agents to a WeChat-style messaging channel through QR-code login, channel tokens, and a real-time agent gateway protocol.

This repository is maintained as a practical developer project for experimenting with:

- authenticated user consent flows based on QR-code login
- message delivery between chat users and local/cloud AI agents
- WebSocket-based agent gateway protocols
- multi-agent routing for tools such as Claude, Codex, Gemini, OpenClaw, and custom OpenAI-compatible agents
- safe local development workflows for personal automation and agent UX research

## Why This Exists

AI agents are much more useful when they can live inside the messaging tools people already use. This project explores the infrastructure layer needed for that experience: login, session state, channel tokens, message streaming, cancellation, reconnects, and final response delivery.

The current implementation focuses on a QClaw/OpenClaw-compatible messaging flow and is intended for personal research, prototyping, and developer evaluation.

## Current Capabilities

- WeChat QR-code login flow helpers
- token and session management helpers
- API client for account, device, invite, and channel-token operations
- AGP WebSocket client for real-time agent messages
- streaming response helpers for agent replies
- reconnect and heartbeat handling
- TypeScript types for prompts, cancellations, content blocks, and responses
- demo flow for running a simple echo agent

## Trial / Evaluation Context

I use this repository as a public technical reference when applying for developer previews or trial access to AI agent platforms. It demonstrates an active project around:

- agent messaging infrastructure
- personal productivity agents
- local-to-cloud agent bridges
- authenticated chat interfaces
- practical TypeScript SDK design

The goal is to evaluate new AI platforms in a real integration setting rather than only through toy prompts.

## Install

```bash
npm install qclaw-wechat-client
# or
pnpm add qclaw-wechat-client
```

## Development

```bash
pnpm install
pnpm build
pnpm typecheck
pnpm demo
```

## Quick Start

```typescript
import { QClawClient } from "qclaw-wechat-client";
import type { WxLoginStateData, WxLoginData } from "qclaw-wechat-client";

const client = new QClawClient({ env: "production" });

const guid = "local-device-id";

// 1. Create a login state.
const stateRes = await client.getWxLoginState({ guid });
const state = QClawClient.unwrap<WxLoginStateData>(stateRes)?.state;

// 2. Show this QR login URL to the user.
const qrUrl = client.buildWxLoginUrl(state!);
console.log("Scan to authorize:", qrUrl);

// 3. Exchange the OAuth callback code for a session.
const loginRes = await client.wxLogin({ guid, code: authCode, state: state! });

// 4. Build an agent-channel config patch.
const channelToken = QClawClient.unwrap<WxLoginData>(loginRes)?.openclaw_channel_token;
const config = await client.buildPostLoginConfig(channelToken!);
```

## AGP WebSocket Client

The package includes an AGP client for real-time agent messaging. The server sends `session.prompt` events when a user sends a message, and the client responds with streaming `session.update` events plus a final `session.promptResponse`.

```typescript
import { AGPClient } from "qclaw-wechat-client";

const client = new AGPClient(
  {
    url: "wss://mmgrcalltoken.3g.qq.com/agentwss",
    token: channelToken,
  },
  {
    onConnected() {
      console.log("Connected. Waiting for prompts...");
    },
    onPrompt(msg) {
      const { session_id, prompt_id, content } = msg.payload;
      const text = content.map((block) => block.text).join("");
      console.log("User:", text);

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

## Main APIs

### Authentication

| Method | Purpose |
|---|---|
| `getWxLoginState({ guid })` | Create CSRF/login state for QR login |
| `buildWxLoginUrl(state)` | Build the QR-code login URL |
| `wxLogin({ guid, code, state })` | Exchange OAuth callback code for a session |
| `getUserInfo({ guid })` | Fetch the current user profile |
| `wxLogout({ guid })` | Invalidate the current session |

### Channel and Agent Setup

| Method | Purpose |
|---|---|
| `createApiKey()` | Create a model-provider API key for the account |
| `refreshChannelToken()` | Refresh the messaging channel token |
| `buildConfigPatch(channelToken, apiKey)` | Build a gateway config patch |
| `buildPostLoginConfig(channelToken)` | Convenience helper: create API key and config patch |

### AGP Client

| Method | Purpose |
|---|---|
| `start()` | Connect to the gateway |
| `stop()` | Disconnect and stop reconnecting |
| `sendMessageChunk(sessionId, promptId, text)` | Stream a partial response |
| `sendTextResponse(sessionId, promptId, text)` | Send the final text response |
| `sendErrorResponse(sessionId, promptId, errorMessage)` | Send an error response |
| `sendCancelledResponse(sessionId, promptId)` | Acknowledge cancellation |

## Safety Notes

- Use only accounts and environments you are authorized to access.
- Do not commit tokens, JWTs, channel tokens, API keys, or user identifiers.
- Treat QR-code login and channel tokens as sensitive credentials.
- This project is for personal research, prototyping, and developer evaluation.

## License

MIT
