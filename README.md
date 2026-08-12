# Shadow — a private, JARVIS-inspired personal AI assistant

Shadow is a voice assistant I built from scratch that runs across my
Android phone and Windows desktop: wake-word activated ("Hey Shadow"),
aware of which device should answer when both hear you, routes what you
say through a multi-tier AI system to figure out what you actually want,
and talks back in a fully offline neural voice.

This repo is a portfolio showcase — architecture, design decisions, and
the real engineering problems I hit and solved. **The full working source
is in a private repo**; happy to share access if you'd like to see it —
reach out.

---

## What it does

- **Wake-word activation** on both a Windows desktop app and an Android
  client, running a local Whisper model to detect "Hey Shadow" and
  transcribe what follows.
- **Cross-device arbitration** — if both my phone and desktop hear the
  wake phrase at once, they negotiate over the network so only one
  answers, instead of both talking over each other.
- **AI-based command routing** — instead of brittle regex trigger
  phrases, spoken commands (however messily phrased, in any word order)
  are classified and parsed into structured actions by an LLM, with a
  three-tier fallback chain: a free fast model first, a free fallback
  model second, and a paid model only as a last resort or on explicit
  request — with a hard, persistent spend cap so the paid tier can never
  run away on cost.
- **Real skills**: note-taking (with a multi-turn "anything else to add?"
  confirmation loop so a dropped mic never loses your note, and automatic
  date-based file organization), calendar scheduling via the Google
  Calendar API, spoken definitions/translations/news briefings, and a
  personality system with switchable voices.
- **Fully offline text-to-speech** via a local neural TTS engine — no
  cloud voice API, no per-word cost, works without internet.
- **Live-tunable behavior** — things like how patient it is before
  assuming you've stopped talking, mic sensitivity, and how often it adds
  a little personality to routine replies are all adjustable by voice
  command in real time, not hardcoded constants that need a redeploy.

## Architecture

```
┌─────────────────┐        ┌──────────────────┐
│  Android client   │◄──────►│  Windows desktop  │
│  (Kotlin)          │  private  │  Core (FastAPI /  │
│                     │  network  │  WebSocket) +      │
└─────────────────┘        │  voice pipeline    │
                             └──────────────────┘
                                       │
                         ┌─────────────┴─────────────┐
                         │      AI command router      │
                         │  fast free tier → free      │
                         │  fallback → paid escalation  │
                         │  (budget-capped)              │
                         └─────────────┬─────────────┘
                                       │
                    ┌──────────┬───────┼───────┬──────────┐
                 notes     calendar  knowledge  persona  settings
```

## A few of the harder problems I solved

**A silent startup crash with no error output.** The desktop app runs
with no console window via `pythonw.exe`, whose `sys.stdout` isn't just
non-interactive — it's `None`. That broke the web server's default
logging setup in a way that produced zero diagnostic output, so the app
just silently died. Root-caused by instrumenting every startup step with
explicit file logging and defensively handling the `None` stream case.

**A multi-second lag after every wake phrase.** Traced to a Windows
quirk: resolving `"localhost"` tries IPv6 before falling back to IPv4,
and that fallback delay got worse with a VPN mesh network's virtual
interfaces active. Fixed by connecting to `127.0.0.1` explicitly instead
of relying on hostname resolution.

**Regex trigger phrases couldn't handle real speech.** Commands like
"meeting Friday 2:32pm calendar" — trigger word at the end, date and time
in the middle — broke phrase-anchored regex matching entirely. That
pushed a mid-project pivot to full AI-based intent classification and
slot extraction, which handles arbitrary phrasing and word order for
free.

**Losing the first word of every command.** "Take note, buy milk"
consistently transcribed as "note, buy milk" — the recording start had a
small timing race against when you actually started speaking. Fixed with
a continuously-recording ring buffer that gets prepended to every capture
as a safety margin.

**Making a paid AI tier that can't overspend.** The escalation tier
(used rarely, only when the free tiers can't confidently handle
something) tracks every call's token cost against a persistent, hard
spend cap stored locally — once hit, Shadow simply stops using the paid
tier rather than risk running past a limit I set.

## Tech stack

Python (FastAPI, WebSockets, faster-whisper, a local neural TTS engine),
Kotlin/Android, a multi-provider LLM routing layer, Google Calendar API,
and a private mesh network (Tailscale) linking the devices.

## Status

Actively developed, personal daily-use project and portfolio piece.
Full source available privately on request.
