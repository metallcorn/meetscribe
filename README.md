# MeetScribe

Real-time meeting transcription that runs in your browser and keeps every file on your own disk — no account, no subscription, no vendor cloud storing your meetings.

Capture audio from a browser tab (Google Meet, Zoom, Slack in the browser) **or** your whole system (Teams / Zoom desktop apps via a loopback device), transcribe it live, and turn it into summaries, searchable notes and chat — all saved as plain files in a folder you choose.

---

## How your data flows (the honest version)

MeetScribe has **no server of its own** — it's a static web page. But it is **not** a fully offline/local tool, because there's no on-device speech model:

- **Your files are local.** Transcripts, audio and summaries are written straight to a folder on your disk via the File System Access API. No cloud storage, no account, nothing uploaded to us — there is no "us".
- **Recognition uses your own API keys.** To turn speech into text (and to summarize/chat), audio and text are sent to the provider **you** choose — Deepgram, Google Gemini, Groq, OpenAI or Mistral — under your own key. That data leaves your machine and goes to that provider, on their terms.
- **You pay providers directly.** No middleman, no markup, no per-seat subscription. Even the free "Web Speech" engine sends audio to Google (that's how the browser API works).

So: **local-first storage, bring-your-own-key processing.** If a meeting must never leave the device, this tool isn't for that (yet — no local model).

---

## What makes it different

- **Own your files.** Everything is plain `.txt` / `.webm` / `.json` in your folder — portable, greppable, feedable into any other tool. Delete the tab and your data is still just… files.
- **No subscription, no lock-in.** Bring your keys, pay providers at cost.
- **Capture more than a tab.** Browser tabs *and* desktop apps (Teams, Zoom client, Slack Huddles) via a system-audio/loopback device.
- **Two speeds.** A quick live draft while you meet, plus optional high-accuracy re-transcription afterwards (Deepgram, with speaker separation).
- **A real post-meeting pipeline.** Transcribe → recognize on-screen slides → summarize → chat — each step optional, each with your choice of model. Turn on auto-processing to run it all on stop.
- **Runs anywhere Chrome does.** One HTML page, no install, no build.

---

## Quick Start

```bash
npx serve .
# Open http://localhost:3000
```

1. **📁 Choose folder** — where recordings are saved (the browser remembers it).
2. **⚙ Settings** — pick a transcription engine and paste your API key.
3. **▶ Start recording** — select the tab to capture and tick **"Share tab audio"**.
4. After the meeting, open the recording and follow the steps (transcript → screens → summary), or enable **auto-process** to do it automatically.

> Requires **Chrome 107+** or **Edge 107+** (Windows, macOS, Linux).
> Firefox and Safari are unsupported — no tab-audio capture / File System Access API.

---

## Capturing system audio (Teams / desktop apps)

The browser can only grab **tab** audio directly. To capture a desktop app, point **Settings → Audio source** at a loopback input device:

| OS | Loopback device |
|---|---|
| **Linux** | The `Monitor of …` source (already there in PipeWire/PulseAudio) |
| **macOS** | [BlackHole](https://existential.audio/blackhole/) (free) as a Multi-Output Device |
| **Windows** | "Stereo Mix" or [VB-Cable](https://vb-audio.com/Cable/) — or share the whole screen with system audio |

⚠️ A loopback source captures **all** system sound (every app and notification), not just the meeting.

---

## Transcription Engines

Chosen in Settings. Speaker separation ("diarization") is available when re-processing the recording, not live.

| Engine | Key | Notes |
|---|---|---|
| **Deepgram** | [console.deepgram.com](https://console.deepgram.com) | Fast, accurate (~$0.003/min). Speaker separation when processing the recording |
| **Groq / Whisper** | [console.groq.com](https://console.groq.com/keys) | Whisper Large v3 Turbo. Very cheap, no speaker labels |
| **Gemini Live** | [Google AI Studio](https://aistudio.google.com/apikey) | Works live; average quality, higher cost |
| **Web Speech** | none | Free, **your microphone only** (not tab audio). For quick notes |

---

## AI features (optional)

Summaries (✨), chat (💬), screenshot analysis (🖼) and auto-naming use a REST key — Gemini, OpenAI, Mistral, or any OpenAI-compatible URL. Recognized on-screen text is folded into the summary so it reflects what was shown, not just what was said.

Custom summary prompts live in a `prompts/` folder inside your transcription folder — drop in `.txt` files and they appear in the ✨ menu. Use `{{transcript}}` as the placeholder for the transcript.

---

## File structure

Everything is plain files — share them, open them in any editor, feed them to other AI tools:

```
~/Documents/MeetScribe/
├── 2026-04-24_standup/
│   ├── 2026-04-24_standup.txt        ← live transcript (draft)
│   ├── 2026-04-24_standup_dg.txt     ← accurate transcript, from the recording (optional)
│   ├── 2026-04-24_standup.webm       ← audio
│   ├── 2026-04-24_standup_chat.json  ← chat history (optional)
│   └── screenshots/                  ← captured slides (optional)
├── prompts/                          ← your summary prompts
└── backgrounds/                      ← optional UI backgrounds
```

---

## Keyboard shortcuts

| Key | Action |
|---|---|
| `Alt+R` | Start recording |
| `Alt+S` | Stop recording |
| `Alt+N` | Stop and start new |
| `Alt+M` | Mute / unmute mic |
| `Alt+F` | Search sessions |
