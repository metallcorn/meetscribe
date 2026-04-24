# MeetScribe

Real-time meeting transcription that saves everything locally — no cloud, no subscriptions, no data leaving your machine.

Captures audio from any browser tab (Google Meet, Zoom, Slack, etc.) and transcribes it live. All transcripts, audio recordings, and AI summaries are stored as plain files in a folder you choose.

---

## Features

- **Live transcription** — see text appear as people speak
- **Speaker detection** — automatically labels "You" vs "Speaker"
- **AI summaries** — generate meeting reports with one click (✨)
- **Chat with transcript** — ask questions about the meeting (💬)
- **AI transcription** — re-process audio post-meeting via Deepgram or Gemini for better accuracy (🎙)
- **AI rename** — automatically suggests a meaningful name for each recording
- **Screen analysis** — describe screenshots taken during the meeting (🖼)
- **100% local** — files saved to your disk, no server, no account required

---

## Quick Start

```bash
npx serve .
# Open http://localhost:3000
```

1. Click **📁 Choose folder** — pick where to save transcriptions
2. Go to **⚙ Settings** — choose a transcription engine and enter your API key
3. Click **▶ Start recording** — select the browser tab you want to capture
4. Check **"Share tab audio"** in the browser dialog

> Requires **Chrome 107+** or **Edge 107+** on Windows or Linux.  
> Firefox and Safari are not supported (no tab audio capture).

---

## Transcription Engines

| Engine | Key needed | Notes |
|---|---|---|
| **Gemini Live** | [Google AI Studio](https://aistudio.google.com/apikey) | Best quality, captures tab + mic |
| **Deepgram Live** | [console.deepgram.com](https://console.deepgram.com) | Fast, good accuracy |
| **Groq / Whisper** | [console.groq.com](https://console.groq.com/keys) | Whisper Large v3 Turbo |
| **Web Speech API** | None | Free, microphone only |

---

## AI Features (optional)

For summaries, chat, and AI rename — configure a REST API key in Settings:

| Provider | Model |
|---|---|
| Gemini | gemini-2.5-flash |
| OpenAI | gpt-4o and others |
| Mistral | mistral-large |
| Custom | any OpenAI-compatible URL |

---

## File Structure

Everything is saved as plain files — easy to share, open in any editor, or feed into other AI tools:

```
~/Documents/MeetScribe/
├── 2026-04-24_standup-team/
│   ├── 2026-04-24_standup-team.txt      ← transcript
│   ├── 2026-04-24_standup-team.webm     ← audio
│   └── 2026-04-24_standup-team_dg.txt  ← AI transcript (optional)
├── prompts/
│   └── meeting-minutes.txt              ← your custom summary prompts
└── backgrounds/
    └── office.jpg                       ← optional UI background
```

Custom summary prompts go in the `prompts/` folder. Use `{{transcript}}` as a placeholder for the transcript text.

---

## Keyboard Shortcuts

| Key | Action |
|---|---|
| `Alt+R` | Start recording |
| `Alt+S` | Stop recording |
| `Alt+N` | Stop and start new |
| `Alt+M` | Mute / unmute mic |
| `Alt+F` | Search sessions |
