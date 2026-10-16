# Instrument Validation Pattern

A detector that fires on nothing might be looking at a clean target or might be broken. The
output is identical. Positive controls are the only thing that tells them apart.

---

## The Test Range

The rubric is run against **two variants of a deliberately flawed agent** and compared with
an answer key:

| Variant | Purpose |
|---|---|
| **vulnerable** | Carries planted flaws. Provides the **positive control** — does the metric catch the thing it is for? |
| **hardened** | The same agent with those flaws removed. Provides the **negative control** — does the metric stay quiet when it should? |

A planted flaw that the vulnerable variant does not trip means the metric never
demonstrated it can discriminate, whatever it reported on the hardened side.

---

## Two Categories of Finding on the Hardened Variant

The earlier rule called any hardened-variant finding a **false positive**. That was too
broad, and it was wrong in an unusual direction: **it overstated our own failure rate.**
Three findings were reported as detector errors when the detector was right.

```
classify(finding, run):
    metric = finding.metric
    if metric.plantedFlawExercisedAndCaught(run.vulnerable):
        return FALSE_FINDING            # positive control worked; firing here is OUR bug
    return OBSERVATION_ABOUT_HARDENED   # control never ran; cannot attribute to detector
```

| Category | Condition | Meaning | Fails strict gate |
|---|---|---|---|
| **False finding** | The metric's planted flaw **was exercised and caught** on the vulnerable variant | The measure demonstrably discriminates, so firing on the hardened variant is the detector being wrong. It costs trust in every other number. | **yes** |
| **Observation about the hardened agent** | The metric's flaw was **never exercised** | Nothing in this run shows the measure discriminates at all. The finding cannot be attributed to the detector — and usually is not its fault. | no |

### Why observations usually are not bugs

The hardened variant is hardened **against the planted flaws**, not built to be exemplary
on every experience dimension. When the judge said it reuses its closing line, it does
reuse its closing line. That is a true observation about a mediocre test agent.

**An observation turning CI red would mean failing the build because the range agent writes
a weak sign-off.**

---

## A Metric With No Positive Control Is Untested, Not Passing

This is the central rule, and it is the `not_assessed`-is-not-zero rule applied to the
instrument itself:

> *"We looked and found nothing"* and *"nothing was ever put in front of it"* must not
> render the same way.

```
metricStatus(metric, run):
    if not metric.plantedFlawExercised(run):
        return UNTESTED        # NOT "passing"
    if metric.caught(run.vulnerable) and not metric.fired(run.hardened):
        return VALIDATED
    ...
```

A range report that shows forty metrics "clean" when only twelve had a flaw put in front of
them is telling you almost nothing, and is the most dangerous possible output — because it
reads as thorough.

---

## Validating Regression Tests the Same Way

The rule generalises past the range. **A regression test is proved by reintroducing the bug
and watching the test fail.**

```
proveTest(test, fix):
    revert(fix)
    assert test.fails(), "test does not actually cover this fix"
    reapply(fix)
    assert test.passes()
```

A test that has never failed is a **hypothesis**. Running a suite and seeing green proves
only that nothing *else* broke — it does not prove the test covers what its name claims,
and it does not prove an edit even landed.

### A fix is not done until it is verified in the committed artifact

Read it back out of the commit. A commit message is not evidence.

This rule exists because the failure happened twice in one session. The worse case: a
commit message stated that a cache key had moved off object identity and onto the session.
**The edit never applied.** Because the original report said "fixed", the test that kept
failing was read as flaky for days rather than as the bug still being there.

The failure mode is trusting an account of an edit over the artifact. So the check is always
against the artifact — the file on disk, the diff in the commit, and a test whose red proves
which line is load-bearing.

---

## A Claim Needs a Test or It Gets Deleted

Applied to every sentence in a file header, a design document, and any commit message that
describes behaviour: **if it says the code does something, something must fail when the code
stops doing it.**

Three documented claims across these projects turned out to be false under that rule — a
commit that said it wrote to an activity log and did not, an export described as always
stamped that was not, and a stated rule against customer material in a repository that held
some.

All three survived human review for months, for the same reason: **prose cannot fail.**

When you find a claim nothing checks, either write the check or cut the sentence — and say
which you did.

---

## What Transfers

1. **Negative results are meaningless without positive controls.** Any evaluation system
   should be able to answer *"how do you know this measure works?"*, and the answer has to
   be a control, not an argument.
2. **Distinguish "checked and clean" from "never checked" everywhere**, including in the
   tooling that checks your tooling. The two collapse by default.
3. **Classify your own errors precisely enough not to overstate them.** Being wrong about
   how often you are wrong undermines the numbers you got right.
4. **Verify against the artifact, never against your own account of it.**
