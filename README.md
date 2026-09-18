# Shadow

A private personal operating layer over a Windows desktop and an Android
phone, working as one machine with no cloud service in the middle. It
supervises its own startup, isolates untrusted extensions in their own
processes, gates every capability behind a switch, arbitrates which device
owns the microphone, and schedules work between a local model and a metered
cloud one by how sensitive the content is.

It started as a voice assistant. Voice is now one of six ways in — wake word,
typed chat, a global hotkey, phone screens, Android share and clipboard
intents, and ambient triggers that nobody initiated at all.

100,700 lines of Python and Kotlin across 25 capability modules discovered at
runtime. 10,704 test assertions across 101 CI-gated suites, 226 merged pull
requests. Running cost to date: under €1.50 — the two paid tiers exist, are
budget-capped, and have never actually been needed.

This repository is a write-up. The implementation is private; what's here is
the design, the reasoning behind it, and a few debugging stories that show how
the harder problems were actually solved.

---

## The mechanisms, before the features

The features below are the visible half. The half that took the work is
underneath, and it is the part that stopped this being a voice assistant:

    process supervision   boot checkpoints and a soak window, so a crash
                          loop cannot keep stamping itself healthy
    extension isolation   third-party modules run as separate OS processes,
                          JSON only, reading exactly the fields their
                          manifest declares — added without being trusted
    capability gating      every module, and individual functions inside
                          one, switchable at runtime with no restart
    device arbitration    both machines hear the wake phrase; a pooling
                          window and priority order resolve exactly one
    compute scheduling    private content is routed to an on-device model
                          that cannot reach a cloud provider; GPU memory is
                          released after each ambient check and skipped
                          entirely while a game has focus
    versioned state       stored sessions carry a schema integer, are
                          migrated on read, and are REFUSED rather than
                          half-read when the version is unknown
    power management      sleep closes the audio stream and unloads the
                          models rather than muting a microphone

## What it does

Say "Hey Shadow" and it answers — from whichever device is nearest, in a
consistent synthesised voice, without sending your speech to anyone. Or type
it, or press a hotkey, or share text into it from another Android app, or let
it notice something and speak first.

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
- **Conversation mode** — a toggle that drops the wake-word requirement
  entirely for back-and-forth talk, holding a short-term memory of that
  session so "tell me more about that" resolves against what was just said.
  A write or delete still can't slip through as ordinary chat: those need an
  explicit spoken prefix, so casual conversation is structurally unable to
  trigger a command.
- **Sleeps without closing.** One trigger drops it to nothing-is-listening and
  actually releases what it was holding — the audio stream closed, the models
  unloaded — rather than muting a microphone that stays open. Waking back up is
  deliberately a second act, because an accidental off costs you nothing and an
  accidental on opens a microphone nobody asked for.
- **Notices, on a local model only.** It can watch what's on screen and speak
  unprompted — but that path is wired to an on-device model and physically
  cannot reach a cloud provider, because a window title carries email subjects
  and document names. It can only ever *say* something; it has no write path at
  all.

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

**A version policy that refuses rather than guesses, with a migration lane in
front of it.** Stored session files carry a schema integer. On an unrecognised
version a temporary file is discarded and a saved one is refused and left on
disk — never half-read, never silently upgraded. Known versions are migrated on
read, in memory, so opening a file never rewrites it. Each migration declares
the version it produces and the loader walks the chain, so a missing hop fails
loudly instead of accepting a half-upgraded file as current. See
[case study 12](CASE-STUDIES.md#12-a-version-bump-that-would-have-made-real-work-unopenable)
for the saved file this protects, and the untested branch found while reviewing it.

**A self-healing record that never acts on its own.** The desktop app records
which boots actually worked — four startup checkpoints plus a soak window, so a
crash loop can't keep stamping itself as healthy — and keeps a small rolling
snapshot of configuration in front of the encrypted backup. It diagnoses and
reports; it restores nothing without someone pressing a button, and it never
puts source files back, because git already holds every version of those.
Live-verified against a running instance afterward — real process kills, a
byte-hash-checked restore, the actual crash-loop dialog on screen — with one
check honestly left open rather than rounded up. See
[case study 13](CASE-STUDIES.md#13-the-one-word-default-that-would-have-left-the-machine-unstartable)
for the threading default that would have made a restart leave the machine down,
the two faults the same review found in my own plan, and what the live run
found (and didn't) afterward.

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
second copy anywhere. 10,704 assertions across 101 suites run offline in
under a minute; thirty-three separate harnesses fuzz and benchmark the
riskier surfaces.

**Breaking the voice pipeline on purpose, so it doesn't break by accident.**
A real recording gets pushed through independent, worsening distortions —
slurred, quieted, buried in noise — and re-run through the actual
recognition pipeline until it reliably fails. That failure point is a number,
tracked over time, so a fix can be proven rather than felt. It runs itself
now: real audio from normal use gets swept automatically in the background,
off the assistant's own thread, so testing never costs it any responsiveness.

**A background agent that is structurally unable to do anything but talk.**
The newest piece can speak without being asked — it watches active-window
state and decides whether anything is worth saying. That is the same shape as
a module I deleted earlier in the project for judging everything that passed
through it and silently *writing* what it decided was worth keeping. So the
difference is built in rather than asserted: the module's entire import list
is the audit surface, and it imports exactly two things — a local-only model
call and the speech path. No notes, no journal, no contacts, no memory
module; there is no function it could call to write anything. Three further
constraints are enforced the same way: it never reaches a cloud provider
(window titles carry email subjects), it never speaks over a live session,
and a file-backed brake can stop it from a phone, because a stop command
delivered over the same connection the loop is saturating is not a brake.

**Training a voice from real recordings, and three instruments that lied.**
The default voice is now a model I trained myself on extracted game
dialogue, rather than an off-the-shelf one. Getting there meant building
measurements for "does this sound right", and the useful part is that the
first three were each confidently wrong in a different way: one took a single
sample of a process that turned out to be non-deterministic, one put a binary
cut through a continuous property, and one used a test case structurally
incapable of showing the effect it was built to detect. A one-variable
control settled in a single run what a week of instrumentation had not, and
a later listening test had to be rebuilt after a catch pair proved its design
couldn't tell the model apart from the slot it was played in. The conclusion
I actually shipped was to stop trusting the metric past a certain precision
and use my own ears, which is not the answer I wanted.

**Saying no to my own idea.** Mid-build on a feature, I proposed a bigger
version of it to myself: three tools able to hand off between them and
reference each other's prior work automatically. Before writing any of it,
I reasoned through what it would actually cost — a live search integration
versus a cheaper but staler model-only guess, a fixed pipeline versus a
general graph where anything can call anything — and concluded the
ambitious version wasn't the right thing to build first. I shipped the
concrete entry point that was actually needed, wrote the two open decisions
down for later, and left the rest deliberately unbuilt rather than build it
because I could.

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
speech synthesis anywhere, and pre-rendered clips for fixed lines so the common
replies cost nothing to say
**Wake word** — sherpa-onnx keyword spotting, on-device, open-vocabulary
**Networking** — WebSocket over a private VPN mesh, Wake-on-LAN with a relay
device for off-network wake
**AI** — tiered routing across four cloud providers with cost control, plus a
local on-device tier (ollama) that private content is routed to exclusively
**Voice training** — a Piper model fine-tuned on extracted real dialogue

---

## Status

Actively developed, and in daily use — which is why the problems in the case
studies are the ones they are. Most of them only surface when you rely on
something every day rather than demoing it.

100,700 lines of Python and Kotlin, 25 self-registering capability modules,
and a test suite of 10,704 assertions across 101 CI-gated suites that runs
offline.
The private repository keeps a "critiques and roadmap" section listing what's
weakest and what's built but not yet verified in real use — it's maintained
in the same spirit as the case studies here, and it's usually the more
interesting document.

Full source is in a private repository. Happy to share access or walk through
any part of it — just ask.
