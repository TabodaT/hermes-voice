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

> **STT caveat (v0.1):** browser `webkitSpeechRecognition` on Android Chrome
> typically streams microphone audio to Google's servers for transcription — it
> is **not** self-hosted and you cannot force on-device recognition. So
> "self-hosted" and "no vendor sees your audio" hold for the *brain and TTS* in
> v0.1, but **not** for STT. Fully self-hosted STT arrives with the Phase-2
> faster-whisper path.

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
│  <audio> queue         │                                  │     ├─ GET  /api/tts?text=…      │
│   └─ per-sentence TTS  │                                  │     │   (Edge TTS, audio/ogg)    │
│                        │                                  │     ├─ POST /api/…/cancel        │
│  service worker (PWA)  │                                  │     └─ GET  /voice  (static PWA) │
└────────────────────────┘                                  │                                  │
                                                             │   AIAgent.run_conversation()     │
                                                             │     (minimax-m3, tool loop)      │
                                                             │                                  │
                                                             │   Other profiles (scout/vector/  │
                                                             │   banel) ── separate ports      │
                                                             └──────────────────────────────────┘
```

**Barge-in path (v0.1 is half-duplex — see the echo note):**
1. **Push-to-talk:** the mic is only live while the button is held, so *pressing
   PTT is itself the interrupt*. On `pointerdown` the client aborts any in-flight
   SSE, pauses and clears the audio queue, and POSTs `/api/sessions/.../cancel`.
2. **Hands-free:** the recognizer is paused while the assistant is speaking and
   resumes when the audio queue drains. This avoids the echo loop below; the cost
   is that you cannot barge in *by voice* mid-sentence in hands-free — tap the
   screen (or use PTT) to interrupt.
3. On interrupt the client calls `audio.pause()` + `AbortController.abort()` on
   the SSE fetch. (We play server TTS through an `<audio>` element, **not** the
   browser `speechSynthesis` API, so there is no `speechSynthesis.cancel()`.)
4. Client POSTs `/api/sessions/.../cancel` to drop the in-flight generation
   server-side, then awaits it before opening the next chat (see the cancel-race
   fix in §6.4).
5. UI shows "interrupted" briefly; the assistant re-engages on the next utterance.

> **Echo / self-barge-in (the hard one):** the Web Speech API exposes no
> acoustic echo cancellation, so an open mic on a phone in speaker mode hears the
> assistant's own TTS and would interrupt itself. v0.1 sidesteps this with
> half-duplex (mic off while speaking). True full-duplex voice barge-in needs
> server-side VAD + AEC and is a Phase-2 item.

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

// Token: the PWA is served from the open /voice path, but every /api/* call
// (chat, tts, cancel) needs the shared token. Store it once in localStorage.
// The <audio> element can't set headers, so /api/tts gets it as a ?token= param.
let TOKEN = localStorage.getItem('axiToken') || '';
function ensureToken() {
  if (!TOKEN) {
    TOKEN = (prompt('Hermes voice token') || '').trim();
    if (TOKEN) localStorage.setItem('axiToken', TOKEN);
  }
  return TOKEN;
}

// ─── DOM ────────────────────────────────────────────────────────────────────
const $ = (id) => document.getElementById(id);
const transcript = $('transcript');
const ptt = $('ptt');
const statusEl = $('status');
const profileSel = $('profile');
const handsfree = $('handsfree');

profileSel.onchange = () => (profile = profileSel.value);
function setStatus(s) { statusEl.textContent = s; }

// ─── transcript helpers ─────────────────────────────────────────────────────
function addLine(role, text) {
  const el = document.createElement('div');
  el.className = `line ${role}`;
  el.textContent = text;
  transcript.appendChild(el);
  transcript.scrollTop = transcript.scrollHeight;
  return el;
}

// ─── speakable text (strip markdown/code so TTS doesn't read symbols) ────────
// The coder profile emits code fences, backticks, JSON and URLs; reading those
// aloud verbatim is unusable. Flatten to something a voice can actually speak.
function speakable(text) {
  const FENCE = '`{3}';   // matches a code fence; built via string, not a literal, so it can't close this block
  return text
    .replace(new RegExp(FENCE + '[\\s\\S]*?' + FENCE, 'g'), ' (code block) ')  // fenced code
    .replace(/`([^`]+)`/g, '$1')                      // inline code
    .replace(/!?\[([^\]]*)\]\([^)]*\)/g, '$1')        // links/images -> label
    .replace(/https?:\/\/\S+/g, ' link ')             // bare URLs
    .replace(/[*_#>~|]+/g, ' ')                        // markdown markup
    .replace(/\s+/g, ' ')
    .trim();
}

// ─── half-duplex flag: the mic is muted while the assistant is speaking ──────
let assistantSpeaking = false;   // audio is (or is about to be) playing
let responseStreaming = false;   // tokens are still arriving for this turn

// ─── STT (browser, streaming via webkitSpeechRecognition) ────────────────────
const SR = window.SpeechRecognition || window.webkitSpeechRecognition;
let recognizer = null;
let recognizerActive = false;

function startRecognizer() {
  if (!SR) { setStatus('STT unsupported in this browser'); return; }
  if (recognizerActive || assistantSpeaking) return;  // guard double-start / echo
  const r = new SR();
  recognizer = r;
  r.continuous = true;
  r.interimResults = true;
  r.lang = 'en-US';

  let interim = '';
  let finalBuf = '';

  r.onstart = () => { recognizerActive = true; };
  r.onresult = (ev) => {
    if (assistantSpeaking) return;          // ignore echo of our own TTS
    interim = '';
    for (let i = ev.resultIndex; i < ev.results.length; i++) {
      const res = ev.results[i];
      if (res.isFinal) finalBuf += res[0].transcript;
      else interim += res[0].transcript;
    }
    if (interim) setStatus(`hearing: ${interim.trim().slice(0, 60)}…`);
    if (finalBuf) {
      const text = finalBuf.trim();
      finalBuf = '';
      if (text) sendUtterance(text);
    }
  };
  r.onerror = (e) => { console.warn('SR error', e.error || e); };
  r.onend = () => {
    recognizerActive = false;
    // Auto-restart only in hands-free, and only while we're not speaking.
    if (handsfree.checked && !assistantSpeaking) {
      try { r.start(); } catch { /* retry on next onend */ }
    }
  };

  try { r.start(); } catch (e) { console.warn('SR start failed', e); }
}

function stopRecognizer() {
  recognizerActive = false;
  try { recognizer?.stop(); } catch { /* ignore */ }
  recognizer = null;
}

// ─── LLM streaming + per-sentence TTS playback ───────────────────────────────
let inflightAbort = null;
const audioQueue = [];     // holds *preloaded* HTMLAudioElement objects (prefetch)
let audioEl = null;

// Cancel the PREVIOUS response: abort the local SSE read first, then tell the
// server to drop generation, and await it so a stale cancel can't race the new
// chat (the chat endpoint also clears its own flag at start — see §6.4).
async function cancelInflight() {
  const had = !!inflightAbort || !!audioEl || audioQueue.length > 0;
  if (inflightAbort) { inflightAbort.abort(); inflightAbort = null; }
  for (const a of audioQueue) { try { a.pause(); } catch {} }
  audioQueue.length = 0;
  if (audioEl) { try { audioEl.pause(); } catch {} audioEl = null; }
  assistantSpeaking = false;
  if (!had) return;
  try {
    await fetch(`${API_BASE}/api/sessions/${sessionId}/cancel?profile=${profile}`,
      { method: 'POST', headers: { 'x-axi-token': ensureToken() } });
  } catch { /* best-effort */ }
}

async function sendUtterance(text) {
  await cancelInflight();                  // barge-in: drop the previous turn first

  addLine('user', text);
  const assistantEl = addLine('assistant', '');

  inflightAbort = new AbortController();
  responseStreaming = true;
  let acc = '';
  let sentenceBuf = '';

  try {
    const r = await fetch(`${API_BASE}/api/sessions/${sessionId}/chat?profile=${profile}`, {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'Accept': 'text/event-stream',
        'x-axi-token': ensureToken(),
      },
      body: JSON.stringify({ message: text, stream: true }),
      signal: inflightAbort.signal,
    });
    if (!r.ok || !r.body) throw new Error(`HTTP ${r.status}`);

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
          // Sentence boundary: punctuation + space + a capital/quote/digit, and a
          // min length, so "v0.1 ", "e.g. " and "1.5 s" don't split mid-thought.
          const m = sentenceBuf.match(/^[\s\S]*?[.!?…]["')\]]?\s+(?=[A-Z0-9"'(])/);
          if (m && m[0].trim().length > 12) {
            const spoken = speakable(m[0]);
            sentenceBuf = sentenceBuf.slice(m[0].length);
            if (spoken) enqueueTts(spoken);
          }
        } catch { /* ignore non-JSON frames */ }
      }
    }
    const tail = speakable(sentenceBuf);
    if (tail) enqueueTts(tail);
  } catch (e) {
    if (e.name !== 'AbortError') {
      console.error(e);
      setStatus(`error: ${e.message}`);
    }
  } finally {
    inflightAbort = null;
    responseStreaming = false;
    if (!audioEl && !audioQueue.length && handsfree.checked) startRecognizer();
  }
}

// Prefetch: build the Audio element (which starts buffering immediately) the
// moment a sentence is ready, so clip N+1 downloads while clip N is playing.
function enqueueTts(text) {
  const url = `${API_BASE}/api/tts?text=${encodeURIComponent(text)}`
            + `&profile=${profile}&token=${encodeURIComponent(ensureToken())}`;
  const a = new Audio();
  a.preload = 'auto';
  a.src = url;
  audioQueue.push(a);
  if (!audioEl) playNext();
}

function playNext() {
  const a = audioQueue.shift();
  if (!a) {
    audioEl = null;
    assistantSpeaking = false;
    // Resume listening only once the turn is fully done (queue drained + stream
    // closed), so an inter-sentence gap doesn't reopen the mic mid-response.
    if (handsfree.checked && !responseStreaming) startRecognizer();
    return;
  }
  audioEl = a;
  assistantSpeaking = true;
  stopRecognizer();           // half-duplex: don't listen to ourselves
  a.onended = playNext;
  a.onerror = playNext;
  a.play().catch(playNext);
}

// ─── PTT button + hands-free toggle ──────────────────────────────────────────
// Pressing PTT is itself the interrupt: cancel anything in flight, then listen.
ptt.addEventListener('pointerdown', () => { cancelInflight(); startRecognizer(); });
ptt.addEventListener('pointerup',   () => stopRecognizer());
ptt.addEventListener('pointerleave', () => stopRecognizer());
handsfree.onchange = () => { if (handsfree.checked) startRecognizer(); else stopRecognizer(); };

// First user gesture: prime the token (and the mic, if hands-free).
// getUserMedia / SpeechRecognition both require a user gesture + secure origin.
document.addEventListener('pointerdown', function once() {
  document.removeEventListener('pointerdown', once);
  ensureToken();
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
const CACHE = 'axi-v2';
const SHELL = ['/voice/', '/voice/app.js', '/voice/style.css', '/voice/manifest.webmanifest'];
self.addEventListener('install', (e) => e.waitUntil(caches.open(CACHE).then(c => c.addAll(SHELL))));
self.addEventListener('activate', (e) => e.waitUntil(self.clients.claim()));
self.addEventListener('fetch', (e) => {
  const url = new URL(e.request.url);
  // Cache only the static app shell. NEVER cache /api/* — chat is a one-shot SSE
  // stream and /api/tts URLs are unique per sentence (the cache would grow without
  // bound and could replay stale audio).
  if (e.request.method !== 'GET' || url.pathname.startsWith('/api/')) return;
  e.respondWith(
    caches.match(e.request).then(hit => hit ||
      fetch(e.request).then(res => {
        if (url.origin === location.origin && res.ok) {
          const copy = res.clone();
          caches.open(CACHE).then(c => c.put(e.request, copy)).catch(() => {});
        }
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
  port: 8642          # per-profile: coder 8642, scout 8643, vector 8644, banel 8645
                      # the phase-1.5 profile router (§7) runs SEPARATELY on 8640
                      # and fronts all of these — never reuse a profile's port for it
  voice_path: /voice  # serves the PWA static files
  require_token: true
  access_token: ${HERMES_VOICE_TOKEN}   # a long random string in .env
```

### 6.2 Endpoints added to `gateway/platforms/api_server.py`

```python
# ─── additions only; merge into the existing FastAPI app ──────────────────
import os
from fastapi import APIRouter, HTTPException, Request, Query
from fastapi.responses import FileResponse, Response

router = APIRouter()
TOKEN = os.environ["HERMES_VOICE_TOKEN"]

# Static client dir, resolved ONCE and reused as the traversal jail below.
CLIENT_ROOT = os.path.abspath(
    os.path.join(os.path.dirname(__file__), "..", "..", "..", "hermes-voice", "client")
)

# 1) Auth gate. Everything under /api/* needs the token (header or ?token=).
#    /voice (the PWA shell) and /health stay open. NOTE: /api/tts lives INSIDE
#    the gate on purpose — it spawns synthesis and must not be a free, public,
#    abusable endpoint once the tunnel makes the box reachable.
@router.middleware("http")
async def _auth(request: Request, call_next):
    if not request.url.path.startswith("/api/"):
        return await call_next(request)
    if request.headers.get("x-axi-token") != TOKEN and \
       request.query_params.get("token") != TOKEN:
        raise HTTPException(status_code=401, detail="bad token")
    return await call_next(request)

# 2) Serve the PWA at /voice (open — it's just the static shell)
@router.get("/voice")
@router.get("/voice/{path:path}")
async def voice(path: str = ""):
    full = os.path.abspath(os.path.join(CLIENT_ROOT, path or "index.html"))
    # Reject traversal: the resolved path must stay inside CLIENT_ROOT. Compare
    # absolute-vs-absolute via commonpath (the old startswith(abspath) check
    # mixed a relative `full` with an absolute root and silently 404'd).
    if os.path.commonpath([full, CLIENT_ROOT]) != CLIENT_ROOT:
        raise HTTPException(404)
    if os.path.isdir(full):
        full = os.path.join(full, "index.html")
    if not os.path.isfile(full):
        raise HTTPException(404)
    media = "text/html" if full.endswith(".html") else None
    return FileResponse(full, media_type=media)

# 3) TTS — UNDER /api/ so the auth gate covers it. The <audio> element can't set
#    headers, so the client passes the token as ?token=. Rate-limit per session
#    (see §9). synthesize() is async (edge-tts streams), so await it directly —
#    do NOT wrap an async fn in asyncio.to_thread (that returns an un-awaited
#    coroutine, not bytes).
@router.get("/api/tts")
async def tts(text: str = Query(..., max_length=2000), profile: str = "coder"):
    from hermes_voice.tts_bridge import synthesize  # tiny shim, see 6.3
    audio = await synthesize(text, profile)
    return Response(content=audio, media_type="audio/ogg")

# 4) Cancel an in-flight response (best-effort; the agent loop notices the token)
@router.post("/api/sessions/{sid}/cancel")
async def cancel(sid: str, profile: str = "coder"):
    from hermes_voice.cancel_registry import request_cancel
    request_cancel(profile, sid)
    return {"ok": True}

# 5) Health (open)
@router.get("/health")
async def health():
    return {"ok": True}
```

### 6.3 Tiny TTS shim — `hermes-voice/server/tts_bridge.py`

```python
"""Bridge to the existing Hermes TTS path. Replace the import with the real
text-to-speech tool used by the Telegram gateway (edge-tts, ElevenLabs, …).
Kept async because edge-tts streams; the /api/tts endpoint awaits it directly.
"""
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

Cancel-race contract (so a stale flag never kills a fresh turn):
- The **chat** endpoint calls `clear(profile, session_id)` at the **start** of a
  new generation — before the first token — so a leftover cancel flag from the
  previous turn can't abort the new response.
- The agent loop calls `is_cancelled()` at the top of each tool iteration and
  `clear()` again when the loop exits (any branch).
- The **client** awaits the cancel POST before opening the next chat (see
  `cancelInflight()` in §5), so the cancel and the new request can't reorder.

---

## 7. Per-profile routing

Each Hermes profile has its own gateway + API server. v0.1 ships a **profile router**
that fronts them all on a single port and dispatches by `?profile=` query param
or `X-Hermes-Profile` header. The router is ~80 lines of Python and runs as a
standalone process; cloudflared points at the router's port.

The router gets its **own** port (8640). It must NOT reuse a profile's port — a
process can't bind a port another process already holds, and forwarding to your
own address loops. coder/scout/vector/banel keep 8642–8645 (matching §6.1).

```
phone ──► cloudflared ──► :8640  (profile router)
                            │
                            ├─ coder  :8642  (real api_server.py)
                            ├─ scout  :8643
                            ├─ vector :8644
                            └─ banel  :8645
```

`server/profile_router.py` (sketch — a thin Starlette + httpx streaming proxy):

```python
from contextlib import asynccontextmanager
from starlette.applications import Starlette
from starlette.requests import Request
from starlette.responses import StreamingResponse, JSONResponse
from starlette.routing import Route
from starlette.background import BackgroundTask
import httpx

PROFILES = {
    "coder":  "http://127.0.0.1:8642",
    "scout":  "http://127.0.0.1:8643",
    "vector": "http://127.0.0.1:8644",
    "banel":  "http://127.0.0.1:8645",
}

# Hop-by-hop headers must not be forwarded (RFC 7230 §6.1) or they corrupt the
# proxied response (notably content-length / transfer-encoding on a stream).
HOP = {b"connection", b"keep-alive", b"transfer-encoding", b"te", b"trailer",
       b"upgrade", b"proxy-authorization", b"proxy-authenticate", b"host"}

client = httpx.AsyncClient(timeout=None)   # one reused client, closed on shutdown

@asynccontextmanager
async def lifespan(app):
    yield
    await client.aclose()

async def proxy(request: Request):
    profile = request.query_params.get("profile") or request.headers.get("x-hermes-profile", "coder")
    if profile not in PROFILES:
        return JSONResponse({"error": "unknown profile"}, status_code=400)
    upstream = f"{PROFILES[profile]}{request.url.path}"
    headers = [(k, v) for k, v in request.headers.raw if k.lower() not in HOP]
    body = await request.body()
    # STREAM the upstream response — never `await r.content` / buffer it, or the
    # SSE token stream gets collected in full before the phone sees a single token
    # and "time to first voice" balloons to the length of the whole answer.
    req = client.build_request(request.method, upstream,
                               params=request.query_params, headers=headers, content=body)
    r = await client.send(req, stream=True)
    resp_headers = {k.decode(): v.decode() for k, v in r.headers.raw if k.lower() not in HOP}
    return StreamingResponse(r.aiter_raw(), status_code=r.status_code,
                             headers=resp_headers, background=BackgroundTask(r.aclose))

app = Starlette(lifespan=lifespan,
                routes=[Route("/{path:path}", proxy, methods=["GET", "POST"]),
                        Route("/", proxy, methods=["GET", "POST"])])
```

Run with `uvicorn profile_router:app --host 127.0.0.1 --port 8640`.

---

## 8. Cloudflare tunnel

### Quick tunnel (good for trying it out)

```bash
# deploy/cloudflared-quick.sh
#!/usr/bin/env bash
set -euo pipefail
# Point at the profile router (§7). For a Phase-1 single-profile trial with no
# router yet, use the coder api_server directly: http://127.0.0.1:8642
exec /home/octa/.local/bin/cloudflared tunnel --url http://127.0.0.1:8640
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

- [ ] **Auth token on every `/api/*` call — including `/api/tts`.** The tunnel makes the server public; the token is the only thing between the world and your agent's tool loop. `/api/tts` is deliberately inside the gate (it spawns synthesis); only `/voice` (static shell) and `/health` are open.
- [ ] **HTTPS only.** `getUserMedia` / `SpeechRecognition` require a secure origin. The Cloudflare URL is HTTPS; `http://localhost` from the phone will not work.
- [ ] **No loopback exposure.** API server (and the router) bind to `127.0.0.1`; only the tunnel reaches them.
- [ ] **Rate limit the chat *and* TTS endpoints.** One short message per second per session is plenty for a single user; reject anything bursty. `/api/tts` especially — it's a free synthesis box otherwise.
- [ ] **Token rotation.** A button in the PWA to rotate the token; the old one is invalidated server-side (and the client's `localStorage` copy is replaced).
- [ ] **Audit log.** Append every `/api/sessions/.../chat` call (text only, no audio) to a local log for later review.
- [ ] **Browser STT sends audio to Google.** In v0.1 `webkitSpeechRecognition` transcribes via Google's servers — your spoken audio leaves the box even though the brain and TTS don't. If that's unacceptable, wait for the Phase-2 faster-whisper path (fully local STT).
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
| Subsequent sentences | client prefetches clip N+1 while clip N plays (§5 `enqueueTts`) | 200–400 ms per sentence |
| Barge-in (cancel) | client-side `audio.pause()` + SSE abort | <50 ms perceived |

> The prefetch matters: without it (`new Audio(url)` only on the *previous*
> clip's `onended`) each sentence pays the full ~300–500 ms synth latency
> serially, leaving an audible gap between sentences. The §5 reference code
> builds the next `Audio` element as soon as the sentence is ready so the fetch
> overlaps playback.

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
6. Open the printed URL on the phone, paste the token once when prompted, install the PWA, talk.
7. **Verify:** utterance → first voice under 1.5 s; PTT interrupt works (hands-free is half-duplex — mic pauses while the assistant speaks).

### Phase 1.5 — Per-profile (≈½ day)
1. Apply config to scout/vector/banel on distinct ports (8643/8644/8645).
2. Stand up `server/profile_router.py` on `:8640` (its own port — not a
   profile's), and repoint cloudflared at `:8640`.
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

- **Per-profile ports vs. one router port?** v0.1 ships the router on its own port (8640) in front of profiles on 8642–8645; profile-direct stays as a fallback.
- **Full-duplex voice barge-in?** Deferred to Phase 2. v0.1 is half-duplex (mic off while speaking) because the Web Speech API has no echo cancellation; true voice interruption needs server-side VAD + AEC.
- **Cancel mid-tool-call?** Acknowledged as best-effort. Decide whether to surface a "stopping…" indicator while the current tool completes.
- **Audio format for TTS?** Edge TTS gives us `audio/ogg` (`opus`). All Android Chrome versions play it. If iOS is ever needed, switch the TTS bridge to `audio/mpeg`.
- **Token onboarding/rotation?** v0.1 prompts once and caches the token in `localStorage`. Rotation must invalidate it server-side and force a re-prompt on the client.
- **Session persistence?** v0.1 generates a session id per tab and forgets on reload. Add `localStorage` later.

---

## 13. License

MIT — same as the rest of the Axi/Hermes stack.
