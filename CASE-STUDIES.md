# Case studies

Problems from this project that took real work — seven recent ones in
detail, six earlier ones briefly. Each follows the same shape: what it looked like,
what I assumed, what the evidence actually said, and what I changed.

The pattern across all of them is the same, and it's the point of this
document.

---

## 1. The log that only recorded successes

**Symptom.** The wake phrase was unreliable. I'd say "Hey Phone" three or four
times before it answered, sometimes shouting.

**What I assumed.** The detection threshold was too high. Obvious fix: lower it.

**Why that was wrong.** I checked the log first and found two clean detections
in the window I'd tested. That looked like the wake word working fine — so I
concluded the problem was elsewhere.

It wasn't. A failed detection writes *nothing*. The keyword spotter reports a
match or stays silent; there's no confidence score in its result type. So five
failures followed by one success and a single success on the first try produce
byte-identical logs. I'd read a log that was structurally incapable of showing
me the thing I was investigating, and drawn a confident conclusion from it.

**What I built.** Since the model wouldn't tell me about near-misses, I measured
the audio myself. The capture loop now tracks a running noise floor and logs
every burst of speech that ended without a match:

```
Speech heard but nothing matched: peak=0.1282 floor=0.0002 length=500ms
Speech heard but nothing matched: peak=0.1338 floor=0.0011 length=500ms
```

**What it showed.** Twenty-five bursts. Peaks of 0.08–0.17 against a floor
around 0.001 — a signal a hundred times above the noise, in a quiet room, for
half a second each. Zero detections.

That killed the threshold theory outright. My voice was arriving in excellent
condition and the model was matching nothing.

**The corroboration.** In the same period the desktop, listening on its own
microphone, transcribed the same speech correctly seven times:

```
wake-check heard: 'hey phone.'  -> Wake phrase for the phone -- staying quiet
```

Two independent systems, same room, same voice. One understood it every time;
the other never did. The difference was the microphone — a Bluetooth headset
whose narrowband capture leaves the upper half of every audio frame empty, which
is exactly the input a model trained on 16kHz speech cannot use.

**What I changed.** On a headset, idle listening now goes through full speech
recognition instead of the keyword model — which handles that microphone
without difficulty. The keyword model stays for the phone's own mic, where it's
cheap enough to run all day.

**The lesson.** Absence of evidence in a log is not evidence of absence. Before
trusting a log to answer a question, check that it's capable of recording the
answer.

---

## 2. Three speculative fixes that each broke working code

Over a few sessions I made three changes on plausible reasoning, without
evidence. All three broke functionality that had been working.

**`recognizer.cancel()` to free the microphone.** The theory: the speech
recogniser was holding the mic and blocking wake detection during playback.
Reasonable. The log later disproved it — the detector restarts happily
mid-reply. Meanwhile cancelling left the recogniser unable to hear anything
afterwards: every wake produced "Yes?" and then silence.

**Shortening the endpointing timeouts.** The theory: ending the turn faster
would make it feel snappier. What actually happens is there's a natural pause
between the acknowledgement finishing and you starting to speak, and the
recogniser counted that as the end of the turn. Every command came back empty.

**Switching the audio source to `VOICE_COMMUNICATION`.** The theory: its echo
cancellation would help detect "stop" over the speaker. That processing is
built for phone calls — heavy suppression and gain control — and it degrades
exactly the signal a keyword model needs. Stop detection got worse, never
better.

**What I changed.** All three reverted. More usefully, each one is now a
comment in the source at the exact place someone would try it again:

> **NOTE: do not set the silence-length extras here.** Shortening them to end
> the turn faster looked reasonable and broke recognition outright... The
> platform defaults account for that gap; leave them alone.

**The lesson.** A negative result is worth as much as a positive one and is
forgotten twice as fast. Codify it where the next person — usually me — will
trip over it.

---

## 3. Stopping a reply that won't stop

**Symptom.** Saying "stop" during a long reply worked perhaps one time in four,
and needed repeating and raising my voice.

**The obvious approach, and why it failed.** "Stop" was a keyword in the same
spotting model as the wake phrase. Tuning its threshold down made it fire on
near-silence — the log caught it triggering at an audio level *below* the
speech floor, on nothing at all — without making it any more reliable when
actually spoken. It's one short syllable competing against the assistant's own
voice through a loudspeaker a metre away. The word simply isn't a strong enough
signal.

**Reframing it.** Once instrumented, the timing was damning. The one time it did
fire, it fired five seconds into a six-second reply. Mechanically a success.
Practically useless — it stopped her as she was finishing anyway.

The insight was that the *word* carries almost no information. There is nothing
else you'd be saying while the assistant is mid-sentence. So the trigger doesn't
need to be a word at all.

**What I built.** Barge-in: 300ms of sustained speech above an adapted noise
floor cuts the reply short. No model, no threshold on a phrase, no vocabulary.

The subtlety is the noise floor, because the assistant's own voice is in the
room. During a 1.5 second warm-up at the start of each reply the floor rises to
include her, so only something clearly louder counts as an interruption. That
produces an asymmetry I chose deliberately:

- On a headset her voice is echo-cancelled, the floor settles near silence, and
  barge-in triggers easily.
- Over the phone's loudspeaker the floor settles high and barge-in becomes hard.

That's the safe direction. A reply you can't interrupt is a nuisance. A reply
that interrupts *itself* on the speaker's first word is unusable.

**The lesson.** When something is hard to detect, question whether you're
detecting the right thing. The best fix here removed the detection problem
rather than solving it.

---

## 4. A microphone that flapped

**Symptom.** After adding Bluetooth headset support, wake detection got worse
rather than better — which was the opposite of expected, since the headset mic
is closer to the speaker's mouth.

**What the log showed.** Not a detection problem at first. A routing one:

```
Listening through Bluetooth headset
Wake word detection started (idle)
...
Back on the phone mic
Listening through Bluetooth headset
```

Capture was bouncing between the headset and the phone, several times a minute.
Each bounce tore down the audio pipeline and built a new one with a fresh model
stream — so any phrase spoken across a bounce was cut in half and could never
match.

**The cause.** I'd registered a callback for audio devices appearing and
disappearing, and acted on every event immediately. But a Bluetooth link drops
out of the device list *during its own state transitions*. My code read that as
"headset unplugged", switched to the phone mic, saw it reappear, and switched
back — a feedback loop of my own making.

**What I changed.** One pending routing change at a time, with a settle period
before believing the device list. Cancel any change already queued when a new
event arrives.

**The related mistake.** In the same work I labelled the device
`Bluetooth headset (classic, narrowband mic)` in the log. Classic Bluetooth runs
at 8kHz *or* 16kHz depending on what the headset negotiates — I never measured
which. Later I read that log line back as evidence for a narrowband diagnosis,
having written the word "narrowband" into it myself. The log now prints the
device's actual reported sample rates.

**The lesson.** Be careful what you assert in a log. Anything you write there
will eventually be read back as evidence, including by you, and a label that
encodes an assumption is worse than no label at all.

---

## 5. The answer it already had

**Symptom.** Mid-conversation, I mentioned a friend's name and a fact about
them in passing — never said "remember this," just said it. A few turns
later I asked whether it had caught that. It told me, flatly, that it had no
record of the conversation at all.

**What I assumed.** That this was a memory gap — the fact simply hadn't been
saved, and the honest answer was "no."

**Why that was wrong.** It had been saved. The fact was sitting in the exact
context every reply already has access to, at the moment the denial was
generated. The model wasn't missing information. It had the right context in
front of it and produced a fluent, confident, wrong answer anyway.

**What I built.** A question shaped like "did you remember X" or "what did I
tell you about X" is now answered by a direct, deterministic lookup against
what's actually stored, before the model is ever asked. Not a better prompt
asking it to please check its context properly — removing the step where it
gets to guess.

**The lesson.** Having the right information available isn't the same as
using it. For anything that has to be *right* rather than *plausible*, the
fix isn't phrasing the question better. It's not asking a model the question
at all.

---

## 6. The magic packet that couldn't leave the house

**Symptom.** Remote wake worked perfectly at home and failed silently the one
time it actually mattered — testing it from mobile data, away from the house,
produced nothing. No error, no packet, no wake.

**What I assumed.** A configuration problem. Wrong MAC address, adapter not
armed for wake, BIOS setting reverted — something checkable, something with a
setting to fix.

**Why that was wrong.** Every setting checked out. The adapter was armed. The
BIOS was correct. The packet was being sent, correctly formed, to the right
address. None of that mattered, because the premise was wrong: this was never
a configuration problem to begin with.

A Wake-on-LAN packet is a link-layer broadcast. It doesn't have a route past
the local network — there's no IP path for it to take, the way there is for a
normal packet, because it isn't addressed to anyone; it's shouted at everyone
on the wire. It can't cross the internet for the same reason a shout in one
room can't be heard in another building. And it can't ride the VPN mesh
either, for a subtler reason: the sleeping machine's own VPN client is asleep
too, so even if the packet somehow arrived at the tunnel, there'd be nothing
listening on the other end to forward it onto the physical network.

No amount of correct configuration fixes a packet that has nowhere to go.

**What I built.** Not a workaround on the same machine — a second, independent
one. An old phone, left permanently on the home network with its screen off,
running a script with exactly one job: accept an authenticated request over
the VPN mesh, and put a magic packet on the *local* wire it's actually sitting
on. It doesn't talk to Core, doesn't import anything from the rest of the
system, and doesn't need Shadow to be working — which is the one property
that actually matters, since it exists specifically for the moment Shadow
isn't.

The main phone now tries both routes on every wake attempt: the direct packet,
which is instant when you're home, and the relay, reached over the VPN, which
is the only one that can possibly succeed from anywhere else.

**What it showed.** Testing end to end — phone on mobile data, Wi-Fi off,
desktop actually asleep — the direct packet failed exactly as predicted, and
the relay carried the request the rest of the way. The desktop woke. A queued
command that had been sitting on the phone ran itself the moment the machine
came back.

**The lesson.** Not every failure is a setting away from working. Some
failures are structural — the thing you're asking for literally cannot happen
along the path you're asking it to travel, no matter how correctly everything
along that path is configured. The fix isn't to configure harder; it's to
notice the path doesn't reach, and add something that stands where it needs
to end.

---

## 7. A safeguard against duplicates that quietly ate real data

**Symptom.** Testing a new health-tracking feature by voice — "log my
resting heart rate as 68" — the assistant said "Logged" every time. The
stored history disagreed: some logs simply weren't there.

**What I assumed.** The storage layer was fine; this had to be a routing or
parsing bug further up, since the confirmation was clearly firing.

**Why that was wrong.** The confirmation firing was exactly the problem —
it meant the code path completed successfully and still didn't write
anything. The feature has two sources of the same kind of reading: you can
say a number out loud, and (a separate later addition) a watch can sync one
in automatically. Automatic syncs re-send the same reading on every sync by
design — the sync has no reliable "only what's new" cursor, so it re-reads
a trailing window every time and leans on deduplication to make that safe.
That dedup keyed on timestamp at second resolution. A voice log stamped in
the same second as anything already stored — trivially easy, since "log my
resting heart rate as 68" takes under a second to say and process — got
silently treated as a repeat of something that didn't exist yet, and
dropped. The safeguard built for one intent (don't re-store what a sync
already sent) was firing against a completely different intent (a human
just told you a fresh number) that happened to share a data shape.

**What I built.** Split what had been one function into two that name the
intent instead of inferring it: a voice log always stores, full stop — you
said it once, it happens once, no dedup logic gets a vote. A synced reading
still deduplicates on timestamp, because that's the exact case the
safeguard exists for. Same storage underneath; the difference is which
guarantee the caller is asking for.

**The related bug, same commit.** The anomaly detector that flags an
unusual reading had a second version of the identical mistake at a
different layer. It compared a new reading to your history using one
threshold — more than two standard deviations from your own average — with
no floor under it. Tested against a baseline that happened to cluster
tightly, a 3bpm swing tripped it. That's not a finding; day-to-day
heart rate moves more than that doing nothing in particular. A detector
that flags ordinary noise trains you to stop trusting it, which is exactly
backwards for something meant to catch the one reading that matters. Fixed
by giving each measurement category its own noise floor — normal day-to-day
movement for *that* metric — and requiring a reading to clear both the
statistical test and the noise floor before it counts as unusual. Checked
against both a tight and a noisy synthetic history afterward: the 3bpm
swing is now ignored, a 13bpm one is still caught.

**The lesson.** A safeguard is built against one specific danger, and it
doesn't know that when a second, different intent starts sending it
data that happens to look the same shape. "Working as designed" and
"correct for this caller" are different claims — the first one doesn't
prove the second.

---

## 8. Three failures that were obvious to a person and invisible to the system

**Symptom.** A friend gave me a real C# problem — an infinite loop he
couldn't place. I passed it to the assistant in front of him. What came
back was confident, plausible, and wrong: an analysis that, in his words,
would have sent him round in circles. This was the first time I'd put the
thing in front of someone who hadn't watched me build it, and it was
embarrassing.

**What I assumed.** The model was the weak link. The reasoning wasn't good
enough for real debugging, and I'd need a better one.

**Why that was wrong.** The model never got the chance to reason. Reading
the logs afterward, three separate failures had stacked, and none of them
were about analysis quality.

First, *the code never arrived*. The entire payload that reached the
system was forty-nine characters: a spoken summary of the problem, not the
problem. The input path was dictation, and dictation captured what my
friend said about his bug rather than the source of it. The model was
asked to find an infinite loop in code it had never seen, and — having no
way to say "you haven't shown me anything" — did what these models do with
a plausible-sounding request: it answered anyway.

Second, *it went to the wrong tool*. I'd sent it to the architecture jar,
which exists to turn a spoken description of a system into a design
diagram. It is not a debugger and was never meant to be. Even with the
full source pasted in perfectly, it would have replied with a system
architecture, because that is the only thing it does. There was no
code-analysis tool to route to; I hadn't built one.

Third, and worst, *the error message lied*. The model did answer — nearly
two thousand characters, billed to my account. Our own JSON parsing threw
it away, because the reply contained a literal newline inside a string
value, which is invalid JSON by the letter of the spec and trivially
recoverable in practice. The failure was then reported to the user as
"couldn't reach Claude." So the one diagnostic signal available pointed
directly away from the actual fault: a working network, a successful paid
call, and a message telling us the network was down.

The thing that stayed with me is how each of these looks from the two
sides. To a person, every one is a one-second diagnosis. *You didn't send
him the code.* *That's the diagram tool.* *The network's fine — it charged
you.* To the system, all three were structurally invisible, because at
every layer something completed successfully: dictation produced valid
text, the jar received a well-formed request, the API call returned 200.
Nothing anywhere had the context to notice that the whole exchange was
meaningless. A simple problem for me was a hard problem for it, and not
because the reasoning was weak — because nothing in the chain was
positioned to know what it was missing.

**What I built.** The parsing fix first, since it was throwing away work
I'd already paid for: a reply whose only fault is unescaped control
characters is now re-parsed leniently and used. Structural damage —
truncation, genuinely broken syntax — is still rejected rather than
"repaired", because a half-guessed design renders as an
authoritative-looking wrong diagram, which is a worse outcome than an
honest failure.

Then the error messages. The failure reason now travels with the failure
instead of being guessed at the call site, so the four distinct ways
generation can fail — unreachable, no JSON at all, unparseable, no diagram
in the reply — each say which one happened. There's a test asserting that
no non-network failure ever mentions reaching Claude again.

**What I haven't built yet.** A real code-analysis tool, separate from the
architecture one, and a way to get actual source onto the phone that isn't
dictation. Those two are what would genuinely have saved that demo; the
fixes above only mean it would have failed honestly instead of
confidently. I'd rather be clear about that than describe this as solved.

**The lesson.** An error message that names the wrong cause is worse than
one that names no cause, because it spends someone's attention actively
moving them away from the fault. And a system that cannot distinguish
"I have the information" from "I have forty-nine characters and no code"
will answer both with the same confidence — which means the confidence
carries no information at all. The fix for that isn't a better model. It's
building the places where a component is allowed to say it hasn't been
given enough to work with.

---

## 9. Earlier problems, more briefly

Six from earlier in the project. Same shape, less space.

**A silent startup crash with no error output.** The desktop app runs without a
console window via `pythonw.exe`, whose `sys.stdout` isn't merely
non-interactive — it's `None`. That broke the web server's default logging
setup in a way that produced *zero* diagnostic output: the app simply died.
Root-caused by instrumenting every startup step with explicit file logging,
then handling the `None` stream defensively. The lesson is the same one as case
study 1 from the other direction — when there's no output at all, suspect the
output mechanism before the code it was meant to be describing.

**A multi-second lag after every wake phrase.** Traced to name resolution:
resolving `localhost` on Windows tries IPv6 first and falls back to IPv4, and
that fallback got substantially worse with a VPN mesh's virtual interfaces
active. Fixed by connecting to `127.0.0.1` explicitly. A one-line change that
took far longer to find than to make, because the symptom looked like slow
speech processing and the cause was in the network stack.

**Regex triggers couldn't cope with real speech.** Commands like "meeting Friday
2:32pm calendar" — trigger word at the end, date and time in the middle — broke
phrase-anchored matching completely. Rather than accumulate patterns
indefinitely, this prompted a mid-project pivot to AI intent classification with
slot extraction, which handles arbitrary phrasing and word order without
enumerating it. The regex layer survives as a fallback for when the model is
unavailable, which turned out to be the right place for it.

**Losing the first word of every command.** "Take note, buy milk" consistently
transcribed as "note, buy milk". The recording start was racing against when I
actually began speaking. Fixed with a continuously-recording ring buffer
prepended to every capture, so the audio always begins slightly before the
trigger. You cannot react fast enough to a sound you haven't heard yet — the
only fix is to already have been recording.

**A cancel that didn't cancel everything.** A skill mid-conversation keeps its
own private state — waiting on a yes/no, a name change, a delete
confirmation. Interrupting with a fresh wake cleared the top-level
bookkeeping but left that private state untouched, so a confirmation you'd
walked away from could resurface later and the assistant would get stuck
re-asking a question you'd already abandoned, forever. Every piece of state
like this now gets its own explicit clear, wired into the same interrupt
path — a lesson that had to be relearned a second time, for a fourth piece of
state, a day after fixing it for the first three.

**A fix for quiet speech that made quiet speech untestable.** Detected speech
below a certain volume gets automatically boosted before anything downstream
sees it — reasonable, since a mumbled command still deserves a chance. It
also meant that any test simulating a quiet or distant voice got quietly
un-simulated: the same boost fired during the test, so "how does this handle
quiet speech" could never actually be answered by turning the volume down and
checking. Testing that specific condition now has to explicitly bypass the
boost it exists to prove is doing its job.

---

## The thread running through all of these

Every one of these was extended by the same habit: forming a plausible theory
and acting on it, when the cost of checking first was low.

What changed was building the instrumentation *before* the fix. Every problem
above became tractable within one attempt once the log could actually answer the
question — and several had resisted multiple attempts before that.

The visible output is a working assistant. The useful output was learning to
tell the difference between a conclusion and a hypothesis that feels like one.
