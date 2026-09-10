# Case studies

Problems from this project that took real work — fifteen recent ones in
detail, six earlier ones briefly. Each follows the same shape: what it looked like,
what I assumed, what the evidence actually said, and what I changed.

Three of them (11, 12 and 13) come from the way this project is now built: I
plan a change, an AI coding agent implements it, and I review the diff
rather than the summary it hands back. All three are included because of what
that review caught, not in spite of it — and 13 because half of what it
caught was wrong in my own plan.

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

## 9. The claim in a comment that closed a phone's connection

**Symptom.** I'd just redesigned how the architecture jar (see [case study
8](#8-three-failures-that-were-obvious-to-a-person-and-invisible-to-the-system))
stores its in-progress work, restarted the desktop app to pick up the
change, then opened the corresponding screen on my phone. It came back with
"couldn't reach core, EOF" and the connection closed immediately.

**What I assumed, and wrote down as fact.** While writing the redesign, I
added an internal field to the data one function returns, and documented it
with a claim about the code that consumes it: that a specific handler "only
forwards a few named fields," so the extra field would never actually leave
the process. I never opened that handler to check. It read as obviously
true from the calling code alone.

**Why that was wrong.** The handler forwards the whole structure wholesale,
with no filtering at all. The extra field I'd added held a filesystem path
object, not a plain string — and the moment a real request tried to
serialize that structure to JSON to send over the wire, it failed outright.
The error handling around that failure closed the socket immediately,
which is exactly the "EOF" the phone saw.

**What the log actually said**, once I checked it instead of the comment
I'd written:

```
TypeError: Object of type WindowsPath is not JSON serializable
when serializing dict item '_folder'
when serializing dict item 'diagrams'
```

Two lines that state the entire bug outright, sitting untouched in the log
the whole time it mattered.

**What I built.** Moved the actual filesystem lookup into a function that
nothing outside the module ever sees the return value of — the data handed
back to any caller now only ever carries plain, wire-safe fields. Added a
test that does the one thing that would have caught this before it
shipped: takes the real function's real output and runs it through the
same serializer the network layer uses, asserting it doesn't raise.

**The lesson.** A comment that states how another piece of code behaves is
a claim about that code, not a fact available from the file you're
actually editing. It needed exactly one grep before being written down,
and it shipped anyway because that grep felt unnecessary — the calling
code's own shape seemed to make the claim obviously true. It didn't cost a
debugging session to catch. It cost a live connection, closing, the moment
it mattered.

---

## 10. The test that would have failed a working component

**Symptom.** None yet — that's the point. I was designing a planning tool whose
core component is a small local model that judges whether a given question is
worth asking about a problem. Nothing was built. Following my own rule for this
project, I wrote the acceptance test for that component before writing the
component: state in advance what result would prove it isn't good enough.

**What I assumed.** I wrote it down like this: *if the judge marks more
questions irrelevant than relevant on a hand-labelled set, the model is too
weak.* The reasoning felt solid. A model that can't see why a question matters
will reject it, the pool of questions shrinks, and the coverage figure I planned
to show myself goes up. It fails in the flattering direction, and it fails
silently — exactly the thing worth writing a test for.

**Why that was wrong.** The model runs locally and costs nothing per call, so
before building anything I built a set of fourteen questions instead — six that
genuinely mattered for a worked example, eight that didn't — and put them past
it one at a time.

It agreed with me on all fourteen.

It also returned eight "not relevant" against six "relevant", which trips my
test. My own acceptance criterion would have condemned a component that had
just scored a hundred percent.

The ratio isn't a property of the model. It's a property of the set I handed it,
and I chose to make most of those questions irrelevant. Worse, the design
deliberately generates far more candidate questions than it needs so that most
get filtered away — which means a *correct* judge will normally return more
rejections than keeps. The test fires hardest exactly when the thing works.

**What the corrected test caught.** I replaced the ratio with the number that
actually matters: how many genuinely relevant questions get wrongly thrown away.
That's the only error I can't see, because a discarded question never appears on
screen at all.

Then I made the set harder — adding questions *adjacent* to a relevant one but
asking something subtly different — and ran it three times:

```
FALSE DROPS (relevant judged NOT)  1 of 8
    DROPPED: Who has to approve a schema change on the order service?
FALSE KEEPS  3 of 12
    obvious    0/4      plausible  0/4      near-miss  3/4
UNSTABLE across 3 runs at temp 0   0 of 20
```

It threw away "who has to approve a schema change" on every run, not as noise.
That's one of six fixed slots the whole design is built around, and it is
near-word-for-word the example my own design document uses to illustrate a
*high-value* question. The original test would have passed this run. The
corrected one failed it on the first attempt.

The rest of that table matters too: perfect on obviously irrelevant questions,
perfect on plausible-but-wrong ones, three of four wrong on the near-misses.
It's a padding filter, not a judge — and the boundary is the only place a judge
would earn its keep.

**The fix I proposed, and what happened to it.** I thought the model might do
better if it had to state what each question eliminates before voting. Testing
that took ten minutes and killed it twice. Letting the model apply the rule made
it worse — wrong keeps went from three in twelve to ten in twelve — and on every
near-miss it wrote `"eliminates": "nothing"` and then voted the question
relevant anyway. So I tried having my code make the call from that field instead
of trusting the vote. Told it was only reporting and not deciding, the model
wrote "nothing" for all sixteen questions, including the four that genuinely
mattered.

**What I changed.** The judge no longer decides the coverage figure — it
annotates, and I can overrule it. The six fixed slots are never sent to it at
all, so the failure that fired cannot recur. Four claims in the design document
that measurement had made false were corrected in place rather than left
standing, and the failed fix was written up with its numbers as a
do-not-retry — a good idea that doesn't work gets re-proposed in a month
otherwise.

**The lesson.** A test is code, and it can be wrong in the same way the thing it
tests can be. Mine wasn't sloppy; it encoded a specific and reasonable
prediction about which direction the failure would come from, and the prediction
was wrong. Every earlier case study here is about checking a claim before acting
on it. This one is a level up: check that the instrument measuring the claim
measures what you think it does. It was catchable for the same reason as all the
others — running it cost three minutes, and I ran it before building on top of
it.

---

## 11. The document that was wrong about its own repository

**Symptom.** None in the code. The failure was in a handoff document — the
kind I write at the end of a working session so the next one doesn't start
cold. This one carried a section headed "where things actually are,
verified not assumed", listing which parts of a feature existed and which
didn't. Three of its rows said a component "does not exist". An
architectural decision had been built on top of those three rows, and was
sitting open, waiting on me.

**What I assumed.** That a section explicitly labelled "verified not
assumed" had been verified. It named specific files and specific handler
functions, in the tone of something checked rather than remembered. The
decision it framed offered three ways forward, and that framing only makes
sense if those three things are genuinely missing.

**Why that was wrong.** All three existed. The session-persistence module
was on disk at 17 KB. All six network handlers it claimed were missing
were in the main server file, at line numbers I could point at. The phone
screen it described as an unbuilt placeholder was written and compiled,
and the file that supposedly still held the placeholder had a comment
saying it had been replaced.

They had been merged the previous evening at 20:12. One command settles
when:

```
git merge-base --is-ancestor 31f64f5 HEAD
```

It confirms the branch was cut *after* that merge. The code was sitting in
the working directory of the session that wrote "does not exist", the
entire time it was writing it.

**What that cost.** Of the three paths the decision offered, one was
"build this whole component first" — already done, the previous evening.
Another was "leave it as a script-only tool and put it on the phone
later" — it was on the phone already. Two of three options were answers to
a question that had stopped being a question. Meanwhile the real problem
underneath, two overlapping stores for the same state, wasn't described
anywhere, because the document couldn't see it from where it thought it
was standing.

**What I built.** Not code, at first. The correction went into the new
plan as its own opening section, with the measurement beside each claim —
file size, line numbers, the ancestry command and its output — so the next
reader can re-run them instead of trusting me the way I'd trusted the last
document. The stale rows were struck through in place rather than deleted,
each annotated with which half of it had actually been right. One row was
partly correct, and quietly rewriting it would have destroyed that
distinction.

Then the check that made the difference. The claim "this module does not
exist" is disproved by `ls`. That is a one-second command, and nobody ran
it, because the sentence was already written down in a section that said
it had been.

**The lesson.** A tool returning nothing isn't evidence of absence until
you know the tool could have seen it — and a *document* returning nothing
is weaker evidence still, because a document can't see anything at all. It
records what someone believed at a moment, and it starts decaying the
instant it's written. "Verified" in a heading is a claim about the past,
not a property of the present.

This is the fourth time in this project that a confident statement of
absence turned out to be a stale or structurally blind check. The others:
a service reported down that was serving fine the whole time, a crash
diagnosed that had never happened, and a branch reported missing that
existed but hadn't been fetched. That repetition is why my working notes
for this project now require naming the single check that would
*disprove* a diagnosis, and running that one — not the check that would
confirm it.

---

## 12. A version bump that would have made real work unopenable

**Symptom.** None yet. This one was caught in design, which is the only
place it could have been caught cheaply.

**The setup.** Two pieces of state storage had grown up independently for
the same feature: one holding a planning tool's own session, the other
holding the handoff between four separate pipeline stages. They
overlapped, they didn't reference each other, and they disagreed about
something real — one of them stored a progress count on disk, the other
deliberately recomputed it on every read so it could never drift from the
data it summarises. Merging them meant changing the file format, which
meant incrementing the schema version integer stamped into every file.

**What I checked before writing the plan.** What the existing version
policy actually does on a mismatch. It's strict, and deliberately
asymmetric: a *temporary* session at an unrecognised version is discarded
with a message naming both versions; a *saved* session is refused
outright, named, and **left on disk untouched**. That asymmetry is right —
throwing away something a person explicitly chose to keep is worse than
declining to open it.

Then what was actually on disk:

```
2026-09-07 202012060642  saved=True  status=ready
"Logs are sent to many different sources, softwares and pcs with
 different operating systems, need aggregation system"
```

One saved session. Real work, not test data — a problem I'd been planning
through the day before.

**Why that mattered.** Incrementing the version without a migration path
would have sent that file straight down the "refuse to open" branch,
permanently, from every surface including the phone. Nothing would have
been deleted. Nothing would have failed in the test suite. The file would
simply have stopped opening — correctly, according to a policy working
exactly as designed.

**What I specified.** A read-time migration lane: a registry of known
upgrades applied in memory only, so an older file is brought up to date as
it's read and the file on disk is never touched. Only a version with no
registered upgrade still hits discard-or-refuse. The migration widens what
counts as readable; it doesn't weaken what happens to something genuinely
unreadable.

The plan was written constraints-first, with the migration step ordered
explicitly *before* the version bump and an instruction not to reorder
them, because in the other order the destructive window is real. Each step
carried its own falsifier — what to measure, what it must read, and what
it means if it reads something else. For this one: open the real saved
session after the change, confirm its text is intact, and hash the file
before and after to prove the migration never wrote.

**Then I reviewed the implementation, and found two things the plan hadn't
protected against.** An AI agent wrote the code from that plan. Reviewing
the diff rather than the summary it returned is what turned these up;
neither is visible from a written report of the work, because the report
is produced by the same process that did it.

The first was a silent failure waiting for the *next* version bump. The
migration registry mapped an old version straight to its upgrade function,
and the loading code assumed every registered function landed on the
current version. True while there's one migration. Add a second, and a
file at the oldest version runs only the first hop and is then accepted as
fully current — remaining upgrade skipped, no error, no warning. I proved
it rather than arguing it: set the version constant forward by one, fed it
an old file, and watched it load clean when it should have refused. The
fix makes each migration declare the version it upgrades *to* and walks
the chain, so a file that can't reach the current version falls through to
the loud path instead of the quiet one.

The second was worse, because it was an absence. The migration lane — the
most dangerous path in the whole change, the reason the plan was ordered
the way it was — had **no test at all**. It looked covered. There were
several tests about schema mismatches. But every one of them wrote a file
at the *current* version and then incremented the constant, which
exercises the refuse-and-discard branch and never once reaches the upgrade
branch. A green suite, over the exact code whose failure would have made
my own saved work unopenable.

**The lesson.** Two, and the second is the one I'd want to be asked about.

A version number is a promise to files that already exist. The cost of
breaking it isn't paid by the code you're writing — it's paid by data
someone already chose to keep, on a path no failing test will ever point
at, because from the code's point of view, refusing to open a file it
doesn't understand *is* success.

And a test suite tells you which branches were exercised, never which were
skipped. Several tests named the schema policy, and it was easy —
correct-looking, even — to read that cluster as coverage. What made the
gap visible wasn't reading the tests. It was asking which specific input
reaches this specific line, and finding that nothing in the suite ever
produced one.

---

## 13. The one-word default that would have left the machine unstartable

**Symptom.** None. Every test passed — 68 new ones for the module, and the
full suite of 56 clean. This one was caught by reading the diff.

**The setup.** I'd specified a self-healing feature: record which boots
actually worked, so a broken one can be diagnosed against the last good
one. A boot only counts as good if it survives a soak window — 120
seconds — because otherwise a crash loop stamps a fresh "known good" every
cycle and the record ends up pointing at the very state doing the
crashing. The implementation was a timer:

```python
threading.Timer(SOAK_SECONDS, promote_if_soaked).start()
```

That line is faithful to the plan. The plan was wrong.

**What I checked.** Four properties, none of them interesting alone:

1. `threading.Timer` inherits its daemon flag from the creating thread.
   Created on the main thread, it is **non-daemon** — so the interpreter
   waits for it at exit. Confirmed by running it, rather than trusting my
   memory of the documentation.
2. Every *other* thread in that file — the web server, the voice loop, the
   startup watcher — is explicitly `daemon=True`. This was the only one
   that wasn't.
3. There is no `os._exit` or `sys.exit` anywhere in the file, so nothing
   forces the process down past a lingering thread.
4. The single-instance lock is a named Windows mutex whose handle is
   stored and never released. Windows frees it **when the process exits**.

**What they meant together.** Quitting the app inside the soak window left
the process alive, holding the mutex, for the rest of the 120 seconds. The
relauncher — the detached helper that brings the app back after a restart —
waits 60 seconds for that mutex and then gives up.

So a restart in the first minute of a boot would have left the machine with
the app down, the tray icon already gone, and the one process that could
have restarted it having quietly timed out. That is the precise failure the
restart design exists to prevent: it spawns the relauncher *before* stopping
anything, so there is always a way back. The way back was still there. It
just couldn't get in.

Worst of all during rapid restarts — which is exactly the workflow the
feature's own safety rule had been written to protect.

**The fix is `daemon=True`.** Safe, because the promotion function already
refuses to run unless a boot is still in progress, so a process on its way
out has nothing to promote.

**Why no test would have found it.** The defect isn't in a function. It
lives in the interaction between a thread's default flag, an OS handle's
lifetime, and a timeout in a different file — and it only appears when a
person quits at the wrong moment. There is no unit that fails.

**The part I'd rather leave out, which is why it's here.** Four findings came
out of that review. Two of them were faults in *my plan*, not in the code
written from it.

I had specified that the always-on health line should report "N settings
files changed" and "N commits since". Both are ordinary daily activity — the
line would have been non-empty most days, on a machine with nothing wrong
with it. That is the exact failure the step I wrote opens by warning about,
three paragraphs above the mistake.

And the falsifier I attached to it — the measurement meant to prove the step
correct — asked *"is it silent when nothing has changed?"* The question that
mattered was *"is it silent when something has changed but nothing is
wrong?"* The implementation answered my question correctly, and passed.

**What I take from it.** A falsifier that only tests the case its author had
in mind is not a falsifier. Reviewing generated code catches what the
generator got wrong; reviewing it against a specification you wrote yourself
catches what *you* got wrong — and that second one is easier to miss,
precisely because the code agrees with you.

**Part two: verifying the fix, and the one thing I couldn't.** A fix
reviewed on a diff is still a claim until it runs against a real system. So
I ran it — a real desktop app, real process kills, a real socket, no mocks.

Five of six planned checks passed with numbers I actually measured, not
inferred: two independent boots reached the "healthy" state at ~121 seconds
against a 120-second target; forty real kills against the live snapshot
-writing function, fired mid-write, produced zero corrupt files; a restore
came back byte-for-byte identical, verified by hash rather than a glance,
and touched nothing else on disk; a crash-loop refusal fired for real — the
actual system dialog, on screen — while a manual restart stayed provably
unblocked at the same time; an unauthenticated write request sent over the
live network socket was refused and confirmed, again by hash, to change
nothing.

The sixth check — does quitting inside that 120-second window actually
avoid the hang the fix exists to prevent — I could not complete. There was
no tool available to me for driving the native desktop UI: no way to click
the system tray's own menu. I tried the nearest safe substitute, a
graceful process-close signal, and it reached the target process (confirmed
against the running log and the durable state file) but produced no
response at all. Even the "safe" workaround didn't exercise the code path
in question.

I reported that plainly rather than rounding a near-miss up to a pass. The
one property left open is written down as open, not folded into "verified"
because four other properties near it were.

**What I take from it, the second time.** A fix that passes review still
owes you a real run before "done" means anything. And when a planned check
turns out to need a tool you don't have, the honest move is to say exactly
which one and why — not to substitute a weaker check and let the report
imply it covered the same ground.

---

## 14. Earlier problems, more briefly

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

## 15. The theory that felt right and the number that settled it

**Symptom.** A multi-turn voice exchange — the assistant reads out a short
list, asks which one I meant, I name one — lost all memory of the list
between the question and my answer. It replied as if I'd said the name out
of nowhere, with nothing to attach it to.

**The first explanation, and why I didn't accept it.** The claim handed to
me was that I'd paused about fourteen seconds before answering, and that
the assistant's wake claim had quietly expired somewhere in that gap. It's
a plausible story — voice systems do lose the floor to silence — and it
was stated with real confidence. I hadn't paused. I answered right after
the assistant finished talking, and I said so.

**What checking it actually looked like.** Rather than argue from memory
against memory, the fix was to find a number in the system that didn't
depend on either of us being right. The component doing the listening has
a hard configured cap on how long it will wait for speech to start — eight
seconds. The gap in the log was closer to fourteen. Eight from fourteen
leaves six seconds that could not have been spent listening for me at all,
which meant they were spent somewhere else: the assistant was still
synthesizing and speaking a four-sentence reply before it ever started
listening.

**What that reframed.** The bug wasn't "the user paused too long." It was
that a long spoken reply eats into the same fifteen-second window the
system uses to hold the floor for an answer, and the code only renewed
that hold *after* getting a reply back — never while it was still talking,
never while it was still listening. A short answer never showed the
problem. A four-item list read out loud, every time, did.

**The part worth naming.** This exact class of bug had already been found
and fixed once before, three weeks earlier — and the comment describing
that fix was still sitting in the file, describing almost the same
symptom. The first fix renewed the hold after a successful answer came
back, which covers a *fast* back-and-forth. It never covered a *single*
reply that was slow to say out loud. Not a new bug. The same bug,
incompletely closed the first time, waiting for a case that happened to
exercise the part that wasn't covered.

**What changed as a result.** The hold now gets renewed the moment
listening starts, not only after it returns — covering a slow reply's own
talking time the same way the first fix covered slow turnarounds between
turns. And every reply now logs how long it took to say, specifically so
the next time something looks like a fourteen-second silence, the log can
answer whether it actually was one instead of two people trading guesses
about it.

**The lesson.** The theory that gets offered with confidence is not
automatically the theory that's correct, including from a tool that just
watched the whole exchange happen. The fix here wasn't "trust the second
guess more" — it was finding the one number already sitting in the
configuration that neither guess needed to win the argument to settle.

---

## 16. The hotkey that also meant undo

**Symptom.** Ctrl+Z started toggling conversation mode on and off, on a build
I'd already confirmed was wired to a different combination — Ctrl+Shift+C.

**What I assumed.** That the two hotkeys were simply colliding somewhere in my
own code — maybe a leftover registration, maybe a stale handler. That would
be a five-minute grep.

**Why that was wrong.** There was no second registration. Grepping the whole
codebase for Ctrl+Z, undo, or anything z-shaped in a hotkey context found
nothing. The only hotkey registered anywhere was the correct one. Whatever
was matching Ctrl+Z wasn't in my code at all — which meant it was in the
library underneath it.

The hotkey library (`pynput`) doesn't document how it decides a combination
is pressed, so I read its actual installed source rather than guess from its
API surface. It matches a hotkey by translating each keystroke into a
*character*, based on whatever modifiers are held at that instant, and
comparing that character against the combination's own parsed form. Held
modifiers change what a keystroke translates to — that's normal keyboard
behavior, the same mechanism that makes Shift+2 produce `@`. Under two
modifiers held together, that translation turned out to produce the same
character for a different physical key than the one actually pressed. I
couldn't pin down the exact trigger condition for that collision with
confidence, and said so rather than overclaim it.

What mattered more than the exact trigger was finding something in the same
source that didn't depend on the unreliable part: the raw, untranslated
virtual-key code. It's reported on every key event regardless of what the
character translation decides, precisely because the OS layer that produces
it runs before any modifier-aware translation happens.

**What I built.** Replaced the library's built-in hotkey matcher with a small
state machine of my own: track Ctrl, Shift, and the letter key as three
independent booleans, updated from raw key-down/key-up events, matching the
letter by its virtual-key code instead of its translated character. The
toggle fires only on the letter's own transition into "pressed while both
modifiers are already down" — not on the modifiers alone.

**The part worth naming.** Writing the test for that fix caught a second,
unrelated bug before it ever shipped. My first version matched on whether the
right keys were currently *held* — a level, not an event. Held keys on
Windows resend their own key-down at the OS's repeat rate, so a single
physical hold would have re-fired the toggle every repeat interval for as
long as the key stayed down — turning conversation mode on, off, on, off,
dozens of times, from one keypress. The test that exercised "hold the
combination for a while" is what surfaced it; nothing about testing the
original collision would have.

**The lesson.** For behavior this central to a fix, the library's
documentation wasn't enough — reading its actual installed source was what
turned up the one field immune to the failure I was fixing. And a fix isn't
finished when it stops reproducing the bug that was reported; the test
written to prove it also needs to try the thing nobody reported yet, like
just holding the key down.

---

## The thread running through all of these

Every one of these was extended by the same habit: forming a plausible theory
and acting on it, when the cost of checking first was low.

What changed was building the instrumentation *before* the fix. Every problem
above became tractable within one attempt once the log could actually answer the
question — and several had resisted multiple attempts before that.

The visible output is a working assistant. The useful output was learning to
tell the difference between a conclusion and a hypothesis that feels like one.
