# Judge and Simulator Pattern

Two models sit inside the measurement path: one plays the customer, one grades the
conversation. Both are instruments, and an instrument that is not characterised is not a
measurement.

---

## They Must Be Independent

**Separate prompts always, and different models by default.** The scorecard records which
model played each role and **flags a shared one.**

A judge scoring text produced by its own model tends to favour it. The failure mode is what
makes this worth enforcing: the bias does not look like bias. It reads as a well-handled
conversation rather than as a shared prior between the author and the grader.

---

## A Conversation the Simulator Ruined Is Not Evidence

```
afterRun(conversation, persona):
    fidelity = checkArc(conversation, persona)
    if not fidelity.followed:
        exclude(conversation)
        report(conversation, reason: "simulator broke character", detail: fidelity)
        return
    score(conversation)
```

A persona carries an emotional arc — where the caller starts, what shifts them, where they
end up. If the simulator drifted out of it, the conversation measures **our simulator, not
the target**, and scoring it attributes our defect to the agent.

Excluded conversations are **reported, not silently dropped.** A run that excluded four of
twelve conversations is telling you something about the simulator that you need to know.

---

## Calibration: Does the Judge Agree With a Person?

Measured with blind human labels, and **always reported against the human-to-human
ceiling** on double-labelled conversations.

```
calibrate(metric, conversations):
    humanLabels = blindLabels(conversations, labellers >= 2)
    ceiling     = kappa(humanLabels[0], humanLabels[1])   # human vs human
    judgeAgree  = kappa(judgeLabels(conversations), humanLabels)
    return { judgeAgree, ceiling, headroom: ceiling - judgeAgree }
```

### Why the ceiling and not 1.0

A judge at kappa 0.55 where two trained people reach 0.58 is **at the limit the rubric
allows.** Reporting that as "0.55 agreement" against an implicit target of perfect
agreement attributes irreducible rubric ambiguity to the model.

The actionable signal is the headroom. Where it is near zero, **it is the rubric that needs
work, not the judge** — the definition is too vague for anyone to apply consistently.

### Labelling is blind

A labeller who has seen the judge's band is anchored to it. The labelling page shows the
conversation and the metric definition, and nothing else.

The labeller is also checking **the judge, not the agent.** That distinction has to be
stated in the instructions, because the natural instinct of a domain expert reading a
transcript is to form an opinion about the agent.

---

## Judge-Scored Metrics Count, and Say They Are Uncalibrated

| Option | Consequence |
|---|---|
| Withhold uncalibrated judge metrics | Whole pillars go unmeasured; **a scorecard silent on empathy reads as one where empathy was fine** |
| Include them, labelled | A number of unestablished reliability is on the page, clearly marked |

The second is the lesser distortion — but it **is** a distortion, so:

**A scorecard cannot be non-provisional while any metric it relies on comes from an
uncalibrated judge.** Every such result is labelled *"judge not yet calibrated"* wherever
it appears.

---

## Consistency: Does the Judge Agree With *Itself*?

A separate measure, for a separate fault, with a different fix.

```
consistency(metric, conversations, n = 3):
    runs = [judgeLabels(conversations) for _ in range(n)]   # identical inputs
    return perCheckVariance(runs), flipRate(runs)
```

### Why both measures are needed

- A judge can be **perfectly consistent and consistently wrong.** Calibration catches that;
  consistency does not.
- A judge can be **well calibrated on average while deciding individual cases close to a
  coin-flip.** The average hides it; only consistency shows it.

Neither substitutes for the other.

### It is not a theoretical concern

Re-scoring stored assessments — identical transcripts, identical questions, identical model
and temperature, nothing changed but the request — moved **between 0.2 and 1.4 check
results per conversation.** Nothing about the agent differed.

**One judgement is one draw from a distribution.** Reporting it as the answer overstates
it, and the scorecard shows judge agreement and stability next to judge-scored metrics so
a reader can weight them accordingly.

### What to do about an unstable measure

An unstable metric is usually a rubric problem rather than a model problem — the same
diagnosis as near-zero calibration headroom. The fix is to tighten the definition until
two people, and then the judge, can apply it repeatably. Raising the sample count only
averages the instability away without removing it.

---

## Changing the Judge Breaks Comparability

```
if scorecard.judgeModel != priorScorecard.judgeModel:
    mark(NOT_DIRECTLY_COMPARABLE)
```

**Calibration and consistency are per judge model and are never inherited.** A new judge
model starts uncalibrated regardless of how well its predecessor scored, because the
measurement instrument changed.

This is why re-scoring happens by [replaying stored conversations](honest-scoring.md)
rather than re-running the agent: it isolates the change being tested instead of mixing it
with a fresh set of conversations.

---

## Judge Failure Is an Error, Never a Low Score

```
try:
    result = judge(conversation, metric)
except JudgeError as e:
    return Check(status = ERROR, rationale = str(e))   # never score = 0
```

A judge that timed out, refused, or returned unparseable output has produced **no
measurement.** Recording that as a low score blames the agent for our infrastructure, and
it is indistinguishable in the output from a genuine poor performance.
