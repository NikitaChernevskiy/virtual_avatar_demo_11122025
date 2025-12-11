# Talking Avatar Demo (Single-File + Clear CONSTS)

A minimal, developer-friendly demo to run a real-time Talking Avatar via Azure Speech SDK in the browser. It focuses on:

- Avatar session setup with WebRTC relay
- Mic speech-to-text (STT) with auto language detection
- Text-to-speech (TTS) spoken by the avatar
- Simple client-side chat history (local assistant replies)
- Clear configuration via a single `CONSTS` block

This demo intentionally does not call Azure AI Foundry — assistant replies are stubbed locally so you can plug in your own backend later.

---

## Quick Start

1. Open the demo HTML in a modern browser (Edge/Chrome).
2. If you just want to try quickly:
   - Set `USE_SERVER_SPEECH_TOKEN=false` and `USE_SERVER_RELAY_TOKEN=false` in `CONSTS`.
   - Paste a Speech resource key into the "Speech Key" input on the page.
   - Click "Start Session" to bring the avatar online.
   - Type messages or start the microphone to see local replies spoken by the avatar.

For production and team workflows, prefer the server-backed setup below to avoid exposing keys in the browser.

---

## Where to Configure: CONSTS

All configuration lives at the top of the demo HTML in a single `CONSTS` object. Edit these values to match your environment and desired flow.

```js
const CONSTS = {
  // Speech service
  SPEECH_REGION: 'eastus',
  // Leave empty to rely on server token; paste only for local testing
  SPEECH_KEY: '',

  // TTS / Avatar
  TTS_VOICE: 'en-US-AndrewMultilingualNeural',
  AVATAR_CHARACTER: 'lisa',
  AVATAR_STYLE: 'casual-sitting',
  STT_LOCALES: ['en-US'],

  // Server endpoints (recommended for production)
  ENDPOINTS: {
    SPEECH_TOKEN: '/api/speech-token',       // GET → { token, region }
    AVATAR_RELAY_TOKEN: '/api/avatar-relay-token', // GET → { Urls, Username, Password }
    CHAT_PROXY: '/api/chat'                  // POST { messages } → { content } or stream
  },

  // Feature flags
  USE_SERVER_SPEECH_TOKEN: true,   // Use server-issued short-lived token
  USE_SERVER_RELAY_TOKEN: true,    // Use server-issued ICE credentials
  USE_LOCAL_REPLY: true            // Local stub replies; set false and wire CHAT_PROXY
};
```

- Change `SPEECH_REGION`, `TTS_VOICE`, `AVATAR_CHARACTER`, `AVATAR_STYLE`, `STT_LOCALES` as needed.
- For secure usage, keep `SPEECH_KEY` empty and enable `USE_SERVER_SPEECH_TOKEN` + `USE_SERVER_RELAY_TOKEN`.
- To demo quickly without backend, disable those flags and paste your Speech Key.

---

## What the Demo Does

- Initializes Azure Speech SDK and creates an `AvatarSynthesizer` using `AvatarConfig`.
- Retrieves WebRTC relay credentials (ICE servers) and establishes a `RTCPeerConnection` for avatar audio/video.
- Provides a clear message input and chat history on the page.
- Optionally enables mic STT; recognized phrases are appended as `user` messages.
- Generates short local `assistant` replies and speaks them via TTS.

No secrets are embedded in code when server endpoints are used — the browser obtains short-lived tokens and ICE credentials from your Functions under `api/`.

---

Optional next:
- Set `USE_LOCAL_REPLY=false` and wire the chat proxy as described below.

---

## Wiring Chat to Backend (Optional)

Replace local stubs with your server `/api/chat`:

```js
async function fetchAssistantReply(messages) {
  // Local stub mode
  if (CONSTS.USE_LOCAL_REPLY) {
    const lastUser = messages.slice().reverse().find(m => m.role === 'user');
    return generateLocalReply(lastUser ? lastUser.content : '');
  }

  // Server-backed mode
  const resp = await fetch(CONSTS.ENDPOINTS.CHAT_PROXY, {
    method: 'POST',
    headers: { 'content-type': 'application/json' },
    body: JSON.stringify({ messages })
  });
  if (!resp.ok) throw new Error('Chat proxy failed');
  const data = await resp.json(); // { content: '...' } or your format
  return data.content;
}
```

- Append streamed or final assistant text to the on-page history.
- When the avatar session is active, call `speak(assistantText)` to have responses spoken.

---

## Running Locally

### With Azure Static Web Apps CLI (Functions + Static)

```bash
# Install SWA CLI
npm install -g @azure/static-web-apps-cli

# Start the app and Functions locally
swa start
```

- The CLI proxies the static site and serverless functions, avoiding CORS hassles.
- In production, pushing to `main` triggers deployment per `.github/workflows/azure-static-web-apps-*.yml`.

### Simple Static Preview (no Functions)
- Open the HTML file directly in the browser.
- Use the vibe-coding mode (paste a Speech Key, disable server flags).

---

## Validations (Prod)

```bash
# Find your SWA hostname
az staticwebapp environment list -g <rg> -n <swa>

# Verify config and tokens
curl https://<hostname>/api/config
curl https://<hostname>/api/speech-token
curl https://<hostname>/api/avatar-relay-token

# Chat proxy (if wired)
curl -X POST -H 'content-type: application/json' \
  -d '{"messages":[{"role":"user","content":"Hello"}]}' \
  https://<hostname>/api/chat
```

---

## Common Pitfalls

- Token lifetime: Speech tokens are short-lived; always prefer `/api/speech-token` for real usage.
- Region mismatch: Ensure the Speech region in `CONSTS` or returned from `/api/speech-token` matches your resource.
- Permissions: Your Speech resource must support Avatar + TTS in the chosen region.
- CORS: Use SWA CLI or deploy Functions with the static site to avoid cross-origin issues.

---

## Extending the Demo

- Personal/Custom Voice: Populate SSML with `endpointId` or speaker profile if needed.
- Idle fallback: Add a local looping video when the remote stream disconnects.
- UI polish: Quick-reply chips, session state indicators, latency counters.

---
Model version note: The Agent controls the underlying model. To use, for example, gpt-4o (version: 2024-11-20), set that model/version in your Agent configuration in Azure AI Foundry Studio. No code change is needed.
- Ensure your microphone permissions are allowed by the browser when starting STT.
