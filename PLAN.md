# hermes-voice — Live voice client for Hermes Agent

> **Status:** planning / pre-implementation. This document is the design + reference code for v0.1.
> The goal is a phone-installable web app that talks to any Hermes Agent profile
> (coder, scout, vector, banel, …) with real-time voice, comparable to ChatGPT's voice mode.

---

## 1. Goals & non-goals

**Goals**
- Push-to-talk (and optional always-on) voice conversation with **any** running Hermes profile.
- End-to-end "time to first voice" under ~1.5 s.
- Real **barge-in** (interrupt the assistant mid-sentence).
- Self-hosted: server stays on Octa's home box, phone connects through a Cloudflare tunnel.
- Zero per-minute cost in the recommended path.
- Per-message text transcript visible on screen (so the user can correct STT errors with a tap).
- Reuse the existing Hermes components: `gateway/platforms/api_server.py` (SSE), faster-whisper STT, Edge TTS, `cloudflared`.

**Non-goals (v0.1)**
- Multi-user / multi-tenant.
- iOS Safari quirks (Android Chrome is the target; iOS is a stretch goal).
- Replacing the brain with a speech-to-speech vendor (OpenAI Realtime, Gemini Live).
- WebRTC media transport (won't traverse the Cloudflare quick tunnel; use WebSocket if/when we go sub-second).
- New profile runtime — we are a *client* to existing profiles.

---

## 2. Why a PWA (not a native app, not Telegram, not WebRTC)

A short version of the research that landed us here:

| Option | Verdict |
|---|---|
| **Android-Chrome PWA** using `webkitSpeechRecognition` + SSE + per-sentence server TTS | **Pick this.** ~1 day, $0, reuses everything, real barge-in, installs to the home screen. |
| WebSocket streaming pipeline (VAD + faster-whisper + Kokoro local TTS) | Phase 2 polish once the PWA proves the UX. |
| OpenAI Realtime / Gemini Live | Replaces Axi's brain (minimax-m3 + 90-iter tool loop). Wrong trade. |
| True WebRTC + Pipecat/LiveKit | Overkill, and WebRTC media can't traverse the Cloudflare quick tunnel. |
| Telegram voice tweaks | Telegram has no realtime audio. Park as parallel polish only. |

The single most important correction: **WebRTC and the Cloudflare quick tunnel are incompatible** — quick tunnels proxy HTTP and WebSocket only. If we ever want sub-second server-side audio, the transport is WebSocket, not WebRTC.

---

## 3. Architecture (v0.1)

```
┌────────────────────────┐                                  ┌──────────────────────────────────┐
│  Android Chrome (PWA)  │                                  │   home box (Ubuntu 24.04)        │
│                        │   HTTPS  ◄──── trycloudflare ───►│                                  │
│  webkitSpeechRecogni…  │   WebSocket/SSE                   │   cloudflared tunnel             │
│   └─ STT (browser)     │                                  │     └─ forwards to :8642         │
│                        │                                  │   api_server.py (Hermes Coder    │
│  AudioWorklet (later)  │                                  │   profile, OpenAI-compatible)    │
│   └─ PCM frames        │                                  │     ├─ POST /sessions/.../chat   │
│                        │                                  │     │   (SSE, streaming tokens)  │
│  <audio> queue         │                                  │     ├─ GET  /tts?text=…          │
│   └─ per-sentence TTS  │                                  │     │   (Edge TTS, audio/ogg)    │
│                        │                                  │     ├─ POST /chat/cancel         │
│  service worker (PWA)  │                                  │     └─ GET  /voice  (static PWA) │
└────────────────────────┘                                  │                                  │
                                                             │   AIAgent.run_conversation()     │
                                                             │     (minimax-m3, tool loop)      │
                                                             │                                  │
                                                             │   Other profiles (scout/vector/  │
                                                             │   banel) ── separate ports      │
                                                             └──────────────────────────────────┘
```

**Barge-in path (client-side, no server cooperation needed for v0.1):**
1. While the assistant is speaking, the user starts talking.
2. `webkitSpeechRecognition.onspeechstart` fires (or first interim `onresult`).
3. Client calls `audio.pause()` + `speechSynthesis.cancel()` + `AbortController.abort()` on the SSE fetch.
4. Client POSTs `/chat/cancel` to drop the in-flight generation.
5. UI shows "interrupted" briefly; assistant can re-engage on the next user utterance.

The "interrupt mid-tool-call" edge case is acknowledged but deferred: cancelling a half-executed tool call is genuinely hard; the current tool call completes, then the loop notices the cancel token and exits cleanly.

---

## 4. Repository layout (target v0.1)

```
hermes-voice/
├── README.md                    # short pitch + "see PLAN.md"
├── PLAN.md                      # this file
├── LICENSE                      # MIT
├── client/                      # the PWA, self-contained, no build step
│   ├── index.html               # entry, ~250 lines
│   ├── app.js                   # STT + SSE + TTS + barge-in
│   ├── style.css                # minimal mobile-first
│   ├── manifest.webmanifest     # PWA install metadata
│   ├── sw.js                    # service worker (cache + offline shell)
│   └── icons/
│       ├── icon-192.png
│       └── icon-512.png
├── server/                      # the thin Hermes-side glue (patches to live repo, not a fork)
│   ├── README.md
│   ├── api_server_patches.py    # drop-in additions to gateway/platforms/api_server.py
│   ├── config_snippet.yaml      # additions to profiles/<name>/config.yaml
│   └── profile_router.py        # (phase 1.5) per-profile routing
├── deploy/
│   ├── cloudflared-quick.sh     # launch a quick tunnel to :8642
│   ├── cloudflared-named.md     # how to set up a stable named tunnel
│   └── systemd/
│       └── hermes-voice.service # optional: tunnel on boot
└── docs/
    ├── security.md              # tunnel auth, mic over HTTPS, rate limits
    ├── latency-budget.md        # where the ~1 s goes, and how to shave it
    └── phase-2-websocket.md     # design notes for the upgrade
```

For v0.1 we **do not** fork Hermes. The `server/` folder ships diffs/snippets that the
user applies to their live Hermes checkout. Forking can come later if the patches grow.

---

## 5. Reference implementation — the PWA

`client/index.html` (skeleton — not the final markup, but the right shape):

```html
<!doctype html>
<html lang="en">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1, viewport-fit=cover" />
  <title>Axi</title>
  <link rel="manifest" href="manifest.webmanifest" />
  <link rel="stylesheet" href="style.css" />
</head>
<body>
  <header>
    <select id="profile" aria-label="Hermes profile">
      <option value="coder">coder</option>
      <option value="scout">scout</option>
      <option value="vector">vector</option>
      <option value="banel">banel</option>
    </select>
    <span id="status" class="status">disconnected</span>
  </header>

  <main id="transcript" aria-live="polite"></main>

  <footer>
    <button id="ptt" class="ptt" aria-label="Hold to talk">●  Hold to talk</button>
    <label class="opt">
      <input type="checkbox" id="handsfree" /> Hands-free
    </label>
  </footer>

  <script src="app.js" type="module"></script>
</body>
</html>
```

`client/app.js` (the actual logic — this is the design we are committing to):

```js
// ─── config ─────────────────────────────────────────────────────────────────
const API_BASE = '';                 // same origin (served by Hermes /voice)
const sessionId = crypto.randomUUID();
let profile = 'coder';

// ─── DOM ────────────────────────────────────────────────────────────────────
const $ = (id) => document.getElementById(id);
const transcript = $('transcript');
const ptt = $('ptt');
const status = $('status');
const profileSel = $('profile');
const handsfree = $('handsfree');

profileSel.onchange = () => (profile = profileSel.value);

// ─── transcript helpers ─────────────────────────────────────────────────────
function addLine(role, text) {
  const el = document.createElement('div');
  el.className = `line ${role}`;
  el.textContent = text;
  transcript.appendChild(el);
  transcript.scrollTop = transcript.scrollHeight;
  return el;
}

// ─── STT (browser, streaming via webkitSpeechRecognition) ───────────────────
const SR = window.SpeechRecognition || window.webkitSpeechRecognition;
let recognizer = null;
let srAbort = null;

function startRecognizer() {
  if (!SR) { setStatus('STT unsupported in this browser'); return null; }
  recognizer = new SR();
  recognizer.continuous = true;
  recognizer.interimResults = true;
  recognizer.lang = 'en-US';

  let interim = '';
  let finalBuf = '';

  recognizer.onresult = (ev) => {
    interim = '';
    for (let i = ev.resultIndex; i < ev.results.length; i++) {
      const r = ev.results[i];
      if (r.isFinal) finalBuf += r[0].transcript;
      else interim += r[0].transcript;
    }
    if (interim) setStatus(`hearing: ${interim.trim().slice(0, 60)}…`);
    if (finalBuf) {
      const text = finalBuf.trim();
      finalBuf = '';
      if (text) sendUtterance(text);
    }
  };
  recognizer.onerror = (e) => { console.warn('SR error', e); };
  recognizer.onend = () => { if (handsfree.checked) recognizer.start(); };

  recognizer.start();
  return recognizer;
}

function stopRecognizer() {
  recognizer?.stop();
  recognizer = null;
}

// ─── LLM streaming + per-sentence TTS playback ──────────────────────────────
let inflightAbort = null;
const audioQueue = [];
let audioEl = null;

function setStatus(s) { status.textContent = s; }

async function sendUtterance(text) {
  // Barge-in: cancel any in-flight response before starting a new one.
  if (inflightAbort) { inflightAbort.abort(); inflightAbort = null; }
  if (audioEl) { audioEl.pause(); audioEl = null; }
  audioQueue.length = 0;

  addLine('user', text);
  const assistantEl = addLine('assistant', '');

  inflightAbort = new AbortController();
  let acc = '';
  let sentenceBuf = '';

  try {
    const r = await fetch(`${API_BASE}/api/sessions/${sessionId}/chat?profile=${profile}`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json', 'Accept': 'text/event-stream' },
      body: JSON.stringify({ message: text, stream: true }),
      signal: inflightAbort.signal,
    });
    if (!r.ok || !r.body) throw new Error(`HTTP ${r.status}`);

    // Best-effort cancel ping (server may ignore if no cancel token yet).
    fetch(`${API_BASE}/api/sessions/${sessionId}/cancel?profile=${profile}`,
      { method: 'POST' }).catch(() => {});

    const reader = r.body.getReader();
    const dec = new TextDecoder();
    let buf = '';
    while (true) {
      const { value, done } = await reader.read();
      if (done) break;
      buf += dec.decode(value, { stream: true });
      let idx;
      while ((idx = buf.indexOf('\n\n')) !== -1) {
        const frame = buf.slice(0, idx);
        buf = buf.slice(idx + 2);
        const line = frame.split('\n').find(l => l.startsWith('data:'));
        if (!line) continue;
        const payload = line.slice(5).trim();
        if (payload === '[DONE]') continue;
        try {
          const tok = JSON.parse(payload);
          const piece = tok.delta ?? tok.content ?? '';
          if (!piece) continue;
          acc += piece;
          assistantEl.textContent = acc;
          sentenceBuf += piece;
          // Sentence boundary -> enqueue TTS.
          if (/[.!?…]\s/.test(sentenceBuf)) {
            const s = sentenceBuf.trim();
            sentenceBuf = '';
            if (s) enqueueTts(s);
          }
        } catch { /* ignore non-JSON frames */ }
      }
    }
    if (sentenceBuf.trim()) enqueueTts(sentenceBuf.trim());
  } catch (e) {
    if (e.name !== 'AbortError') {
      console.error(e);
      setStatus(`error: ${e.message}`);
    }
  } finally {
    inflightAbort = null;
  }
}

function enqueueTts(text) {
  audioQueue.push(text);
  if (!audioEl) playNext();
}

function playNext() {
  if (!audioQueue.length) { audioEl = null; return; }
  const text = audioQueue.shift();
  const url = `${API_BASE}/tts?text=${encodeURIComponent(text)}&profile=${profile}`;
  const a = new Audio(url);
  audioEl = a;
  a.onended = playNext;
  a.onerror = playNext;
  a.play().catch(playNext);
}

// ─── PTT button + hands-free toggle ─────────────────────────────────────────
ptt.addEventListener('pointerdown', () => startRecognizer());
ptt.addEventListener('pointerup',   () => stopRecognizer());
ptt.addEventListener('pointerleave', () => stopRecognizer());
handsfree.onchange = () => { if (handsfree.checked) startRecognizer(); else stopRecognizer(); };

// Initial mic prompt on first interaction.
document.addEventListener('pointerdown', function once() {
  document.removeEventListener('pointerdown', once);
  if (handsfree.checked) startRecognizer();
});
```

`client/manifest.webmanifest`:

```json
{
  "name": "Axi",
  "short_name": "Axi",
  "start_url": "/voice",
  "scope": "/voice",
  "display": "standalone",
  "background_color": "#0b0d10",
  "theme_color": "#0b0d10",
  "icons": [
    { "src": "/voice/icons/icon-192.png", "sizes": "192x192", "type": "image/png" },
    { "src": "/voice/icons/icon-512.png", "sizes": "512x512", "type": "image/png" }
  ]
}
```

`client/sw.js` (offline shell — install the PWA so the home-screen launch is snappy):

```js
const CACHE = 'axi-v1';
const SHELL = ['/voice/', '/voice/app.js', '/voice/style.css', '/voice/manifest.webmanifest'];
self.addEventListener('install', (e) => e.waitUntil(caches.open(CACHE).then(c => c.addAll(SHELL))));
self.addEventListener('activate', (e) => e.waitUntil(self.clients.claim()));
self.addEventListener('fetch', (e) => {
  if (e.request.method !== 'GET') return;
  e.respondWith(
    caches.match(e.request).then(hit => hit ||
      fetch(e.request).then(res => {
        const copy = res.clone();
        caches.open(CACHE).then(c => c.put(e.request, copy)).catch(() => {});
        return res;
      }).catch(() => hit))
  );
});
```

---

## 6. Reference implementation — Hermes-side glue

These are *patches* to the live Hermes repo (`~/.hermes/`), shipped as
diffs in `server/`. They are designed to be applied with `git apply` and
unapplied with `git apply -R`. No fork required for v0.1.

### 6.1 Config additions (per profile, e.g. `profiles/coder/config.yaml`)

```yaml
# enable the HTTP/SSE API server for this profile
api_server:
  enabled: true
  host: 127.0.0.1
  port: 8642          # coder -> 8642, scout -> 8643, vector -> 8644, banel -> 8645
  voice_path: /voice  # serves the PWA static files
  require_token: true
  access_token: ${HERMES_VOICE_TOKEN}   # a long random string in .env
```

### 6.2 Endpoints added to `gateway/platforms/api_server.py`

```python
# ─── additions only; merge into the existing FastAPI app ──────────────────
import os, re, asyncio
from fastapi import APIRouter, HTTPException, Request, Query
from fastapi.responses import FileResponse, Response, StreamingResponse

router = APIRouter()
TOKEN = os.environ["HERMES_VOICE_TOKEN"]

# 1) Auth gate (applied to the router, NOT to /voice or /tts or /health)
@router.middleware("http")
async def _auth(request: Request, call_next):
    if not request.url.path.startswith("/api/"):
        return await call_next(request)
    if request.headers.get("x-axi-token") != TOKEN and \
       request.query_params.get("token") != TOKEN:
        raise HTTPException(status_code=401, detail="bad token")
    return await call_next(request)

# 2) Serve the PWA at /voice
@router.get("/voice")
@router.get("/voice/{path:path}")
async def voice(path: str = ""):
    root = os.path.join(os.path.dirname(__file__), "..", "..", "..", "hermes-voice", "client")
    full = os.path.normpath(os.path.join(root, path or "index.html"))
    if not full.startswith(os.path.abspath(root)):
        raise HTTPException(404)
    if os.path.isdir(full):
        full = os.path.join(full, "index.html")
    if not os.path.exists(full):
        raise HTTPException(404)
    media = "text/html" if full.endswith(".html") else None
    return FileResponse(full, media_type=media)

# 3) TTS proxy (wraps the existing text_to_speech_tool)
@router.get("/tts")
async def tts(text: str = Query(..., max_length=2000), profile: str = "coder"):
    # Delegate to the same TTS code path the Telegram voice-note path uses.
    from hermes_voice.tts_bridge import synthesize  # tiny shim, see 6.3
    audio = await asyncio.to_thread(synthesize, text, profile)
    return Response(content=audio, media_type="audio/ogg")

# 4) Cancel an in-flight response (best-effort; the agent loop notices the token)
@router.post("/api/sessions/{sid}/cancel")
async def cancel(sid: str, profile: str = "coder"):
    from hermes_voice.cancel_registry import request_cancel
    request_cancel(profile, sid)
    return {"ok": True}

# 5) Health
@router.get("/health")
async def health():
    return {"ok": True}
```

### 6.3 Tiny TTS shim — `hermes-voice/server/tts_bridge.py`

```python
"""Bridge to the existing Hermes TTS path. Replace the import with the real
text-to-speech tool used by the Telegram gateway (edge-tts, ElevenLabs, …).
"""
import asyncio
from .config import load_profile

async def synthesize(text: str, profile: str) -> bytes:
    cfg = load_profile(profile)
    provider = cfg["tts"]["provider"]
    if provider == "edge":
        import edge_tts
        voice = cfg["tts"].get("voice", "en-US-AvaMultilingualNeural")
        communicate = edge_tts.Communicate(text, voice)
        buf = bytearray()
        async for chunk in communicate.stream():
            if chunk["type"] == "audio":
                buf += chunk["data"]
        return bytes(buf)
    raise NotImplementedError(f"tts provider not bridged: {provider}")
```

### 6.4 Cancel registry — `hermes-voice/server/cancel_registry.py`

```python
"""Per-profile, per-session cancellation tokens. The agent loop checks the
token between tool calls. Cancelling mid-tool-call is acknowledged as best-effort.
"""
import threading
from typing import Dict, Set

_lock = threading.Lock()
_flags: Dict[str, Set[str]] = {}  # profile -> set of session ids

def request_cancel(profile: str, session_id: str) -> None:
    with _lock:
        _flags.setdefault(profile, set()).add(session_id)

def is_cancelled(profile: str, session_id: str) -> bool:
    with _lock:
        return session_id in _flags.get(profile, set())

def clear(profile: str, session_id: str) -> None:
    with _lock:
        _flags.get(profile, set()).discard(session_id)
```

The agent loop should call `is_cancelled()` at the top of each tool iteration
and `clear()` when the loop exits (any branch).

---

## 7. Per-profile routing

Each Hermes profile has its own gateway + API server. v0.1 ships a **profile router**
that fronts them all on a single port and dispatches by `?profile=` query param
or `X-Hermes-Profile` header. The router is ~80 lines of Python and runs as a
standalone process; cloudflared points at the router's port.

```
phone ──► cloudflared ──► :8642  (profile router)
                            │
                            ├─ coder  :8642  (real api_server.py)
                            ├─ scout   :8653
                            ├─ vector  :8654
                            └─ banel   :8655
```

`server/profile_router.py` (sketch — the real one is a thin aiohttp/Starlette proxy):

```python
from starlette.applications import Starlette
from starlette.responses import Response, JSONResponse
from starlette.routing import Route
import httpx

PROFILES = {
    "coder":  "http://127.0.0.1:8642",
    "scout":  "http://127.0.0.1:8653",
    "vector": "http://127.0.0.1:8654",
    "banel":  "http://127.0.0.1:8655",
}

async def proxy(request: Request):
    profile = request.query_params.get("profile") or request.headers.get("x-hermes-profile", "coder")
    if profile not in PROFILES:
        return JSONResponse({"error": "unknown profile"}, status_code=400)
    target = PROFILES[profile]
    # Strip the profile selector before forwarding.
    url = httpx.URL(path=request.url.path, query=request.url.query.decode())
    upstream = httpx.URL(f"{target}{url.path}?{url.query.decode()}")
    body = await request.body()
    r = await httpx.AsyncClient().request(
        request.method, str(upstream),
        headers={k: v for k, v in request.headers.raw if k != b"host"},
        content=body, timeout=None,
    )
    return Response(r.content, status_code=r.status_code, headers=dict(r.headers))

app = Starlette(routes=[Route("/{path:path}", proxy), Route("/", proxy)])
```

Run with `uvicorn profile_router:app --host 127.0.0.1 --port 8642`.

---

## 8. Cloudflare tunnel

### Quick tunnel (good for trying it out)

```bash
# deploy/cloudflared-quick.sh
#!/usr/bin/env bash
set -euo pipefail
exec /home/octa/.local/bin/cloudflared tunnel --url http://127.0.0.1:8642
```

URL rotates on every run. Fine for development, painful for a home-screen PWA.

### Named tunnel (recommended for daily use)

```bash
# one-time
cloudflared tunnel login
cloudflared tunnel create hermes-voice
cloudflared tunnel route dns hermes-voice voice.tabodat.dev   # pick a hostname you own

# then run
cloudflared tunnel run hermes-voice
```

Stable URL, no auth prompts, survives restarts. Drop a systemd unit in
`deploy/systemd/hermes-voice.service` to launch on boot.

---

## 9. Security checklist (do not skip)

- [ ] **Auth token on every `/api/*` call.** The tunnel makes the server public; the token is the only thing between the world and your agent's tool loop.
- [ ] **HTTPS only.** `getUserMedia` / `SpeechRecognition` require a secure origin. The Cloudflare URL is HTTPS; `http://localhost` from the phone will not work.
- [ ] **No loopback exposure.** API server binds to `127.0.0.1`, only the tunnel reaches it.
- [ ] **Rate limit the chat endpoint.** One short message per second per session is plenty for a single user; reject anything bursty.
- [ ] **Token rotation.** A button in the PWA to rotate the token; the old one is invalidated server-side.
- [ ] **Audit log.** Append every `/api/sessions/.../chat` call (text only, no audio) to a local log for later review.
- [ ] **No STT/TTS logging at the vendor.** Edge TTS already does not log; if you swap to a paid TTS provider, turn off their training-data opt-in.

Full version: `docs/security.md`.

---

## 10. Latency budget (v0.1)

| Stage | Where | Time |
|---|---|---|
| Mic → final transcript | browser STT | ~stream (depends on utterance length) |
| Transcript → first token | SSE from Hermes | 300–600 ms (minimax-m3 TTFB) |
| First token → first audio | Edge TTS, per-sentence | 300–500 ms |
| **Time to first voice** | | **~0.8–1.5 s** |
| Subsequent sentences | overlap TTS fetch with token stream | 200–400 ms per sentence |
| Barge-in (cancel) | client-side `audio.pause()` + SSE abort | <50 ms perceived |

Where to shave it later (Phase 2):
- Local TTS (Kokoro-82M) drops first-audio to 80–150 ms.
- Server-side VAD + streaming STT drops transcript latency to ~150–300 ms.
- End-to-end target with both: ~0.5–0.9 s.

Full version: `docs/latency-budget.md`.

---

## 11. Phased build

### Phase 1 — MVP (≈1 day)
1. Apply config snippet to `profiles/coder/config.yaml`. Enable `api_server`.
2. Add the four endpoints to `gateway/platforms/api_server.py`.
3. Drop `client/` next to the Hermes checkout (or symlink).
4. `uvicorn` (or the existing runner) starts the API server on `:8642`.
5. `cloudflared tunnel --url http://127.0.0.1:8642` for first try.
6. Open the printed URL on the phone, install the PWA, talk.
7. **Verify:** utterance → first voice under 1.5 s; barge-in works.

### Phase 1.5 — Per-profile (≈½ day)
1. Apply config to scout/vector/banel; pick distinct ports.
2. Stand up `server/profile_router.py` on `:8642`.
3. Add the `?profile=` selector to the PWA header.

### Phase 2 — WebSocket streaming pipeline (≈1–2 wks, optional)
See `docs/phase-2-websocket.md`. Server-side Silero VAD for proper
endpointing and barge-in; Kokoro-82M local TTS; AudioWorklet → WebSocket
on the client. Streaming replaces the SSE-then-TTS model.

### Phase 3 — Polish (∞)
Wake word ("Hey Axi"), tool-call streaming ("running `gh repo list`…"),
multi-language STT, transcript export, dark/light theme, iOS Safari
support, accessibility audit, named tunnel + stable hostname.

---

## 12. Open questions

- **Per-profile ports vs. one router port?** v0.1 ships the router; profile-direct stays as a fallback.
- **Cancel mid-tool-call?** Acknowledged as best-effort. Decide whether to surface a "stopping…" indicator while the current tool completes.
- **Audio format for TTS?** Edge TTS gives us `audio/ogg` (`opus`). All Android Chrome versions play it. If iOS is ever needed, switch the TTS bridge to `audio/mpeg`.
- **Session persistence?** v0.1 generates a session id per tab and forgets on reload. Add `localStorage` later.

---

## 13. License

MIT — same as the rest of the Axi/Hermes stack.
