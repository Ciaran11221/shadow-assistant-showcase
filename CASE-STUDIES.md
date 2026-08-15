# Case studies

Problems from this project that took real work — four recent ones in detail,
four earlier ones briefly. Each follows the same shape: what it looked like,
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

## 5. Earlier problems, more briefly

Four from earlier in the project. Same shape, less space.

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

---

## The thread running through all of these

Every one of these was extended by the same habit: forming a plausible theory
and acting on it, when the cost of checking first was low.

What changed was building the instrumentation *before* the fix. Every problem
above became tractable within one attempt once the log could actually answer the
question — and several had resisted multiple attempts before that.

The visible output is a working assistant. The useful output was learning to
tell the difference between a conclusion and a hypothesis that feels like one.
