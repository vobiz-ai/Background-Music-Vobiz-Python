# Background Music on a Live Call — Vobiz + Python

Play looping background music **mixed in parallel with the live conversation** during a phone call, using the [Vobiz](https://vobiz.ai) telephony platform. No conference room required — works directly on any active call via the Vobiz Live Call Play API.

---

## How It Works

```
Caller dials in (or receives an outbound call)
         │
         ▼
POST /answer  ←  Vobiz webhook
         │
         ├─ Returns XML: <Speak> greeting + <Wait> (holds the line)
         │
         └─ Fires Live Call Play API in background thread (2s delay):
              POST /Account/{auth_id}/Call/{call_uuid}/Play/
              {
                "urls": ["https://your-music.mp3"],
                "loop": true,    ← Vobiz loops natively
                "mix":  true,    ← audio MIXED alongside live speech
                "legs": "both"   ← both parties hear the music
              }
                    │
                    ▼
         Caller hears: voice + background music simultaneously
```

**Key insight:** The `mix: true` parameter tells Vobiz to blend the music into the call audio alongside the live voice — not replace it. `loop: true` means Vobiz handles the repeat internally — no polling or re-trigger logic needed on your server.

---

## Features

- Background music plays **in parallel** with live speech — true audio mixing
- **Loops automatically** — set it and forget it, Vobiz handles the repeat
- **Both legs** hear the music (caller + agent/bot)
- **No conference room needed** — works on any standard call
- Control API to start/stop/change music mid-call
- Live status endpoint showing active calls and music state

---

## Project Structure

```
.
├── server.py            # FastAPI server — webhooks + control API
├── background_store.py  # (legacy) Conference-based state store — kept for reference
├── requirements.txt     # Python dependencies
├── .env.example         # Configuration template
└── README.md
```

---

## Quick Start

### 1. Clone & install

```bash
git clone https://github.com/vobiz-ai/Background-Music-Vobiz-Python.git
cd Background-Music-Vobiz-Python
pip install -r requirements.txt
```

### 2. Configure

```bash
cp .env.example .env
```

Edit `.env`:

```env
VOBIZ_AUTH_ID=MA_XXXXXXXXX
VOBIZ_AUTH_TOKEN=XXXXXXXXXXXXXXXXXXXXXXXX
FROM_NUMBER=+91XXXXXXXXXX
BACKGROUND_AUDIO_URL=https://www.soundhelix.com/examples/mp3/SoundHelix-Song-1.mp3
PUBLIC_URL=https://your-ngrok-or-server-url.com
```

### 3. Start the server

```bash
python server.py
```

Output:
```
  09 — Background Audio (Live Call Play API)
  Answer URL     : https://your-url.ngrok-free.app/answer
  Hangup URL     : https://your-url.ngrok-free.app/hangup
  Trigger music  : POST https://your-url.ngrok-free.app/trigger-background
  Stop music     : POST https://your-url.ngrok-free.app/stop-background
  Status         : GET  https://your-url.ngrok-free.app/status
  Audio URL      : https://www.soundhelix.com/examples/mp3/SoundHelix-Song-1.mp3
  Mode           : loop=true  mix=true  legs=both
```

### 4. Configure your Vobiz number

In the [Vobiz Dashboard](https://app.vobiz.ai):
- Set **Answer URL** → `POST https://your-url/answer`
- Set **Hangup URL** → `POST https://your-url/hangup`

### 5. Make a call

Either call your Vobiz number inbound, or trigger an outbound call via the API:

```bash
curl -X POST "https://api.vobiz.ai/api/v1/Account/{AUTH_ID}/Call/" \
  -H "X-Auth-ID: YOUR_AUTH_ID" \
  -H "X-Auth-Token: YOUR_AUTH_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "from": "+91XXXXXXXXXX",
    "to": "+91XXXXXXXXXX",
    "answer_url": "https://your-url/answer",
    "answer_method": "POST",
    "hangup_url": "https://your-url/hangup",
    "hangup_method": "POST"
  }'
```

When the call is answered, you'll hear the greeting voice message followed by background music playing underneath — mixed in parallel with any conversation.

---

## API Endpoints

| Method | Path | Description |
|--------|------|-------------|
| POST | `/answer` | Vobiz webhook — called when call is answered. Returns XML + triggers music. |
| POST | `/hangup` | Vobiz webhook — called when call ends. Cleans up state. |
| POST | `/trigger-background` | Start or restart music on a live call |
| POST | `/stop-background` | Stop music on a live call (call stays active) |
| GET  | `/status` | Active calls and their music state |
| GET  | `/health` | Health check + current config |

---

## Control API Usage

### Start/restart music on a live call

```bash
curl -X POST https://your-url/trigger-background \
  -H "Content-Type: application/json" \
  -d '{
    "call_uuid": "CALL_UUID_HERE",
    "url": "https://example.com/your-music.mp3"
  }'
```

Response:
```json
{
  "status": "started",
  "call_uuid": "abc-123",
  "audio_url": "https://example.com/your-music.mp3",
  "vobiz_response": { "message": "play started", "api_id": "..." }
}
```

### Stop music (call stays active)

```bash
curl -X POST https://your-url/stop-background \
  -H "Content-Type: application/json" \
  -d '{ "call_uuid": "CALL_UUID_HERE" }'
```

### Check active calls and music state

```bash
curl https://your-url/status
```

Response:
```json
{
  "active_call_count": 1,
  "active_calls": {
    "51d473ff-4522-44fd-87d9-fce46c827c21": {
      "from": "918065481638",
      "music_playing": true
    }
  },
  "default_audio_url": "https://www.soundhelix.com/examples/mp3/SoundHelix-Song-1.mp3"
}
```

---

## Environment Variables

| Variable | Required | Default | Description |
|----------|----------|---------|-------------|
| `VOBIZ_AUTH_ID` | Yes | — | Your Vobiz account auth ID |
| `VOBIZ_AUTH_TOKEN` | Yes | — | Your Vobiz account auth token |
| `FROM_NUMBER` | Yes | — | Your Vobiz DID number |
| `BACKGROUND_AUDIO_URL` | No | SoundHelix MP3 | HTTPS URL to an MP3 or WAV file |
| `HTTP_PORT` | No | `8000` | Server port |
| `PUBLIC_URL` | No | — | Production/ngrok URL (skips starting ngrok from code) |
| `NGROK_AUTH_TOKEN` | No | — | ngrok auth token (for local dev without PUBLIC_URL) |

---

## Audio File Tips

- Use **MP3 or WAV** — both are supported by Vobiz
- Host on any **public HTTPS URL** (S3, CDN, GitHub releases, etc.)
- For telephony quality, **8kHz or 16kHz mono** files sound best on phone calls
- The `loop: true` flag means the file repeats seamlessly — use a clean-looping track for best results
- A 30–120 second clip works well; shorter loops may sound repetitive

---

## Vobiz API Reference

- [Make a Call](https://vobiz.ai/docs/call/make-call)
- [Play Audio on a Call](https://vobiz.ai/docs/call/play-audio) — `POST /Call/{uuid}/Play/`
- [Stop Audio on a Call](https://vobiz.ai/docs/call/play-audio/stop-audio) — `DELETE /Call/{uuid}/Play/`
- [Conference Member Play](https://vobiz.ai/docs/conference/members/play-audio) — alternative conference-based approach

---

## Local Development with ngrok

If you don't have a public URL, use [ngrok](https://ngrok.com):

```bash
ngrok http 8000
```

Copy the `https://....ngrok-free.app` URL into your `.env` as `PUBLIC_URL`, or set `NGROK_AUTH_TOKEN` and let the server start ngrok automatically.

---

## License

MIT
