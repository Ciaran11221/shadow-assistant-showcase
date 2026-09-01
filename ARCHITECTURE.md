# Architecture

The parts of the design worth explaining, and why they're built this way.

---

## Thin clients, thick core

Every device runs the same thin client: capture audio, detect the wake phrase,
send text to Core, play what comes back. Nothing about *what Shadow can do*
lives on a device.

Core owns skill routing, device arbitration, conversation state and voice
synthesis. A new capability appears on the phone and the desktop simultaneously,
with no app update.

The cost is that a device is useless when Core is unreachable. That's handled
explicitly rather than ignored — see [offline behaviour](#when-the-desktop-is-asleep).

---

## Skills register themselves

The original design had a router that knew every skill by name. Adding one
meant editing the router, the dispatch table, the AI prompt and the fallback
chain — four places, and a chance to forget one.

Now a skill is a single file exposing four functions, one of which describes
itself:

- what it's called
- one sentence on what it does, fed into the AI router's prompt
- its parameters, described in prose

The registry discovers every module in the skills package at import, verifies it
exposes the required interface, and orders them by an optional priority. Nothing
central knows any skill by name.

**Adding a capability is adding a file.** The router's prompt, the fallback
chain and the parameter extraction all update themselves.

A skill may optionally declare two more hooks:

- **`validate_params`** — "these parameters aren't mine". The AI router sorts
  confidently but not always correctly; without this, a wrongly-picked skill
  answers anyway, usually with a clarifying question about the wrong subject.
- **`claim_continuation`** — "this utterance isn't part of my open exchange".
  Conversations survive interruption, so a news briefing left open must not
  swallow an unrelated request as a headline choice.

Both are optional. A skill that doesn't declare them gets the old behaviour.
That's what keeps the interface small enough that adding a skill stays a
one-file job.

---

## Routing a request

1. **Is a skill mid-conversation with this device?** If so, and if it claims the
   utterance, it handles it.
2. **AI routing.** The request plus every skill's self-description goes to a fast
   model, which returns a skill name and extracted parameters.
3. **Regex fallback.** Each skill in priority order gets a chance to match the
   raw text.
4. **Conversational reply**, in character, rather than an error.

The tiers exist because they fail differently. AI routing handles phrasing
nobody anticipated. Regex handles the AI being unavailable, over budget, or
confidently wrong about a phrase with an unambiguous form.

### Model tiers

Four providers in cost order — three free in sequence, then a paid one — with
a budget guard that stops the paid tier when a spending cap is hit. Below the
tiers is regex matching, so the assistant degrades rather than breaking.

Requests are also classified as *factual* or *conversational*. Factual requests
drop the persona prompt and run at low temperature; they need to be right, not
characterful.

---

## Two-layer listening

Running a full speech recogniser continuously to catch two words is enormously
wasteful on a phone battery, which is what the first version did.

**Layer one** is a small keyword-spotting model doing nothing but scoring audio
against one phrase. **Layer two** — full recognition — starts only after layer
one fires, and stops when the exchange ends.

Only one layer can hold the microphone, so the handover is explicit in both
directions.

The wake phrase itself is open-vocabulary: it's a line of text in a file, not a
trained model, so changing it means regenerating one small file rather than
retraining anything or visiting a vendor console.

**Exception:** on a Bluetooth headset, layer one is replaced by full recognition,
because the keyword model can't work with that microphone. This was measured,
not assumed — see [case study 1](CASE-STUDIES.md#1-the-log-that-only-recorded-successes).

---

## Which device answers

Both devices hear "Hey Shadow". Exactly one should answer.

A claim protocol in Core resolves it:

- Claims arriving within a short **pooling window** are collected rather than
  raced, and the highest-priority device wins.
- A **cooldown** after a granted claim stops the same utterance being re-heard
  as a second wake.
- Naming a device — "Hey Phone", "Hey Desk" — is **explicit** and bypasses
  arbitration entirely, which also makes it noticeably faster to answer.
- Wearing a Bluetooth headset counts as naming that device. If the buds are in,
  you're nearest the phone by definition.
- A device that doesn't hold the floor has its commands rejected, so a
  mis-detection on one device can't hijack an exchange on another.

Claims expire on a timer, so a client that dies mid-exchange doesn't lock the
system.

---

## When the desktop is asleep

Some things only the desktop can do. It's also usually asleep.

1. The command is stored on the phone.
2. A Wake-on-LAN packet goes out — **twice, by two different routes.**
3. **You're acknowledged immediately** — "Loading remote device" — rather than
   standing waiting on a boot.
4. A background drainer polls until Core answers, then runs what's queued and
   reports back.

Commands are deleted only once Core has actually handled them. If the wake
fails, the app is killed, or the phone reboots, they're still on disk and get
retried.

**The interesting limitation, and how it got solved.** A Wake-on-LAN packet is
a LAN broadcast — it can't route over the internet, and it can't route over the
VPN either, because a sleeping machine's own VPN client is asleep along with
it. That's physics, not a missing setting, so it holds regardless of how the
network is configured.

The fix isn't in the phone or the desktop at all. It's a third, tiny device: an
old phone left permanently awake on the home network, running a stripped-down
relay with exactly one job — accept "wake the desktop" over the VPN, and put
the broadcast on the LAN it's actually sitting on. The main phone fires both
routes on every attempt: the direct packet, which is instant when you're home
and needs nothing else running, and the relay, which is the only one that can
possibly work from a phone on mobile data three hundred miles away.

The relay is deliberately built as its own standalone thing — no shared code,
no shared config with the rest of the system. It exists specifically for the
moment the main system is asleep or broken, so it can't depend on anything
that might be down at that moment.

Verified, not assumed: phone on mobile data, Wi-Fi off, desktop asleep, a
command queued. The direct packet failed exactly as expected. The relay
carried the request over the VPN, broadcast it locally, and the desktop came
up.

---

## Latency

Perceived speed matters more than actual speed, and the fixes are different.

**Pre-rendered clips.** Fixed phrases are rendered ahead of time through the
real voice pipeline and bundled with the app. "Yes?" plays instantly instead of
waiting on synthesis. A phrase with no clip falls through to synthesis
automatically, so reworded text never produces a stale clip confidently saying
the wrong thing.

**Interim messages.** A skill returns one reply, so anything slow — a feed
fetch, an AI call — is silence from the user's side. Several seconds of nothing
is indistinguishable from being ignored, and the instinct is to repeat
yourself, which makes it worse. Slow skills now emit a short line while they
work. Those lines are deliberately fixed and short so they can be clips
themselves, rather than needing the synthesis they exist to cover.

**Sentence-by-sentence synthesis**, streamed as binary frames, so playback
starts before the whole reply is rendered.

**Ordering that isn't obvious.** After an exchange, listening restarts *before*
the network cleanup. Releasing the claim first left the phone deaf for five and
a half seconds after every exchange — long enough that speaking again in that
window was simply never heard, which felt like the wake word being unreliable.

---

## Conversation state

Skills can hold an exchange open — a news briefing waiting for you to name a
headline, a calendar entry missing a date.

The subtlety is what happens when you interrupt. Cutting the assistant off used
to discard the exchange, on the reasoning that interrupting meant walking away.
But interrupting a briefing to say "the third one" is the *opposite* of walking
away, and discarding the list left nothing to resolve that against.

So exchanges now survive interruption, and the risk that creates — an unrelated
request being read as an answer — is handled by the `claim_continuation` hook.
Skills decide for themselves whether an utterance belongs to them. "Weather in
Cork" mid-briefing goes back to the router; "what about cyberpunk game news"
stays with the news skill.

Stories already read out are remembered per device for six hours, so asking
twice moves the conversation forward instead of replaying it — with an escape,
because "say that again" must not be suppressed by a repeat filter.

---

## Privacy

- Speech recognition runs **on-device** on both platforms. Captured audio is
  never sent to a cloud recogniser.
- Only the resulting **text** reaches Core, over a private VPN mesh — never the
  open internet.
- Clients **authenticate on connect** with a shared secret. The mesh alone was
  once treated as sufficient; it isn't, because it makes every device on the
  network a trusted one. Identity asserted by the caller is not identity.
- Personal data — location, interests, preferences, credentials — is local and
  git-ignored.
- Nothing is inferred and quietly stored. Everything Shadow knows about you was
  stated on purpose, can be listed back, and can be removed. A profile that
  grows on its own is one you can't reason about.

**Two different questions get two different gates.** "Is this roughly the
same person talking?" and "is this person allowed to change something
permanent?" don't need the same certainty, and answering both with one
threshold causes real problems — loosen it and a stranger gets through,
tighten it and the real user gets locked out on an off day, and no single
number avoids both. So they're separate: an ongoing, low-stakes voice signal
that informs conversation but never blocks it, and a short spoken code
required only at the moment something permanent is actually written or
deleted — checked the same way regardless of who the voice sounds like.

**Short-term memory expires on purpose.** What's said in a session is kept
for the day, searchable, then gone — nothing persists past that without a
deliberate save. Saving something permanently, or deleting it, goes through
the confirm-then-code gate above; nothing reaches long-term storage as a
side effect of ordinary conversation.

---

## How this gets verified

A voice assistant is awkward to test: the interesting failures involve a
microphone, a room, and a person talking. Three layers cover that.

**Unit suites, offline.** Around 1,500 assertions across 36 suites, no network
and no audio hardware required, running in about a minute. Each suite runs in
its own process — one of them deliberately swaps out the routing modules to
test the router in isolation, which is fine alone and poisonous to anything
importing them for real afterwards.

**Audio injection instead of a microphone.** Recorded and synthesised speech is
fed directly through the real detection functions — the same code path a live
microphone reaches, minus the microphone. That makes barge-in, wake detection,
and echo settling reproducible rather than something you evaluate by talking at
a laptop and forming an impression. It also surfaced a genuinely awkward
finding: a fix that boosts quiet speech was also boosting the *test* audio, so
"how does this behave with a quiet voice" was a question the test suite was
structurally unable to answer until the boost was explicitly bypassed.

**Adversarial harnesses on the risky surfaces.** Twenty-four of them, aimed at
places where correctness is hard to eyeball: misspelled and ambiguous natural
dates, cross-skill phrase collisions, scope leaks between isolated skills, and
above all the irreversible delete paths, which are fuzzed with randomised
inputs against an explicit invariant rather than hand-picked examples. The
distinction that drives all of it: most of this system's worst case is a wrong
answer you can correct, and a small number of paths have a worst case of data
that no longer exists anywhere.

What this does *not* cover is stated plainly in the private repo's roadmap:
work that is built, unit-tested, and installed but never yet exercised by hand
in real use is tracked as a standing debt rather than counted as finished.
