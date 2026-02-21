# Voice AI Scheduling Agent

> A real-time voice assistant that books appointments on your Google Calendar through a natural conversation. Just speak — the AI collects your name, preferred date/time, and meeting title, then creates the event automatically.

<!-- Screenshot placeholder – replace with your actual screenshot -->
![Voice AI Agent Screenshot](https://via.placeholder.com/900x500/0f0f19/6c63ff?text=Voice+AI+Scheduling+Agent)

---

## Live Demo

| Resource | Link |
|----------|------|
| **Live App** | [YOUR_DEPLOYED_URL]([https://YOUR_DEPLOYED_URL](https://voiceai-agent.vercel.app/)) |
| **Demo Video** | [Watch on Google Drive](https://drive.google.com/file/d/YOUR_VIDEO_ID/view) |

> **Note:** The live demo requires the backend server to be running. If the call connects but doesn't book, the backend may be offline.

---

## How to Use the Agent

1. **Open the app** — visit the live URL above or run it locally (see below).
2. **Click the purple phone button** at the bottom-right corner.
3. **Allow microphone access** when your browser prompts.
4. **Talk to the assistant** — it will guide you through:

   | Step | What the AI asks | What you say |
   |------|-----------------|--------------|
   | 1 | *"Who am I speaking with?"* | Your name |
   | 2 | *"What date and time?"* | e.g. "February 22nd at 8 AM" |
   | 3 | *"What's the meeting title?"* | e.g. "Team standup" |
   | 4 | *"Just to confirm… Is that correct?"* | "Yes" |

5. **The AI books it** — you'll hear *"Your appointment has been booked!"*
6. **Check Google Calendar** — the event appears within seconds.

---

## Screenshots

<table>
  <tr>
    <td><img src="https://via.placeholder.com/400x300/0f0f19/6c63ff?text=Landing+Page" alt="Landing Page" /></td>
    <td><img src="https://via.placeholder.com/400x300/0f0f19/34d399?text=Call+In+Progress" alt="Call In Progress" /></td>
  </tr>
  <tr>
    <td align="center"><em>Landing page with call button</em></td>
    <td align="center"><em>Live call with transcript</em></td>
  </tr>
  <tr>
    <td><img src="https://via.placeholder.com/400x300/0f0f19/f59e0b?text=Google+Calendar+Event" alt="Calendar Event" /></td>
    <td><img src="https://via.placeholder.com/400x300/0f0f19/ef4444?text=Backend+Logs" alt="Backend Logs" /></td>
  </tr>
  <tr>
    <td align="center"><em>Event created in Google Calendar</em></td>
    <td align="center"><em>Backend processing the tool call</em></td>
  </tr>
</table>

> **Replace** the placeholder images above with your actual screenshots.

---

## Architecture

```
┌──────────────┐     WebRTC      ┌──────────────┐    webhook     ┌──────────────┐
│   Browser    │ ◄─────────────► │  Vapi Cloud  │ ─────────────► │  FastAPI     │
│   (Frontend) │   voice call    │  (AI + STT   │  tool-calls    │  (Backend)   │
│              │                 │   + TTS)     │                │              │
│  index.html  │                 │  GPT-4o +    │                │  main.py     │
│  app.js      │                 │  Deepgram    │                │  calendar_   │
│  style.css   │                 │              │                │  utils.py    │
└──────────────┘                 └──────────────┘                └──────┬───────┘
                                                                       │
                                                                       │ Google
                                                                       │ Calendar
                                                                       │ API v3
                                                                       ▼
                                                                ┌──────────────┐
                                                                │   Google     │
                                                                │   Calendar   │
                                                                └──────────────┘
```

### Flow

1. User clicks the call button → **Vapi** connects a WebRTC voice call via browser mic.
2. Vapi's AI assistant (GPT-4o) conducts the conversation, collecting name, date/time, and title.
3. Once confirmed, the AI triggers the `book_appointment` function → Vapi sends a **webhook POST** to the FastAPI backend.
4. The backend parses the tool-call payload, calls the **Google Calendar API**, and inserts the event.
5. The result is returned to Vapi → the AI reads the confirmation aloud.

---

## Project Structure

```
VoiceaiAgent/
├── frontend/
│   ├── index.html            # Landing page with Vapi widget
│   ├── style.css             # Dark glassmorphism UI
│   └── app.js                # Transcript display handler
├── backend/
│   ├── main.py               # FastAPI – webhook endpoint for Vapi
│   ├── calendar_utils.py     # Google Calendar auth & event creation
│   ├── creds.json            # Google Service Account key (DO NOT commit)
│   └── .env                  # Environment config
├── requirements.txt          # Python dependencies
├── .gitignore
└── README.md
```

---

## Google Calendar Integration

### Authentication

The backend uses a **Google Cloud Service Account** for server-to-server authentication — no browser popup or user consent needed at runtime.

| Component | Description |
|-----------|-------------|
| `creds.json` | Service Account key file downloaded from Google Cloud Console |
| `GOOGLE_SERVICE_ACCOUNT_FILE` | `.env` variable pointing to the key file |
| `GOOGLE_CALENDAR_ID` | Your Gmail address — you share your calendar with the service account |

**How it works:**
1. The service account authenticates using its private key (no tokens to refresh manually).
2. You share your personal Google Calendar with the service account's email address (found in `creds.json` → `client_email`).
3. The service account can then create/read/update events on your calendar.

### Event Creation

`calendar_utils.py → create_event()` handles all event logic:

| Parameter | Type | Required | Default |
|-----------|------|----------|---------|
| `name` | string | Yes | — |
| `start_time` | ISO 8601 string | Yes | — |
| `end_time` | ISO 8601 string | No | start + 30 min |
| `title` | string | No | "Meeting with {name}" |

**What gets created:**
- **Summary**: The meeting title
- **Description**: "Scheduled by Voice AI Agent for {name}"
- **Time zone**: `Asia/Kathmandu` (configurable in `.env`)
- **Calendar**: Your personal Gmail calendar

### Webhook Endpoint

`main.py → POST /webhook/calendar`

- Receives **all** Vapi server messages (speech-update, transcript, status-update, etc.)
- Returns `200 OK` for informational events (no processing needed)
- Processes only `tool-calls` type → extracts parameters → calls `create_event()` → returns result to Vapi
- Handles both Vapi payload formats (`toolCallList[].arguments` and legacy `function.arguments`)

---

## Run Locally

### Prerequisites

| Tool | Purpose |
|------|---------|
| Python 3.12+ | Backend runtime |
| ngrok | Expose local backend to the internet |
| Google Cloud project | Service Account + Calendar API enabled |
| Vapi account | AI voice assistant platform |

### 1. Clone & Install

```bash
git clone https://github.com/YOUR_USERNAME/VoiceaiAgent.git
cd VoiceaiAgent

# Create virtual environment
python3 -m venv myenv
source myenv/bin/activate

# Install dependencies
pip install -r requirements.txt
```

### 2. Google Cloud Setup

1. Go to [Google Cloud Console](https://console.cloud.google.com/)
2. Create a project → Enable **Google Calendar API**
3. Go to **IAM & Admin** → **Service Accounts** → Create a service account
4. Create a JSON key → download and save as `backend/creds.json`
5. Go to [Google Calendar Settings](https://calendar.google.com) → **Share with specific people** → add the service account email (from `creds.json` → `client_email`) with **"Make changes to events"** permission

### 3. Configure Environment

Create `backend/.env`:

```env
GOOGLE_SERVICE_ACCOUNT_FILE=creds.json
GOOGLE_CALENDAR_ID=your-email@gmail.com
TIMEZONE=Asia/Kathmandu
```

### 4. Vapi Dashboard Setup

1. Sign up at [vapi.ai](https://vapi.ai) and create an assistant
2. Set the system prompt to instruct it to collect name, date/time, and title
3. Create a **Function tool** named `book_appointment` with parameters:
   - `name` (string, required) — caller's name
   - `start_time` (string, required) — ISO 8601 datetime
   - `end_time` (string, optional) — ISO 8601 datetime
   - `title` (string, optional) — meeting title
4. Set the tool's **Server URL** to your ngrok URL + `/webhook/calendar`
5. Attach the tool to your assistant

### 5. Start ngrok

```bash
ngrok http 8000
```

Copy the HTTPS forwarding URL and update it in the Vapi tool's Server URL.

### 6. Start the Backend

```bash
source myenv/bin/activate
cd backend
uvicorn main:app --host 0.0.0.0 --port 8000 --reload
```

Verify:
```bash
curl http://localhost:8000/health
# → {"status": "ok"}
```

### 7. Start the Frontend

```bash
cd frontend
python -m http.server 5500
```

Open **http://localhost:5500** in your browser.

### 8. Test End-to-End

1. Click the purple phone button (bottom-right)
2. Allow microphone access
3. Have a conversation → provide name, date/time, title
4. Confirm → the event appears in your Google Calendar
5. Check `backend/webhook.log` for detailed request logs

---

## Deployment (Free Tier)

### Frontend → Vercel / Netlify / GitHub Pages

The frontend is static HTML/CSS/JS — deploy to any static host for free:

**Vercel:**
```bash
npm i -g vercel
cd frontend
vercel --prod
```

**Netlify:**
- Drag-and-drop the `frontend/` folder at [app.netlify.com/drop](https://app.netlify.com/drop)

**GitHub Pages:**
- Push to GitHub → Settings → Pages → Source: `main` branch, folder: `/frontend`

### Backend → Render (Free)

[Render](https://render.com) is the best free option for FastAPI — it gives you a stable HTTPS URL (no more ngrok).

1. Push your repo to GitHub
2. Go to [render.com](https://render.com) → New **Web Service**
3. Connect your repo, set:
   - **Root Directory**: `backend`
   - **Build Command**: `pip install -r ../requirements.txt`
   - **Start Command**: `uvicorn main:app --host 0.0.0.0 --port $PORT`
4. Add environment variables in the Render dashboard:
   - `GOOGLE_SERVICE_ACCOUNT_FILE` = `creds.json`
   - `GOOGLE_CALENDAR_ID` = `your-email@gmail.com`
   - `TIMEZONE` = `Asia/Kathmandu`
5. Upload `creds.json` as a secret file
6. Update the Vapi tool's **Server URL** to `https://your-app.onrender.com/webhook/calendar`

> Once deployed to Render, **ngrok is no longer needed**. The Render URL is permanent.

---

## Tech Stack

| Layer | Technology | Purpose |
|-------|-----------|---------|
| Voice AI | [Vapi](https://vapi.ai) | WebRTC calls, STT, TTS, LLM orchestration |
| LLM | GPT-4o (via Vapi) | Conversational intelligence & function calling |
| Backend | [FastAPI](https://fastapi.tiangolo.com/) | Webhook handler for tool-calls |
| Calendar | [Google Calendar API v3](https://developers.google.com/calendar) | Event creation |
| Auth | Google Service Account | Server-to-server authentication |
| Frontend | Vanilla HTML / CSS / JS | No build step, CDN-loaded Vapi widget |
| Tunnel | [ngrok](https://ngrok.com) | Local development tunneling |

---

## Environment Variables

| Variable | Description | Example |
|----------|-------------|---------|
| `GOOGLE_SERVICE_ACCOUNT_FILE` | Path to service account JSON key | `creds.json` |
| `GOOGLE_CALENDAR_ID` | Gmail address of your calendar | `you@gmail.com` |
| `TIMEZONE` | IANA timezone for events | `Asia/Kathmandu` |

---

## Troubleshooting

| Problem | Solution |
|---------|----------|
| Call button doesn't appear | Check internet — Vapi widget loads from CDN |
| Call connects but AI doesn't respond | Check Vapi dashboard for credit balance |
| AI talks but doesn't book | Verify the Function tool's Server URL points to your backend |
| Backend error "Address already in use" | Run `fuser -k 8000/tcp` then restart |
| Event not in your calendar | Ensure you shared your calendar with the service account email |
| ngrok URL changed | Update the Server URL in the Vapi tool settings |
| "No valid Google credentials" | Check that `creds.json` exists in `backend/` |

---

## License

MIT

---

<p align="center">
  Built with Vapi + FastAPI + Google Calendar API
</p>
