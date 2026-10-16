# Voice Measurement Pattern

Turn-taking, latency and speech quality measured over a telephony path — where every
interesting metric is a difference between two timestamps, and the measuring apparatus sits
inside the thing being measured.

---

## One Monotonic Clock, Per-Speaker Tracks

```
Stream: full-duplex, bidirectional
Tracks: [ caller (ours), agent (target) ]   — separate audio tracks
Clock:  one monotonic source, both tracks timestamped against it
```

**The shared clock is the foundation**, because every turn-taking metric is a difference
between two timestamps:

| Metric | Computed as |
|---|---|
| Response latency | agent first speech − caller end of speech |
| Barge-in responsiveness | agent playback stop − caller speech start |
| Talk-over duration | intersection of the two tracks' speech intervals |
| Turn count, pace | boundaries on each track |

Two tracks timestamped against two clocks produce differences that are a mix of real
behaviour and clock skew, and nothing in the output distinguishes them.

Per-speaker tracks are equally non-negotiable: talk-over is **the intersection of two speech
intervals**, which a mixed single track cannot recover.

---

## Measure Our Own Latency and Subtract It

```
agentResponseLatency = agentFirstAudio − callerSpeechEnd − ourTtsFirstByteDelay
```

Our synthesized-speech first-byte delay sits **inside** the interval that would otherwise be
attributed to the target. So it is measured, continuously, and subtracted.

**Reporting it as the agent's latency is a measurement error, not a conservative estimate.**
That distinction matters: a conservative estimate is a defensible choice, while an
unattributed component of our own apparatus in the target's number is simply wrong, and
wrong in the direction that makes every agent look worse than it is.

The general rule: **when the instrument participates in the thing it measures, its
contribution must be characterised and removed, not absorbed.**

---

## Turn-Based Audio Cannot Measure Turn-Taking

An earlier pipeline recorded one utterance per call leg, then hung up — record, transcribe
in batch, done. It could not measure barge-in, talk-over or response latency **however it
was wrapped**, because the behaviours it needed to observe cannot occur in a
one-utterance-per-leg exchange.

The adapter therefore reports `supports_conversation = False` and raises a capability error
for conversational calls, so the runner reports those metrics **`not_assessed` rather than
recording a conversation that never happened.**

```
if not adapter.supports_conversation:
    raise AdapterCapabilityError(metric, "adapter is turn-based")
    # runner -> not_assessed, with the reason
```

**This is the honest handling of a capability gap:** declare it, report the affected
metrics as unmeasured with a stated reason, and require a real conversational path rather
than approximating one. Wrapping the old pipeline would have produced plausible-looking
numbers for behaviours that were never observable.

### What streaming requires that batch does not

| Need | Why batch fails |
|---|---|
| Streaming STT with voice activity detection | Batch transcription cannot detect end-of-speech live, so there is no turn boundary to timestamp |
| Streaming TTS | Required to speak as a caller at all, and to measure our own first-byte delay |
| Playback flush on interrupt | Barge-in is only observable if our own audio can actually be stopped mid-utterance |

---

## Speech Quality Is Relative to the Path

```
predicted_mos:
  objective: relative
  reference: same_path_baseline        # known-good call, same trunk and codec
  # reports not_assessed until such a baseline exists
```

The MOS predictors are trained on wideband audio. An absolute threshold over a narrowband
telephony path **fails a target for the offence of being on a phone.**

The question worth answering is not *"is this audio good"* but **"is this agent worse than
this path allows."** That requires a reference recorded through the same trunk and codec,
and until one exists the metric reports `not_assessed` rather than guessing.

### Absolute MOS is a separate, narrower metric

Where synthesized speech can be captured **before** the telephony path, an absolute
wideband score is meaningful — and it is a distinct white-box metric with a different name,
not the same metric with a different threshold. Two different questions do not share a
metric id.

---

## Consent and Disclosure Are Part of the Measurement Path

A voice assessment records audio, which engages recording-consent obligations wherever the
call lands. And the synthetic caller is not a person.

Both constraints are handled in the run path rather than in guidance, and both interact
with the [ownership policy](assessment-ethics.md): a voice assessment of an agent the
operator does not own carries the same stop-at-offer rule, the same pacing limits, and the
same obligation that the target's call minutes be reported as cost billed to its owner.

---

## Replay Is the Primary Test Path

Voice runs are the most expensive and least repeatable thing the platform does — every
attempt places a real call.

So stored audio and per-track timestamps are the primary input to development and to
re-scoring. A threshold change, a rubric change or a judge change re-scores the stored
tracks; it does not redial.

This is the same principle as text replay, with a sharper edge: redialling to test a
scoring change produces **different** audio over a **different** path instance, so the
comparison the change was meant to inform is destroyed by the act of gathering it again.
