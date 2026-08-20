# Background Music on a Live Call — Vobiz + Python

Play looping background music mixed in parallel with the live conversation on any active call, using the [Vobiz](https://vobiz.ai) Live Call Play API — no conference room required.

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](./LICENSE)
[![Python](https://img.shields.io/badge/Python-3.9%2B-3776AB.svg)](https://www.python.org/)
[![Docs](https://img.shields.io/badge/Docs-docs.vobiz.ai-0A66C2.svg)](https://docs.vobiz.ai)

## Overview

Playing audio on a call is easy. Playing audio *underneath* a call — so the caller
hears music and the live voice at the same time — is the part that usually turns
into an architecture problem. The common workaround is to build a conference room,
put the caller and the agent in it, and join a third media source to play the
track. That works, but it means every call now carries conference semantics you
did not otherwise need.

This example takes the direct route. Vobiz exposes a Play API on an individual
live call, and that API accepts a `mix` flag. With `mix: true` the audio is blended
into the call's media alongside whatever is already being spoken, rather than
replacing it. Combined with `loop: true`, Vobiz repeats the file internally for the
lifetime of the playback, so your server does not poll, re-trigger, or run a timer
to keep the music going. One API call per call, and the platform handles the rest.

The repository is a complete FastAPI service demonstrating that pattern. It hosts
the two Vobiz webhooks (`/answer`, `/hangup`), and it exposes a small control API
so you can start, swap, or stop the music on any call that is currently up —
useful for hold music you want to kill the moment an agent picks up.

What you get at the end is a working reference for background audio you can lift
straight into an AI voice agent, an IVR, or a hold-queue: the same three
parameters (`loop`, `mix`, `legs`) apply wherever a call UUID exists.

## What you can build with it

- **Hold music that stops on answer** — start the loop when the caller enters the
  queue, `DELETE` it the instant an agent joins.
- **Ambience behind an AI voice agent** — a low bed of music or room tone makes a
  synthetic voice feel less clinical, without touching the agent's audio pipeline.
- **Branded audio identity** — play a signature track under the greeting on every
  inbound call to your main line.
- **Compliance and notice loops** — repeat a recorded notice underneath a call for
  its full duration, rather than only at the start.
- **Waiting-room atmosphere** — background sound during long-running verification
  or processing steps, so silence never reads as a dropped call.
- **Live A/B testing of audio beds** — swap the track mid-call through
  `/trigger-background` and compare caller reaction.

## How it works

```
Caller dials in (or receives an outbound call)
         │
         ▼
POST /answer  ←  Vobiz webhook
         │
         ├─ Returns XML: <Speak> greeting + <Wait length="120"> (holds the line)
         │
         └─ Fires Live Call Play API in a background thread (2s delay):
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

POST /hangup  ←  Vobiz webhook  →  call removed from active_calls
```

**Key insight:** the `mix: true` parameter tells Vobiz to blend the music into the
call audio alongside the live voice — not replace it. `loop: true` means Vobiz
handles the repeat internally, so there is no polling or re-trigger logic on your
server.

### Why the playback is deferred

`/answer` does not call the Play API inline. It hands the work to
`start_music_after_answer()`, which sleeps in a daemon thread — 2.0 seconds as
called from `/answer` — before firing the request. The delay exists because the
call has only just been answered: issuing Play immediately can race the media
path being fully established, and it lets the `<Speak>` greeting begin cleanly
before the bed fades in. Because the thread is a daemon and the handler returns
straight away, the XML response is never blocked by the API round trip.

The consequence worth knowing: if the Play API fails, `/answer` has already
returned `200`. The failure surfaces only in the log line
`Live Play start failed`, and `music_playing` stays `false` in `/status`.

### Looping behaviour

Looping is entirely server-side at Vobiz. `loop: true` is sent once, and the file
repeats until one of three things happens: something issues
`DELETE /Call/{uuid}/Play/`, a fresh `POST` to the same endpoint replaces the
playback, or the call ends. There is no loop interval to tune and no maximum
repeat count in the payload — which is why the local state is a single
`music_playing` boolean rather than a counter or a timer.

Restarting is simply another `POST`. `/trigger-background` on a call that is
already playing swaps the track rather than layering a second one, so changing the
music mid-call is one request, not a stop-then-start pair.

## Architecture

| File | Responsibility |
|---|---|
| `server.py` | The whole running service. Holds the Vobiz webhooks, the control API, `call_play_start()` / `call_play_stop()`, the deferred-start thread helper, the ngrok bootstrap, and the in-memory `active_calls` dictionary. |
| `background_store.py` | A self-contained implementation of the alternative conference-based approach — `ConferencePlayAPI` wraps the Conference Member Play endpoints and `BackgroundAudioStore` tracks members and audio state. It is **not imported by `server.py`**; it is kept as a reference for anyone who needs conference semantics rather than per-call mixing. |
| `.env.example` | Template for the seven environment variables the server reads. Copy to `.env`. |
| `requirements.txt` | Pinned dependencies: FastAPI, Uvicorn, python-multipart, python-dotenv, pyngrok, requests, pydantic. |
| `.gitignore` | Keeps `.env` and local artefacts out of version control. |

State is a single module-level dictionary, `active_calls`, mapping `CallUUID` to
`{"from", "music_playing"}`. It is populated in `/answer` and removed in
`/hangup`, so it reflects only calls that are currently up, and it does not
survive a restart.

## Prerequisites

- A **Vobiz account** with an auth ID and auth token — [sign up](https://vobiz.ai).
- A **Vobiz phone number (DID)** with its Answer and Hangup URLs pointed at this
  service.
- **Python 3.9 or newer** — `server.py` annotates `active_calls` as
  `dict[str, dict]`, which requires 3.9+.
- **A publicly reachable HTTPS URL**, both so Vobiz can reach your webhooks and
  because the audio file itself must be fetchable by the platform. For local
  development the server can open an [ngrok](https://ngrok.com) tunnel for you.
- **An audio file on a public HTTPS URL.** The default is a SoundHelix sample so
  the example runs with no setup; replace it with your own track.

## Setup

1. **Clone the repository and enter it.**

   ```bash
   git clone https://github.com/vobiz-ai/Background-Music-Vobiz-Python.git
   cd Background-Music-Vobiz-Python
   ```

2. **Create a virtual environment and install dependencies.**

   ```bash
   python3 -m venv .venv
   source .venv/bin/activate      # Windows: .venv\Scripts\activate
   pip install -r requirements.txt
   ```

3. **Create your environment file.**

   ```bash
   cp .env.example .env
   ```

   The server resolves `.env` relative to `server.py`, so this works regardless of
   the directory you launch from.

4. **Fill in `.env`.** Set `VOBIZ_AUTH_ID` and `VOBIZ_AUTH_TOKEN`. Set
   `BACKGROUND_AUDIO_URL` to your own track, or leave the default to hear the
   example working immediately. Leave `PUBLIC_URL` empty to use ngrok.

5. **Point your Vobiz number at the service.** In the
   [Vobiz Dashboard](https://app.vobiz.ai) set the application's **Answer URL** to
   `POST https://your-url/answer` and its **Hangup URL** to
   `POST https://your-url/hangup`, then assign that application to your number.

## Configuration

Every variable the code reads, all via `os.getenv` in `server.py`:

| Variable | Required | Default | Description |
|---|---|---|---|
| `VOBIZ_AUTH_ID` | Yes | *(empty)* | Vobiz account auth ID. Sent as the `X-Auth-ID` header and interpolated into every Play API path. |
| `VOBIZ_AUTH_TOKEN` | Yes | *(empty)* | Vobiz account auth token. Sent as the `X-Auth-Token` header. |
| `BACKGROUND_AUDIO_URL` | No | SoundHelix sample MP3 | Publicly reachable HTTPS URL of the track to mix in. Used on `/answer` and as the fallback when `/trigger-background` omits `url`. |
| `HTTP_PORT` | No | `8000` | Port Uvicorn binds on `0.0.0.0`, and the port ngrok tunnels. |
| `PUBLIC_URL` | No | *(empty)* | Public HTTPS base URL of this service. When set, ngrok is skipped entirely. A trailing slash is stripped automatically. |
| `NGROK_AUTH_TOKEN` | No | *(empty)* | ngrok auth token for local development. Ignored when `PUBLIC_URL` is set. |
| `FROM_NUMBER` | No | *(empty)* | Your Vobiz DID. Read into a module constant and reported for completeness; the inbound flow in `server.py` does not send it. Set it if you extend the example to place outbound calls. |

## Running it

```bash
python server.py
```

Expected output:

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

Call your Vobiz number. You should hear the greeting, then the music fading in
underneath it about two seconds after answer, continuing for the 120-second
`<Wait>`. In the server log, look for:

```
Call answered — CallUUID=... From=...
Live Play started — uuid=... url=... resp={...}
```

You can also drive an outbound call to the same webhook:

```bash
curl -X POST "https://api.vobiz.ai/api/v1/Account/{AUTH_ID}/Call/" \
  -H "X-Auth-ID: YOUR_AUTH_ID" \
  -H "X-Auth-Token: YOUR_AUTH_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "from": "+15550003333",
    "to": "+15550003333",
    "answer_url": "https://your-url/answer",
    "answer_method": "POST",
    "hangup_url": "https://your-url/hangup",
    "hangup_method": "POST"
  }'
```

## API reference

### Endpoints

| Method | Path | Description |
|---|---|---|
| `POST` | `/answer` | Vobiz webhook, called when the call is answered. Registers the call, schedules the deferred Play, and returns the XML. |
| `POST` | `/hangup` | Vobiz webhook, called when the call ends. Removes the call from `active_calls`. |
| `POST` | `/trigger-background` | Starts or replaces music on a live call. Body: `{"call_uuid": "...", "url": "..."}` — `url` optional, falls back to `BACKGROUND_AUDIO_URL`. `400` if the body is not JSON, if `call_uuid` is missing, or if the Play API errors. |
| `POST` | `/stop-background` | Stops music on a live call; the call itself stays up. Body: `{"call_uuid": "..."}`. |
| `GET` | `/status` | Active call count, the `active_calls` map, and the default audio URL. |
| `GET` | `/health` | Health check with the resolved `base_url`, answer URL, audio URL, and mode string. |

### Vobiz Live Call Play API

Both helpers target the same URL,
`https://api.vobiz.ai/api/v1/Account/{auth_id}/Call/{call_uuid}/Play/`, with a
10-second timeout and the `X-Auth-ID` / `X-Auth-Token` headers.

| Method | Purpose | Payload / result |
|---|---|---|
| `POST` | Start or replace playback | `{"urls": [audio_url], "loop": true, "mix": true, "legs": "both"}`. Returns the parsed JSON body, or `{"error": "..."}` on a request failure. |
| `DELETE` | Stop playback | No body. `call_play_stop()` treats `200` or `204` as success. |

Payload fields used here:

| Field | Value | Effect |
|---|---|---|
| `urls` | array with one URL | The audio file(s) Vobiz fetches and plays. |
| `loop` | `true` | Vobiz repeats the file internally until stopped, replaced, or the call ends. |
| `mix` | `true` | Blends the audio into the live call media instead of replacing it. This is what makes it background music rather than a prompt. |
| `legs` | `"both"` | Both parties hear the track. |

### Voice XML elements used

| Element | Attributes used here | Purpose |
|---|---|---|
| `<Speak>` | `voice="WOMAN"`, `language="en-US"` | The greeting spoken over the music bed. |
| `<Wait>` | `length="120"` | Holds the line open so the mixed audio is audible. In a real deployment replace this with `<Stream>` for an AI agent or `<Dial>` to bridge an agent. |
| `<Hangup/>` | — | Ends the call once the wait elapses. |

### Control API usage

Start or swap the track on a live call:

```bash
curl -X POST https://your-url/trigger-background \
  -H "Content-Type: application/json" \
  -d '{
    "call_uuid": "CALL_UUID_HERE",
    "url": "https://example.com/your-music.mp3"
  }'
```

```json
{
  "status": "started",
  "call_uuid": "abc-123",
  "audio_url": "https://example.com/your-music.mp3",
  "vobiz_response": { "message": "play started", "api_id": "..." }
}
```

Stop the music while leaving the call up:

```bash
curl -X POST https://your-url/stop-background \
  -H "Content-Type: application/json" \
  -d '{ "call_uuid": "CALL_UUID_HERE" }'
```

Inspect live state:

```bash
curl https://your-url/status
```

```json
{
  "active_call_count": 1,
  "active_calls": {
    "51d473ff-4522-44fd-87d9-fce46c827c21": {
      "from": "+15550003333",
      "music_playing": true
    }
  },
  "default_audio_url": "https://www.soundhelix.com/examples/mp3/SoundHelix-Song-1.mp3"
}
```

## Audio file requirements

- **MP3 or WAV** — both are supported.
- The file must be on a **public HTTPS URL** that Vobiz itself can fetch: S3, a
  CDN, or GitHub releases all work. A URL reachable only from your laptop, or one
  behind authentication, will fail at the platform rather than in your code.
- For telephony, **8 kHz or 16 kHz mono** sources sound best. Calls are carried at
  narrowband, so a 44.1 kHz stereo master buys you nothing and is downmixed and
  resampled before it reaches the caller.
- **Master the bed quieter than speech.** `mix: true` blends the two streams; if
  the track is mastered loud it will compete with the voice. Attenuate the file
  itself before hosting it — the mix is not level-controlled from the payload.
- **Use a clean-looping track.** `loop: true` restarts the file at its end, so any
  silence, fade, or dangling reverb tail at the boundary is audible on every
  repeat. Trim to a true loop point.
- **A 30–120 second clip works well.** Shorter loops become noticeable quickly on
  a long call.

## Security notes

- **Credentials live in `.env` only.** `.env` is git-ignored and `.env.example`
  contains placeholders. Never commit a real auth token.
- **The control API is unauthenticated as written.** Anyone who can reach
  `/trigger-background` with a valid `call_uuid` can inject audio into a live
  call. Put it behind your own authentication before it leaves localhost.
- **The Vobiz webhooks are unauthenticated as written.** Restrict `/answer` and
  `/hangup` to Vobiz by source-IP allowlist, a shared secret in the URL, or
  signature validation before deploying.
- **`/status` exposes caller numbers.** The `active_calls` map includes the `From`
  of every live call, which is personal data. Do not leave it publicly reachable.
- **`url` on `/trigger-background` is passed straight to the Play API.** It is
  caller-controlled; validate it against an allowlist of hosts if the endpoint is
  reachable by anyone but your own backend.
- **ngrok tunnels are public.** Shut the tunnel down when you have finished
  testing.

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| Call connects and the greeting plays, but there is no music | The Play API call failed after `/answer` had already returned, so nothing surfaced in the call. | Look for `Live Play start failed` in the log and check `music_playing` via `/status`. The usual causes are bad credentials or an audio URL Vobiz cannot fetch. |
| `Live Play start failed` with an HTTP error | Wrong `VOBIZ_AUTH_ID`/`VOBIZ_AUTH_TOKEN`, or the `call_uuid` is no longer live. | Verify credentials, and confirm the UUID appears in `/status` — playback can only be started on a call that is currently up. |
| Music plays but the voice is drowned out | The source file is mastered too loud. `mix: true` blends the streams without level control. | Attenuate the audio file itself, re-host it, and update `BACKGROUND_AUDIO_URL`. |
| Music plays but the caller hears it once, not looping | The playback was replaced or stopped — a second `POST` to Play supersedes the first. | Check for a stray `/trigger-background` or `/stop-background` call; `loop: true` otherwise repeats until the call ends. |
| An audible click, gap, or silence on every repeat | The audio file does not loop cleanly — leading or trailing silence, or a reverb tail. | Trim the file to an exact loop point and re-host it. |
| Credentials appear empty although `.env` is filled in | The file is not next to `server.py`. | The loader resolves `.env` relative to `server.py`; run `cp .env.example .env` inside the repository directory. |
| `/trigger-background` returns `{"error": "JSON body required"}` | The request had no JSON body or was missing `Content-Type: application/json`. | Send the header and a valid JSON object containing `call_uuid`. |
| Music keeps playing after the caller hangs up | Not possible on that call — playback dies with the call. If `/status` still lists it, the `/hangup` webhook never arrived. | Confirm the Hangup URL is set on the Vobiz application and reachable; `active_calls` is only cleaned up there. |
| Server exits at startup with an ngrok error | No `NGROK_AUTH_TOKEN` and the anonymous tunnel limit was hit, or a tunnel is already open. | Set `NGROK_AUTH_TOKEN`, kill any stray `ngrok` process, or set `PUBLIC_URL` to skip tunnelling. |

## Roadmap

> Planned improvements to this example. Ideas and pull requests are welcome —
> open an issue to discuss anything here.

- [ ] Report Play API failures back into the call rather than only to the log, so
      a silent bed is visible to the caller-facing flow.
- [ ] Retry `call_play_start()` with backoff instead of firing once from the
      deferred thread.
- [ ] Move `active_calls` to Redis so state survives a restart and the service can
      run more than one replica.
- [ ] Replace the `<Wait>` placeholder with a worked `<Stream>` example, showing
      background audio underneath a live AI voice agent.
- [ ] Add authentication to the control API and signature verification on the
      webhooks.
- [ ] Add a test suite covering the Play payload, the deferred-start thread, and
      the `/trigger-background` validation branches.
- [ ] Support per-call track selection driven by the dialled number or an
      application parameter, rather than one global default.

## Contributing

Issues and pull requests are welcome. Before opening a PR:

```bash
pip install -r requirements.txt
python -m compileall server.py background_store.py
python server.py        # confirm the server starts and /health responds
```

Keep changes grounded in what the code actually does, and update the
Configuration and API reference tables if you add or rename anything.

## License

Released under the [MIT License](./LICENSE) © Vobiz.

MIT is permissive: you may use, modify, and redistribute this code, including in
closed-source commercial products, provided the copyright notice and licence text
are retained. There is no warranty. If your organisation needs a different
licensing arrangement, contact [piyush@vobiz.ai](mailto:piyush@vobiz.ai).

## Built by Team Vobiz

[Vobiz](https://vobiz.ai) is a programmable voice and SIP-trunking platform for
voice APIs, SIP trunking, and AI voice agents. This repository is built and
maintained by the Vobiz team.

**Maintainer:** Piyush Sahoo — [piyush@vobiz.ai](mailto:piyush@vobiz.ai) · [LinkedIn](https://www.linkedin.com/in/piyush-s713/)

Questions, or want to talk through an integration? Open an issue on this repo,
or reach out directly at [piyush@vobiz.ai](mailto:piyush@vobiz.ai).

**Useful links:** [Docs](https://docs.vobiz.ai) · [API reference](https://docs.vobiz.ai/api-reference) · [Sign up](https://vobiz.ai)
