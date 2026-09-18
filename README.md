# Shadow

A private operating layer over a Windows desktop and an Android phone. No cloud
service in the middle.

It supervises its own startup, runs extensions as separate OS processes, gates
every capability behind a switch, decides which device answers when both hear
you, and sends anything private to a model on the machine instead of an API.

It started as a voice assistant. Voice is one of six ways in now.

    100,700 lines of Python and Kotlin      25 capability modules
    10,704 test assertions, 101 suites      226 merged pull requests
    total AI spend, all time: under 1.50 euro

This repo is the write-up, not the source.

## Start here

**[CASE-STUDIES.md](CASE-STUDIES.md)** is the part worth your time. Twenty-two
problems that took real work, each one written the same way: what it looked
like, what I assumed, what the evidence said, what I changed.

Three of them I fixed by reasoning about it, watched the fix make things worse,
and had to admit I was guessing. That's most of what I learned building this.

If you only read one, read number 19. I planted a bug in my own code to test a
tool I had not built yet, and the model scored 0 out of 19 while answering the
same question correctly when I asked it without the code attached.

## What makes it a layer and not an assistant

**Supervises its own startup.** Four checkpoints and a soak window, so a crash
loop cannot keep marking itself healthy. After three failed boots it stops
relaunching itself. A manual start is never blocked.

**Runs extensions as separate processes.** A third-party module gets its own OS
process, JSON in and out, and only the fields its manifest asks for. You can add
one without trusting it.

**Switches anything off while running.** Any module, or one function inside a
module, off at runtime with no restart.

**Decides which device answers.** Both machines hear the wake phrase. Claims
pool for 200ms and resolve by priority, which replaced a race whose winner was
decided by whichever detector happened to be faster.

**Routes private content on-device.** Five AI tiers with a spend cap checked
before every paid call. The local tier is not a fallback. Nothing can reach it
by falling through the others, and it refuses rather than sending your screen
contents to a cloud provider.

**Refuses a file it does not recognise.** Saved sessions carry a schema number
and get migrated on read. An unknown version is refused outright instead of
half-read.

**Sleeps properly.** One trigger closes the audio stream and unloads the models.
It does not mute a microphone and call that off.

## What it does

Say "Hey Shadow" and it answers, from whichever device is nearest, in a voice
that never leaves the machine. Or type it. Or press a hotkey. Or share text into
it from another Android app. Or let it notice something and speak first.

- **News** filtered against your interests, drillable by topic or position, with
  anything you already heard suppressed for six hours.
- **Notes** with an "anything else?" loop, so a dropped word does not cost you
  the note.
- **Calendar** through the Google API, which asks for a date and time instead of
  guessing.
- **Body vitals** flagged against your own history, never a population
  threshold. It says "not enough data" instead of guessing, and it never
  diagnoses.
- **A journal voice cannot reach.** Not discouraged from reaching. The module
  opts out of routing entirely, so no phrasing of any sentence gets to it.
- **Wakes the desktop when it is asleep** and runs what you asked once it is up.
  Works from mobile data, not just home wifi, because a small relay device on
  the home network does the last local hop.
- **Interruptible.** Talk over it, it stops, keeps the context, takes what you
  said next as the new request.
- **Conversation mode** drops the wake word entirely. Writes and deletes still
  need a spoken prefix, so ordinary chat cannot trigger one by accident.

## Why it exists

I wanted an assistant that did not ship my kitchen conversations to a third
party. I also wanted a project big enough that the problems would be systems
problems, not tutorial problems. It delivered on the second more than I
expected.

## Architecture

```
   Android app              Desktop client
   (Kotlin)                 (Python)
   wake word                wake word
   capture                  capture
   playback                 playback
       |                        |
       +--- WebSocket over -----+
            a private VPN mesh
                    |
                  Core
                (FastAPI)
              arbitration
                routing
             speech synth
                    |
                 Skills
          self-registering, 25 of them
```

Core owns everything shared: which device has the floor, how a request maps to a
module, and voice synthesis. Clients just capture audio, send text, play what
comes back. That is why the phone and the desktop behave identically, and why a
new module works on both the moment it exists.

[ARCHITECTURE.md](ARCHITECTURE.md) has the parts worth explaining properly.

## A few things I would point at

**A security hole I found by attacking my own system.** Core's WebSocket
accepted anything that could reach the port. The field naming the caller was
supplied by the caller, so it proved nothing. Anything on the network could
change settings, attach extensions, and delete arbitrary files through a path
traversal. Nothing external found this. It needed someone to sit down and ask
what the connection actually checked, and the answer was nothing.

The two gates guarding sensitive paths now fail in opposite directions on
purpose. The spoken code protecting deletions fails closed. The connection token
fails open. Making them symmetrical would have been wrong: a locked setting is
an inconvenience, but a connection layer that fails closed before you have
provisioned it locks out every device at once, including the one you would fix
it with.

**A duplicate guard that ate real data.** Most of this system's worst case is a
wrong answer. Two paths destroy data instead, so those get randomised harnesses
thrown at a one-line invariant: after equals before minus exported. First run
found a real bug. Deletion matched entries by value, so exporting one of two
identical entries destroyed both, with no second copy anywhere.

**A free model that invented a pattern in noise.** One feature reads back a
month of journal entries and reports patterns, which is the kind of task where a
wrong answer looks exactly like a right one. So I built an eval that plants a
known pattern and checks the model finds it, plus a pure-noise case it has to
report as having nothing. The free tier scored 3/5 then 1/5 on identical input,
and on the worse run claimed a pattern in the noise. Inventing a trend in
someone's private journal is the failure that actually matters. That feature
runs on a pinned paid model now. Still open: every other call site makes the
same free-tier assumption and I have not checked them this way.

**A background agent that cannot do anything but talk.** It watches what is on
screen and decides whether anything is worth saying. I deleted an earlier module
for doing the opposite, judging everything that passed through it and quietly
writing down what it decided to keep. So the difference here is built in rather
than promised: the whole audit surface is the import list, and it imports two
things, a local model call and the speech path. There is no function it could
call to write anything.

**A stall that fixes itself.** The desktop's audio input can silently stop
delivering data and nothing about that looks like an error. The process just
goes quiet. Listening now tracks the gap since the last chunk and reopens the
stream itself. It has since caught a real nineteen second stall during normal
use, unprompted.

**Saying no to my own idea.** Mid-build I proposed a bigger version of a feature
to myself: three tools handing off between them, each referencing the others'
prior work. I reasoned through what it would cost before writing any of it, and
shipped the concrete entry point that was actually needed instead. The two open
decisions are written down. The rest is deliberately unbuilt.

## Built with

    Android      Kotlin, coroutines, foreground service, SpeechRecognizer,
                 AudioRecord, Bluetooth audio routing, OkHttp
    Desktop      Python, FastAPI, WebSockets, Whisper, and three offline TTS
                 engines. No cloud speech synthesis anywhere.
    Wake word    sherpa-onnx keyword spotting, on-device, open vocabulary
    Networking   WebSocket over a private VPN mesh, Wake-on-LAN with a relay
                 device for off-network wake
    AI           four cloud tiers with cost control, plus a local on-device
                 tier that private content is routed to exclusively
    Voice        a Piper model I trained on extracted real dialogue

## Status

In daily use, which is why the problems in the case studies are the ones they
are. Most of them only show up when you rely on something rather than demo it.

The private repo keeps a "critiques and roadmap" section listing what is
weakest and what is built but not yet proven in real use. It is usually the more
interesting document.

Source is private. Happy to walk through any part of it.
