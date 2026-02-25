# <img src="https://www.agora.io/en/wp-content/uploads/2024/01/Agora-logo-horizantal.svg" alt="Agora" width="120" style="vertical-align: middle;" /> Vibe Claude — Voice AI Agent

This folder exists for parity with `vibe-lovable` and `vibe-v0`. Unlike those
projects, there's no generated code here — just prompts. Claude Code doesn't
need scaffolding; give it one of the prompts below and it builds the whole app.

## Voice Client

Paste this into Claude Code from an empty directory:

> Build a real-time Voice AI Agent web app using Agora Conversational AI.
> React + Vite + TypeScript + Tailwind CSS. The app should:
>
> 1. On load, call a `/api/check-env` endpoint to verify required env vars
>    are set: `APP_ID`, `APP_CERTIFICATE`, `AGENT_AUTH_HEADER`, `LLM_API_KEY`,
>    `TTS_VENDOR`, `TTS_KEY`, `TTS_VOICE_ID`. Show which are missing.
>
> 2. Show a Connect button. On click, POST to `/api/start-agent` with optional
>    `{ prompt, greeting }`. The backend generates an Agora RTC+RTM token using
>    the `agora-token` npm package (AccessToken with ServiceRtc + ServiceRtm),
>    then calls the Agora ConvoAI REST API
>    (`POST https://api.agora.io/api/conversational-ai-agent/v2/projects/{appId}/join`)
>    to start the agent. Return `{ appId, channel, token, uid, agentUid, agentId }`.
>
> 3. On the client, dynamically import `agora-rtc-sdk-ng` (browser-only).
>    Join the RTC channel, create a microphone audio track with AEC/ANS/AGC,
>    publish it. Subscribe to the agent's audio and play it. Listen for the
>    `stream-message` event to receive transcript JSON from the agent
>    (`{ object, text, turn_id, final/turn_status }`). Display transcripts
>    as chat bubbles, grouping by `turn_id`.
>
> 4. Dynamically import `agora-rtm`. Login, then expose a `sendTextMessage()`
>    that publishes to the channel so the user can type messages to the agent.
>
> 5. Show an animated orb that pulses when the agent is speaking (detect via
>    remote audio volume polling). Show a waveform visualizer for the local mic.
>    Mute/unmute button. End button that leaves RTC, logs out RTM, and POSTs
>    to `/api/hangup-agent` with the `agentId`.
>
> 6. Use Express (or Next.js API routes, or Supabase Edge Functions) for the
>    backend. The ConvoAI agent payload should include: `enable_rtm: true`,
>    `transcript.enable: true`, `protocol_version: "v2"`, ASR vendor `ares`,
>    and configurable TTS (rime/openai/elevenlabs/cartesia).

## Video Avatar Client

> Same as above, but add a video avatar. After the agent joins, also subscribe
> to the agent's video track and render it in a `<video>` element next to or
> behind the orb. The backend payload should include `enable_video: true` and
> a `video` config section with the avatar provider settings. The user still
> only sends audio (no camera needed).

## Environment Variables

| Variable | Description |
|----------|-------------|
| `APP_ID` | Agora App ID |
| `APP_CERTIFICATE` | Agora App Certificate |
| `AGENT_AUTH_HEADER` | `Basic <base64(customerKey:customerSecret)>` |
| `LLM_API_KEY` | OpenAI API key (or compatible) |
| `TTS_VENDOR` | `rime`, `openai`, `elevenlabs`, or `cartesia` |
| `TTS_KEY` | TTS provider API key |
| `TTS_VOICE_ID` | Voice ID (e.g. `astra`, `alloy`) |

## Reference

- [Agora Conversational AI Docs](https://docs.agora.io/en/conversational-ai/overview/product-overview)
- [Agent Samples](https://github.com/AgoraIO-Conversational-AI/agent-samples) — production-grade reference implementations
- [vibe-lovable](https://github.com/AgoraIO-Conversational-AI/vibe-lovable) — Lovable-generated version
- [vibe-v0](https://github.com/AgoraIO-Conversational-AI/vibe-v0) — v0-generated version

## License

MIT
