# LiveKit × Twilio AI Outbound Calling

Trigger AI-powered outbound phone calls from a browser. The AI agent dials a number via Twilio SIP, waits for the person to pick up, then holds a real-time voice conversation powered by LiveKit Inference, Deepgram STT, and OpenAI.

## Stack

| Layer | Technology |
|-------|-----------|
| Voice pipeline | LiveKit Agents v1.5+ (`Agent` + `AgentSession`) |
| STT | Deepgram nova-2-phonecall (via LiveKit Inference) |
| LLM | OpenAI gpt-4o-mini |
| TTS | Cartesia Sonic-2 and others (via LiveKit Inference) |
| VAD | Silero |
| PSTN | Twilio Elastic SIP Trunking → LiveKit SIP |
| API server | FastAPI + uvicorn |
| Frontend | Vanilla HTML/JS + Tailwind CDN |

> **LiveKit Inference** is LiveKit's unified AI gateway. It proxies STT/TTS to Deepgram, Cartesia, Rime, ElevenLabs, and others — authenticated with your existing LiveKit API key. No separate Deepgram or ElevenLabs API keys are required.

---

## Prerequisites

- Python 3.11+
- A [LiveKit Cloud](https://cloud.livekit.io) project
- A [Twilio](https://twilio.com) account with:
  - An Elastic SIP Trunk configured for outbound calling
  - A phone number assigned to the trunk as the caller ID
- An [OpenAI](https://platform.openai.com) API key

---

## 1. Clone & install

```bash
git clone <repo-url>
cd livekit-twilio-ai-outbound-calling

python -m venv .venv

# Windows
.venv\Scripts\activate

# macOS / Linux
source .venv/bin/activate

pip install -r requirements.txt
```

---

## 2. Environment variables

```bash
cp .env.example .env
```

Open `.env` and fill in every value:

| Variable | Where to find it |
|----------|-----------------|
| `LIVEKIT_URL` | LiveKit Cloud → project → Settings → URL (`wss://...livekit.cloud`) |
| `LIVEKIT_API_KEY` | LiveKit Cloud → project → Settings → Keys |
| `LIVEKIT_API_SECRET` | LiveKit Cloud → project → Settings → Keys |
| `LIVEKIT_SIP_TRUNK_ID` | LiveKit Cloud → SIP → Outbound Trunks (created in step 3) |
| `OPENAI_API_KEY` | platform.openai.com → API keys |

---

## 3. Set up Twilio → LiveKit SIP outbound trunk

### 3a. Create a Twilio Elastic SIP Trunk

1. Twilio Console → **Elastic SIP Trunking** → **Trunks** → **Create new trunk**
2. Give it a friendly name (e.g. `AI Outbound Calls`)
3. Under **Origination**, add your LiveKit SIP address as an origination URI:
   - Format: `sip:<your-livekit-sip-domain>` (found in LiveKit Cloud → SIP → Settings)
4. Under **Phone Numbers**, attach the number that will appear as the caller ID

### 3b. Create a LiveKit outbound SIP trunk

1. LiveKit Cloud → **SIP** → **Outbound Trunks** → **Add trunk**
2. Set **SIP server address** to your Twilio termination URI:
   - Format: `your-trunk-name.pstn.twilio.com`
3. Add a **SIP credential** (username + password) — Twilio will validate these on every call
4. Set **media encryption** to **Disabled** (Twilio does not support SRTP on standard trunks)
5. Copy the resulting **Trunk ID** and set it as `LIVEKIT_SIP_TRUNK_ID` in `.env`

### 3c. Important trunk settings

The trunk **must** have media encryption disabled. If it was created with encryption enabled, update it via the LiveKit CLI or API:

```bash
lk sip outbound update <TRUNK_ID> --media-encryption=0
```

Or via the LiveKit Python SDK:

```python
from livekit import api
from livekit.protocol.sip import SIPOutboundTrunkInfo, SIP_MEDIA_ENCRYPT_DISABLE

lk = api.LiveKitAPI(url=..., api_key=..., api_secret=...)
await lk.sip.update_outbound_trunk(
    trunk_id="ST_...",
    trunk=SIPOutboundTrunkInfo(media_encryption=SIP_MEDIA_ENCRYPT_DISABLE),
)
```

---

## 4. Running the application

Both processes must run simultaneously (two terminals).

### Terminal 1 — Agent worker

```bash
# Windows
.venv\Scripts\activate

# macOS / Linux
source .venv/bin/activate

python agent.py dev
```

`dev` mode connects to LiveKit Cloud and waits for dispatch jobs. You should see:

```
INFO  livekit.agents  Starting worker...
INFO  livekit.agents  Connected to LiveKit Cloud
```

### Terminal 2 — API server

```bash
# activate venv as above
python server.py
```

Server starts at `http://localhost:8000`.

Open `http://localhost:8000/static/index.html` in a browser.

---

## 5. Using the UI

The browser UI has four tabs:

| Tab | Purpose |
|-----|---------|
| **Voice Setup** | Browse and activate a TTS voice (Cartesia, Deepgram, Rime). The selected voice applies to all calls in the current session. |
| **Campaign** | Upload or paste a contact list (phone + optional personalization notes), set agent instructions, and launch a bulk campaign. |
| **Single Call** | Dial one number manually with custom instructions. |
| **Active Calls** | View all live rooms, auto-refreshed every 10 seconds. Stop any call immediately. |

---

## API reference

### `POST /call`

Dispatch a single outbound call.

**Request:**

```json
{
  "phone_number": "+12125551234",
  "agent_name": "Alex",
  "instructions": "You are Alex, a friendly assistant following up on a demo request.",
  "voice_id": "cartesia/sonic-2:79a125e8-cd45-4c13-8a67-188112f4dd22"
}
```

`voice_id` is optional. If omitted, defaults to `cartesia/sonic-2` (default Cartesia voice).

**Response:**

```json
{
  "success": true,
  "room_name": "call-a3f9c12b8e01",
  "message": "Outbound call to +12125551234 dispatched. Room: call-a3f9c12b8e01"
}
```

### `POST /campaign`

Dispatch a batch of outbound calls. Contacts are dialed sequentially with a 1.5 s delay between dispatches.

**Request:**

```json
{
  "base_instructions": "You are a friendly assistant following up on a demo request.",
  "agent_name": "Alex",
  "voice_id": "cartesia/sonic-2",
  "contacts": [
    { "phone_number": "+12125551234", "personalization_context": "Asked about enterprise pricing." },
    { "phone_number": "+13105559876", "personalization_context": "Trial user, signed up 2 days ago." }
  ]
}
```

Maximum 500 contacts per campaign.

**Response:**

```json
{ "campaign_id": "abc123...", "total": 2 }
```

### `GET /campaign/{campaign_id}`

Poll campaign progress.

```json
{
  "campaign_id": "abc123...",
  "total": 2,
  "counts": { "pending": 0, "dispatching": 0, "dispatched": 2, "error": 0 },
  "contacts": [...]
}
```

### `GET /active-calls`

List all live rooms (single calls and campaigns).

### `DELETE /active-calls/{room_name}`

Immediately terminate a call by deleting its room.

### `GET /health`

```json
{ "status": "ok" }
```

---

## How it works

1. Browser POSTs to `/call` (or `/campaign`) with a phone number and agent config
2. FastAPI creates a LiveKit room name and calls `agent_dispatch.create_dispatch`, sending the config as JSON job metadata
3. The agent worker picks up the job and connects to the room
4. The agent calls `sip.create_sip_participant` with the phone number — LiveKit routes this through the configured Twilio SIP trunk
5. Twilio dials the number over PSTN; when the person answers, a SIP participant joins the LiveKit room
6. The agent greets the callee (using `session.generate_reply`) and the real-time voice conversation begins
7. The agent uses LiveKit Inference for STT (Deepgram) and TTS (Cartesia/Rime/etc.) — no separate provider credentials needed

---

## Troubleshooting

| Error | Cause | Fix |
|-------|-------|-----|
| `SipCallTo should be a phone number or SIP user, not a full SIP URI` | Passing `sip:+1...@domain` to `sip_call_to` | Pass only the E.164 number; the trunk handles the domain |
| `488 Not Acceptable Here` / SRTP rejection | Trunk has `media_encryption=1` (SRTP required) but Twilio doesn't support it | Set trunk `media_encryption` to `0` (disabled) |
| `400 32101 The called number is not correctly formatted` | Wrong SIP URI format | `sip_call_to` should be just the E.164 number (e.g. `+12125551234`) |
| `module 'livekit.api' has no attribute 'AgentDispatchClient'` | Old SDK | Use `lk.agent_dispatch.create_dispatch(...)` |
| Agent worker connects but no calls go through | Worker not registered with correct `agent_name` | Confirm `AGENT_NAME` in `agent.py` matches the name used in `create_dispatch` |
