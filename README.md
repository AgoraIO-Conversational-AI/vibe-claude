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

| Variable            | Required | Description                                                       |
| ------------------- | -------- | ----------------------------------------------------------------- |
| `APP_ID`            | Yes      | 32-char hex App ID from [Agora Console](https://console.agora.io) |
| `APP_CERTIFICATE`   | Yes      | App Certificate (enables token auth for RTC + RTM)                |
| `AGENT_AUTH_HEADER` | Yes      | `Basic <base64(customerKey:customerSecret)>` for the REST API     |
| `LLM_API_KEY`       | Yes      | OpenAI API key (or compatible provider)                           |
| `TTS_VENDOR`        | Yes      | `rime`, `openai`, `elevenlabs`, or `cartesia`                     |
| `TTS_KEY`           | Yes      | API key for your TTS vendor                                       |
| `TTS_VOICE_ID`      | Yes      | Voice ID (e.g. `astra` for Rime, `alloy` for OpenAI)              |
| `LLM_URL`           | No       | Custom LLM endpoint (defaults to OpenAI)                          |
| `LLM_MODEL`         | No       | Model name (defaults to `gpt-4o-mini`)                            |

## Implementation Details

### Backend: Express Server

Three routes. Use `dotenv` for env vars. Enable CORS. Serve the Vite build from `dist/` in production.

### Backend: `GET /api/check-env`

Validates all 7 required env vars are set via `process.env`. Returns JSON:

```json
{ "configured": { "APP_ID": true, ... }, "ready": true, "missing": [] }
```

### Backend: `POST /api/start-agent`

Accepts optional POST body `{ prompt, greeting }`. Defaults: prompt = "You are a friendly voice assistant. Keep responses concise, around 10 to 20 words." greeting = "Hi there! How can I help you today?"

**Token generation** — combined RTC+RTM token using `agora-token`:

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

### Frontend: RTC Voice

```typescript
const AgoraRTC = (await import("agora-rtc-sdk-ng")).default;
const client = AgoraRTC.createClient({ mode: "rtc", codec: "vp8" });
client.on("user-published", async (user, mediaType) => {
  if (mediaType !== "audio") return;
  await client.subscribe(user, "audio");
  user.audioTrack?.play();
  // Poll user.audioTrack.getVolumeLevel() to detect agent speaking
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

### Frontend: Transcript Listener

The agent sends transcripts via RTC data stream:

```typescript
client.on("stream-message", (_uid: number, data: Uint8Array) => {
  const text = new TextDecoder().decode(data);
  const msg = JSON.parse(text);
  // msg.object = "user.transcription" or "assistant.transcription"
  // msg.text = transcript text
  // msg.turn_id = groups messages into turns
  // For user: msg.final = true means end of utterance
  // For assistant: msg.turn_status === 1 means end of turn
});
```

Display transcripts as chat bubbles grouped by `turn_id`. Update in-place for partial transcripts, mark final when complete. No hardcoded greeting — the agent sends its greeting via the transcript stream.

### Frontend: RTM Text Messaging

```typescript
const AgoraRTM = await import("agora-rtm");
const rtm = new AgoraRTM.default.RTM(appId, String(uid), {
  token: token ?? undefined,
});
await rtm.login();

// Send text message to agent
await rtm.publish(channel, text);

// Disconnect
await rtm.logout();
```

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
