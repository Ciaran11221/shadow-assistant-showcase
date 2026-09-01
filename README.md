# Shadow

A private voice assistant that runs on my own hardware — an Android phone and a
Windows desktop working as one system, with no cloud service in the middle.

This repository is a write-up. The implementation is private; what's here is
the design, the reasoning behind it, and a few debugging stories that show how
the harder problems were actually solved.

---

## What it does

Say "Hey Shadow" and it answers — from whichever device is nearest, in a
consistent synthesised voice, without sending your speech to anyone.

- **News** — briefings filtered against your stated interests and tolerance,
  drillable by topic or position ("the third one"), with stories you've already
  heard suppressed for six hours so asking twice moves forward instead of
  replaying.
- **Notes** — with a multi-turn "anything else to add?" loop, so a dropped word
  never costs you the note, and automatic date-based filing.
- **Calendar** — scheduling through the Google Calendar API, which asks for a
  date and time rather than guessing when you didn't give one.
- **Weather, music, contacts, tasks, personas, preferences** — each a self-contained
  skill, added by dropping in one file.
- **Body vitals** — flags a reading against your own history rather than a
  hardcoded threshold, and refuses to diagnose.
- **A private journal** that voice cannot reach at all. Not "discouraged from"
  reaching — structurally unable to: the module opts out of routing entirely,
  so no phrasing of any sentence can reach it. Entries are categorised into a
  private spreadsheet and the local copies deleted only once the write is
  confirmed.
- **Wakes the desktop when it's asleep**, queues what you asked for, and runs it
  once the machine is up — so a sleeping PC costs you a few seconds, not a lost
  request. Works from anywhere, not just the home network — a magic packet
  can't route over the internet, so a small always-on relay device sits on the
  home LAN and does that last local hop on request.
- **Interruptible.** Talk over it and it stops mid-sentence, keeps the context,
  and takes whatever you say next as the new request.

---

## Why it exists

Two reasons, honestly. I wanted an assistant that didn't ship my kitchen
conversations to a third party. And I wanted a project big enough that the
interesting problems would be *systems* problems — concurrency, device
coordination, latency, hardware that doesn't behave as documented — rather than
tutorial problems.

It has delivered on the second more than I expected.

---

## Architecture

```
   ┌────────────────┐         ┌────────────────┐
   │  Android app   │         │ Desktop client │
   │  (Kotlin)      │         │  (Python)      │
   │                │         │                │
   │  wake word     │         │  wake word     │
   │  capture       │         │  capture       │
   │  playback      │         │  playback      │
   └───────┬────────┘         └───────┬────────┘
           │      WebSocket over      │
           │    a private VPN mesh    │
           └────────────┬─────────────┘
                        │
                ┌───────▼────────┐
                │      Core      │
                │   (FastAPI)    │
                │                │
                │  arbitration   │
                │  routing       │
                │  speech synth  │
                └───────┬────────┘
                        │
                ┌───────▼────────┐
                │     Skills     │
                │  self-registering
                │  news · weather · calendar · notes
                │  music · contacts · tasks · vitals
                │  journal · settings · personas
                └────────────────┘
```

**Core owns everything shared** — which device has the floor, how a request maps
to a skill, and voice synthesis. Clients are deliberately thin: capture audio,
send text, play what comes back.

That split is what makes the desktop and the phone behave identically. It also
means a new skill is available on every device the moment it exists, with no
client change.

See [ARCHITECTURE.md](ARCHITECTURE.md) for the parts worth explaining properly.

---

## Engineering highlights

**Skills self-register.** Adding a capability means adding one file. The router
discovers it, reads its self-description, and includes it in AI routing —
nothing central knows any skill by name. Before this refactor, a new skill meant
edits in four places and a chance to forget one.

**Multi-tier AI routing with a budget guard.** Requests fall through a fast free
model, then a second, then a paid one, degrading to regex matching if all three
are unavailable. Cost is capped and the assistant keeps working when it's
exhausted — just less cleverly.

**Two-layer listening.** A small keyword-spotting model runs continuously and
does nothing but score audio against one phrase. Full speech recognition only
starts once that fires. This replaced an always-on recogniser and cut idle
battery drain substantially.

**Wake-on-LAN with an offline queue, working from anywhere.** Commands issued
while the desktop is asleep are stored on the phone, a wake attempt goes out on
two independent routes, and a background drainer runs the queue when the
machine answers. A magic packet is a LAN broadcast, so it can't reach a
sleeping machine over the internet — a small relay device left running on the
home network exists purely to carry the request that last local step, which is
what makes this work from mobile data and not only from home Wi-Fi. See
[case study 6](CASE-STUDIES.md#6-the-magic-packet-that-couldnt-leave-the-house)
for how that was found and fixed. Acknowledged immediately either way, so you
never stand waiting on a boot.

**Latency as a feature.** Fixed phrases are pre-rendered as audio clips in the
real voice, so "Yes?" is instant rather than synthesised. Slow skills speak an
interim line while they work, because several seconds of silence is
indistinguishable from being ignored. Synthesis is streamed sentence by
sentence rather than rendered whole.

**Device arbitration.** Both devices hear "Hey Shadow". A claim protocol with a
pooling window, priority ordering, and a cooldown ensures exactly one answers.
Naming a device explicitly ("Hey Phone") bypasses arbitration entirely, and
wearing a Bluetooth headset counts as naming it.

**Nothing permanent without asking twice.** Saving a new fact — a name, a
contact, a preference — means the words get confirmed back before anything
sticks. Deleting something already saved needs that same confirmation and a
short spoken code besides. The two are protected differently on purpose:
getting an add wrong is clutter you can remove; getting a delete wrong
destroys something.

**A boundary I only found by attacking my own system.** Core's WebSocket
originally accepted any client that could reach its port. The `device` field
identifying the caller was supplied *by the caller* and proved nothing, which
meant anything on the network could change settings, attach extensions, and —
before a filename guard landed the same day — delete arbitrary files through a
path traversal. Closed with a shared secret the phone presents on connect. The
part worth saying out loud is that nothing external found this: it needed
someone to sit down and ask what the connection actually verified, and the
answer was nothing at all.

Two gates guard the sensitive paths and they deliberately fail in *opposite*
directions. The spoken code protecting deletions fails closed — unconfigured
means refuse. The connection token fails open — unconfigured means allow.
Symmetry would have been the wrong instinct: a locked-out setting is an
inconvenience, while a connection layer that fails closed before you've
provisioned it locks every device out of the system at once, including the one
you'd use to fix it.

**Sensitive screens share one lock.** What began as three separately-built PIN
systems — one for settings, one for the coding tool, one for health data, each
reasoned about on its own — is now a single hardware-backed prompt (fingerprint
or the phone's own lock-screen credential) reused across seven screens. The
tradeoff is stated rather than hidden: any fingerprint enrolled on the phone is
trusted by all of them, on the explicit judgement that the physical device is
the real boundary.

**A stall that fixes itself.** The desktop's audio input can silently stop
delivering data — a driver hiccup, a device change — and nothing about that
looks like an error, the process just goes quiet. Listening now tracks the
gap since the last chunk arrived and reopens the input stream itself once
it's been stalled too long, rather than needing a restart. It's since caught
and recovered from a real nineteen-second stall during normal use, unprompted.

**Measuring a model instead of trusting it.** One feature reads back a month of
journal entries and reports patterns — the kind of task where a wrong answer is
indistinguishable from a right one unless you check. So I built an evaluation
that plants a known pattern and checks the model finds it, alongside a
pure-noise case that it must report as having no pattern. The free tier scored
3/5 and then 1/5 on byte-identical input across consecutive runs, and on the
worse run claimed a pattern in the noise — the one failure that actually
matters, since inventing a trend in someone's private journal is worse than
declining to answer. That feature now runs on a pinned paid model. The open
question I haven't closed: every other call site makes the same free-tier
assumption and hasn't been checked this way.

**Randomised testing on the paths that can't be undone.** Most of this system's
worst case is a wrong answer. Two paths destroy data instead, and those get
adversarial harnesses rather than examples: randomised stores thrown at the
export-then-delete path against a one-line invariant (*after = before −
exported*), plus an injected mid-export outage to prove nothing is deleted when
the write fails. The very first run found a real bug — deletion matched entries
by value, so exporting one of two identical entries destroyed both, with no
second copy anywhere. Around 1,500 assertions across 36 suites run offline in
under a minute; twenty-four separate harnesses fuzz the riskier surfaces.

**Breaking the voice pipeline on purpose, so it doesn't break by accident.**
A real recording gets pushed through independent, worsening distortions —
slurred, quieted, buried in noise — and re-run through the actual
recognition pipeline until it reliably fails. That failure point is a number,
tracked over time, so a fix can be proven rather than felt. It runs itself
now: real audio from normal use gets swept automatically in the background,
off the assistant's own thread, so testing never costs it any responsiveness.

---

## What I'd point at in an interview

Not the feature list — the [case studies](CASE-STUDIES.md).

Three times on this project I fixed something by reasoning about it, watched the
fix make things worse, and had to conclude I'd been guessing. The response was
to stop guessing: build the instrumentation that would settle the question, then
read it.

That shift — from "this should work" to "the log says" — is the most useful
thing I've taken from building this, and it's what those write-ups are about.

---

## Built with

**Android** — Kotlin, coroutines, foreground service, `SpeechRecognizer`,
`AudioRecord`, Bluetooth audio routing, OkHttp
**Desktop** — Python, FastAPI, WebSockets, Whisper, and three interchangeable
offline TTS engines (Piper with a custom effects chain, Kokoro, XTTS) — no cloud
speech synthesis anywhere
**Wake word** — sherpa-onnx keyword spotting, on-device, open-vocabulary
**Networking** — WebSocket over a private VPN mesh, Wake-on-LAN with a relay
device for off-network wake
**AI** — tiered routing across four providers with cost control

---

## Status

Actively developed, and in daily use — which is why the problems in the case
studies are the ones they are. Most of them only surface when you rely on
something every day rather than demoing it.

Roughly 55,000 lines of Python and Kotlin, around 50 self-registering skill
modules, and a test suite of about 1,500 assertions that runs offline. The
private repository keeps a "critiques and roadmap" section listing what's
weakest and what's built but not yet verified in real use — it's maintained
in the same spirit as the case studies here, and it's usually the more
interesting document.

Full source is in a private repository. Happy to share access or walk through
any part of it — just ask.
