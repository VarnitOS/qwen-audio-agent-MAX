# Gemini Live Connection Debug Notes

**Date:** 2026-11-02  
**Branch:** debug/gemini-live-connection

## Setup

- Forked repo linked as global `qwenaudio` via `npm link` from `qwen-audio-agent-MAX/`
- Node: v24.22.2 set as nvm default (was v24.11.1, upgraded to meet `^24.15.0` requirement)
- Gateway runs from local source at `~/Documents/MY_Profile/agentmax/qwen-audio-agent-MAX/`
- Config: `~/.config/qwaudio/config.env`

## Current Config

```dotenv
QWEN_AUDIO_REALTIME_PROVIDER=google-live
QWEN_AUDIO_REALTIME_API_KEY=<redacted>
# QWEN_AUDIO_REALTIME_MODEL not set — defaults to gemini-3.8-live (correct, it's in the catalog)
AGENT_PROTOCOL=claude
```

## Problem

Gateway connects to Google Live WebSocket but closes immediately (`connectedMs:0`).  
Repeats on a ~10s retry loop. No `realtime.connected` or `realtime.connect_failed` ever logged.

## What We Ruled Out

| Hypothesis | Status |
|---|---|
| Wrong model name (`gemini-3.8-live`) | **Ruled out** — model exists and supports `bidiGenerateContent` |
| Invalid API key | **Ruled out** — REST API call returns valid model list |
| Missing `inputAudioTranscription` / `outputAudioTranscription` | **Ruled out** — direct WS test with these fields gets `setupComplete` |
| Invalid tool declarations in setup | **Inconclusive** — not yet tested with real gateway tool list |
| Node version incompatibility | **Unlikely** — same behaviour on v24.11.1 and v24.22.2 |

## Direct WS Test (works)

Minimal setup message (`model`, `generationConfig`, `systemInstruction`, `tools:[]`, `outputAudioTranscription`, `inputAudioTranscription`) receives `setupComplete` + `sessionResumptionUpdate` successfully.

## Root Cause Still Open

The gateway never logs the WS close code/reason — `ws.on('close', () => ...)` doesn't receive the code/reason because it was originally declared with no parameters. Debug instrumentation was added to capture it but the launchd service was running the old global binary, not the patched local source.

**Next step:** With the gateway now properly linked to local source, open `http://127.0.0.1:3101` in the browser (which triggers a Google Live WS connection), then check `/tmp/ws_close_debug.log` for the actual close code Google is sending back.

## Gateway Notes

- Runs as a launchd service: `com.qwen-audio-agent.gateway`
- Plist: `~/Library/LaunchAgents/com.qwen-audio-agent.gateway.plist`
- To stop permanently: `launchctl unload ~/Library/LaunchAgents/com.qwen-audio-agent.gateway.plist`
- To start from local source directly: `node server/src/index.mjs`
- Old global binary was at `~/.nvm/versions/node/v24.11.1/lib/node_modules/qwen-audio-agent/` (now symlinked to local source)
