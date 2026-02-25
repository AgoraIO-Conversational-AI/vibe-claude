# <img src="https://www.agora.io/en/wp-content/uploads/2024/01/Agora-logo-horizantal.svg" alt="Agora" width="120" style="vertical-align: middle;" /> Vibe Claude — Voice AI Agent

A voice AI agent app built entirely by Claude Code. No scaffolding needed — point
Claude at this README from an empty directory and it builds the whole app.

**Prompt:** `Build this Agora Voice AI Agent: https://github.com/AgoraIO-Conversational-AI/vibe-claude`

## Features

- **Real-time Voice** — Full-duplex audio via Agora RTC with echo cancellation,
  noise suppression, and auto gain control
- **Live Transcripts** — User and agent speech appears in the chat window as it
  happens (via RTC stream-message)
- **Text Chat** — Type a message and send it to the agent via Agora RTM
- **Agent Visualizer** — Animated orb shows agent state (idle, joining, listening, speaking, disconnected)
- **Customizable** — Settings for custom system prompt and greeting before connecting
- **Self-contained** — Express backend handles token generation, agent start, and hangup

## Quick Start

### 1. Install dependencies

```bash
npm install
```

### 2. Configure environment

```bash
cp .env.example .env
```

Fill in your keys in `.env`.

### 3. Run the app

```bash
npm run dev
```

Open the app, click **Connect**, and start talking.

## Architecture

```
Browser (React + Vite)
  │
  ├─ RTC audio ←→ Agora Conversational AI Agent
  ├─ RTC stream-message ← agent transcripts
  └─ RTM publish → text messages to agent

Express Backend
  ├─ GET  /api/check-env    — validates required env vars
  ├─ POST /api/start-agent  — generates RTC+RTM tokens, calls Agora ConvoAI API
  └─ POST /api/hangup-agent — stops the agent
```

## Environment Variables

| Secret              | Required    | Description                                                                                        |
| ------------------- | ----------- | -------------------------------------------------------------------------------------------------- |
| `APP_ID`            | ✅          | Agora App ID                                                                                       |
| `APP_CERTIFICATE`   | ❌ Optional | Agora App Certificate. Leave empty or set to `""` if token auth is disabled on your Agora project. |
| `AGENT_AUTH_HEADER` | ✅          | Agora REST API auth header (e.g. `Basic <base64>`)                                                 |
| `LLM_API_KEY`       | ✅          | API key for your LLM provider                                                                      |
| `LLM_URL`           | ❌ Optional | LLM endpoint URL (default: `https://api.openai.com/v1/chat/completions`)                           |
| `LLM_MODEL`         | ❌ Optional | LLM model name (default: `gpt-4o-mini`)                                                            |
| `TTS_VENDOR`        | ✅          | TTS vendor: `openai`, `elevenlabs`, `cartesia`, or `rime`                                          |
| `TTS_KEY`           | ✅          | API key for your TTS provider                                                                      |
| `TTS_VOICE_ID`      | ✅          | Voice ID for TTS (e.g. `astra` for Rime, `alloy` for OpenAI)                                       |

> **Note:** `APP_CERTIFICATE` is optional. If your Agora project does not have token authentication enabled, you do not need to set this secret at all.

## Implementation Details

### Backend: Express Server

Three routes. Use `dotenv` for env vars. Enable CORS. Serve the Vite build from `dist/` in production.

### Backend: `GET /api/check-env`

Validates all 6 required env vars are set via `process.env`. `APP_CERTIFICATE` is optional (reported but not required). Returns JSON:

```json
{ "configured": { "APP_ID": true, ... }, "ready": true, "missing": [] }
```

### Backend: `POST /api/start-agent`

Accepts optional POST body `{ prompt, greeting }`. Defaults: prompt = "You are a friendly voice assistant. Keep responses concise, around 10 to 20 words." greeting = "Hi there! How can I help you today?"

**Token generation** — combined RTC+RTM token using `agora-token`. When `APP_CERTIFICATE` is set, generates real tokens. Otherwise, falls back to using `APP_ID` as the token value for both the agent payload and the user response (required for RTM to work without certificate auth).

```typescript
import { AccessToken, ServiceRtc, ServiceRtm } from "agora-token";

function buildToken(
  channelName: string,
  uid: string,
  appId: string,
  appCertificate: string,
): string {
  const token = new AccessToken(appId, appCertificate, 86400);
  const rtcService = new ServiceRtc(channelName, uid);
  rtcService.addPrivilege(ServiceRtc.kPrivilegeJoinChannel, 86400);
  rtcService.addPrivilege(ServiceRtc.kPrivilegePublishAudioStream, 86400);
  token.addService(rtcService);
  const rtmService = new ServiceRtm(uid);
  rtmService.addPrivilege(ServiceRtm.kPrivilegeLogin, 86400);
  token.addService(rtmService);
  return token.build();
}
```

**Token fallback** — when `APP_CERTIFICATE` is not set, use `APP_ID` as the token value everywhere (agent payload `token` field AND the `token` returned to the frontend). Do NOT return `null` or empty string — RTM login requires a non-empty token value.

UIDs are strings: agent = `"100"`, user = `"101"`. Channel is random 10-char alphanumeric. Agent RTM UID = `"100-{channel}"`.

**Agent payload** — POST to `https://api.agora.io/api/conversational-ai-agent/v2/projects/{appId}/join`:

```json
{
  "name": "{channel}",
  "properties": {
    "channel": "{channel}",
    "token": "{agentToken}",
    "agent_rtc_uid": "100",
    "agent_rtm_uid": "100-{channel}",
    "remote_rtc_uids": ["*"],
    "enable_string_uid": false,
    "idle_timeout": 120,
    "advanced_features": {
      "enable_bhvs": true,
      "enable_rtm": true,
      "enable_aivad": true,
      "enable_sal": false
    },
    "llm": {
      "url": "{LLM_URL or https://api.openai.com/v1/chat/completions}",
      "api_key": "{LLM_API_KEY}",
      "system_messages": [{ "role": "system", "content": "{prompt}" }],
      "greeting_message": "{greeting}",
      "failure_message": "Sorry, something went wrong",
      "max_history": 32,
      "params": { "model": "{LLM_MODEL or gpt-4o-mini}" },
      "style": "openai"
    },
    "vad": { "silence_duration_ms": 300 },
    "asr": { "vendor": "ares", "language": "en-US" },
    "tts": "{ttsConfig}",
    "parameters": {
      "transcript": {
        "enable": true,
        "protocol_version": "v2",
        "enable_words": false
      }
    }
  }
}
```

**TTS config builder** — supports multiple vendors:

- **rime** (default): `{ vendor: "rime", params: { api_key, speaker: voiceId, modelId: "mistv2", lang: "eng", samplingRate: 16000, speedAlpha: 1.0 } }`
- **openai**: `{ vendor: "openai", params: { api_key, model: "tts-1", voice: voiceId, response_format: "pcm", speed: 1.0 } }`
- **elevenlabs**: `{ vendor: "elevenlabs", params: { key, model_id: "eleven_flash_v2_5", voice_id: voiceId, stability: 0.5, sample_rate: 24000 } }`
- **cartesia**: `{ vendor: "cartesia", params: { api_key, model_id: "sonic-3", sample_rate: 24000, voice: { mode: "id", id: voiceId } } }`

Returns: `{ appId, channel, token, uid, agentUid, agentRtmUid, agentId, success }`

### Backend: `POST /api/hangup-agent`

POST with `{ agentId }`. Calls `POST https://api.agora.io/api/conversational-ai-agent/v2/projects/{appId}/agents/{agentId}/leave` with `AGENT_AUTH_HEADER`.

### Frontend: React + Vite + TypeScript + Tailwind CSS

Install `agora-rtc-sdk-ng` and `agora-rtm` from npm. Both are browser-only SDKs — dynamically import them inside async functions at connect time, never at the top of the file.

### Frontend: RTC Voice + Transcript Listener

**Register ALL event listeners BEFORE `client.join()`.** The `stream-message` listener is critical — it receives ALL transcripts (both user speech and agent responses).

```typescript
const AgoraRTC = (await import("agora-rtc-sdk-ng")).default;
const client = AgoraRTC.createClient({ mode: "rtc", codec: "vp8" });

// Subscribe to agent audio
client.on("user-published", async (user, mediaType) => {
  if (mediaType !== "audio") return;
  await client.subscribe(user, "audio");
  user.audioTrack?.play();
  // Poll user.audioTrack.getVolumeLevel() to detect agent speaking
});

// CRITICAL: Transcript listener — agent sends ALL transcripts via RTC data stream
// Protocol v2 sends data as pipe-delimited base64 chunks: messageId|partIdx|partSum|base64data
// You MUST decode this format — raw JSON.parse() will NOT work.
const messageCache = new Map<string, { part_idx: number; content: string }[]>();

client.on("stream-message", (_uid: number, data: Uint8Array) => {
  try {
    const raw = new TextDecoder().decode(data);
    const parts = raw.split("|");

    let msg: any;
    if (parts.length === 4) {
      // v2 chunked format: messageId|partIdx|partSum|base64data
      const [msgId, partIdxStr, partSumStr, partData] = parts;
      const partIdx = parseInt(partIdxStr, 10);
      const partSum = partSumStr === "???" ? -1 : parseInt(partSumStr, 10);

      if (!messageCache.has(msgId)) messageCache.set(msgId, []);
      const chunks = messageCache.get(msgId)!;
      chunks.push({ part_idx: partIdx, content: partData });
      chunks.sort((a, b) => a.part_idx - b.part_idx);

      if (partSum === -1 || chunks.length < partSum) return; // wait for more chunks
      const base64 = chunks.map((c) => c.content).join("");
      msg = JSON.parse(atob(base64));
      messageCache.delete(msgId);
    } else if (raw.startsWith("{")) {
      msg = JSON.parse(raw); // fallback: raw JSON
    } else {
      return;
    }

    // msg.object = "user.transcription" or "assistant.transcription"
    // msg.text = transcript text
    // msg.turn_id = groups messages into turns
    // For user: msg.final = true means end of utterance
    // For assistant: msg.turn_status === 1 means end of turn
    if (msg.object && msg.text !== undefined) {
      const role =
        msg.object === "assistant.transcription" ? "assistant" : "user";
      const isFinal =
        role === "user" ? msg.final === true : msg.turn_status === 1;
      updateMessages(role, msg.turn_id, msg.text, isFinal);
    }
  } catch {
    /* ignore malformed data */
  }
});

await client.join(appId, channel, token, uid);
const micTrack = await AgoraRTC.createMicrophoneAudioTrack({
  encoderConfig: "high_quality_stereo",
  AEC: true,
  ANS: true,
  AGC: true,
});
await client.publish(micTrack);
```

**IMPORTANT: Transcripts arrive via RTC `stream-message`, NOT via RTM.** Protocol v2 encodes transcripts as base64 inside a pipe-delimited string (`messageId|partIdx|partSum|base64data`) — you MUST split on `|`, accumulate chunks by messageId, `atob()` the joined base64, then `JSON.parse()`. Raw `JSON.parse()` on the stream data will NOT work. Both user speech transcripts and agent response transcripts come through this single listener. The agent greeting also arrives here — do not hardcode it. Display transcripts as chat bubbles grouped by `turn_id`. Update in-place for partial transcripts, mark final when complete.

### Frontend: RTM Text Messaging

RTM is used **ONLY for sending text messages** from the user to the agent. Do NOT use `createStreamChannel`, `joinTopic`, `publishTopicMessage`, or `sendMessage`.

```typescript
const AgoraRTM = await import("agora-rtm");
const rtm = new AgoraRTM.default.RTM(appId, String(uid), {
  token: token ?? undefined,
});
await rtm.login(); // no arguments — token goes in constructor above

// Send text message — target is agent's RTM UID, NOT the channel name
const payload = JSON.stringify({ message: text, priority: "APPEND" });
await rtm.publish(agentRtmUid, payload, {
  customType: "user.transcription",
  channelType: "USER",
});

// Disconnect
await rtm.logout();
```

**IMPORTANT RTM rules:**

- Publish target is `agentRtmUid` (e.g. `"100-{channel}"`), NOT the channel name
- Message must be JSON: `{ "message": "text", "priority": "APPEND" }`
- Options must include `customType: "user.transcription"` and `channelType: "USER"`
- Show the user's message optimistically in the chat UI before sending
- Never `console.log()` the RTM client object — it causes `RangeError: Invalid string length` from circular references

### Frontend: UI Layout

**Pre-connection:** Centered orb, Connect button, collapsible settings panel for custom system prompt and greeting.

**Connected:** Split layout — left panel has animated pulsing orb (scales/glows when agent speaks), mute/unmute button, audio waveform bars (via Web Audio API AnalyserNode); right panel has scrolling chat messages with user/assistant bubbles, text input with send button. Header shows channel name, elapsed timer, and End button. Mobile collapses to single column.

**Agent orb states:** idle (dim, scaled down), joining (spinning border), listening (gentle pulse), talking (ping rings + glow + scale up), disconnected (dim).

### Frontend: Audio Visualization

Create an `AudioContext` and `AnalyserNode` from the microphone track's `MediaStreamTrack`. Poll `getByteFrequencyData()` via `requestAnimationFrame` and render as vertical bars.

## Video Avatar Client

Same as above, but also subscribe to the agent's video track and render it in a `<video>` element. The backend payload should additionally include video avatar configuration for the provider. The user still only sends audio (no camera needed).

## Tech Stack

- **Frontend:** React, Vite, TypeScript, Tailwind CSS
- **Backend:** Express
- **RTC SDK:** agora-rtc-sdk-ng v4.24+
- **RTM SDK:** agora-rtm v2.2+
- **Token gen:** agora-token (server-side)

## Reference

- [simple-backend](https://github.com/AgoraIO-Conversational-AI/agent-samples/tree/main/simple-backend) — Python reference implementation of the same token generation, start-agent, and hangup-agent logic
- [Agora Conversational AI Docs](https://docs.agora.io/en/conversational-ai/overview/product-overview)
- [Agora Console](https://console.agora.io)
- [Agent Samples](https://github.com/AgoraIO-Conversational-AI/agent-samples)
- [vibe-lovable](https://github.com/AgoraIO-Conversational-AI/vibe-lovable) — Lovable-generated version
- [vibe-v0](https://github.com/AgoraIO-Conversational-AI/vibe-v0) — v0-generated version

## License

MIT
