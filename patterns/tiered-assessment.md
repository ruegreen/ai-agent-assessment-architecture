# Tiered Assessment Pattern

Access determines what is knowable. Making that structural — rather than a footnote — is
the difference between an assessment and a guess with a number on it.

---

## The Three Tiers

| Tier | Access | What becomes knowable |
|---|---|---|
| **white** | Traces, tool calls, internal state | Everything the rubric declares, including trajectory and tool-level correctness |
| **gray** | Channel plus platform transcripts | Most of it; trajectory partially |
| **black** | The conversation only | Outcomes, experience, security probes — governance becomes *attested* |

Every metric declares `applies_to_tiers`. Out-of-scope metrics report `not_assessed`
**with the tier named as the reason**, and they count against coverage.

That last clause is the whole design. The missing access is made *visible* rather than
hidden. A black-box scorecard is supposed to look thinner than a white-box one, and a
reader is supposed to be able to see exactly where the thinness is.

---

## Evidence Source Travels With Every Value

```
EvidenceSource = {
  RECORDS:   "read from the target's own records",      # white-box only
  INFERRED:  "inferred from the conversation",          # all a channel-only run can do
  OBSERVED:  "observed by attacking it",                # a probe result
  ATTESTED:  "stated by the operator, not verified",    # someone's word
}
```

### Weakest source wins

```
sourceOf(metric):
    return weakest(check.source for check in metric.checks)
```

A metric fed partly by transcript inference **is not a records read**, however many of its
other checks were. Same weakest-link logic as the scoring: a composite claim cannot be
stronger than its weakest component.

### A black-box assessment cannot claim a records source

This is enforced structurally, not by convention. The adapter at black box has no state
channel, so no check can return `RECORDS`, so no metric can report it.

**A tool that overstates its own access is worse than no tool.** It produces a conclusion
that looks identical to a well-founded one.

---

## Attestation Is Evidence of a Claim, Not of a Control

At black box, governance controls cannot be tested. They become *attested* — a
questionnaire flow where the operator states what is in place and uploads supporting
evidence.

An attested control is scored, labelled `ATTESTED`, and never presented as equivalent to a
tested one. The scorecard distinguishes:

- **tested** — we exercised it and observed the result
- **attested** — somebody told us, and we recorded who and when

Both are legitimate inputs to a governance assessment. Rendering them the same way is not.

---

## Comparing Across Tiers — Only One Way Is Honest

Two scorecards side by side mean one of two things, and they are not interchangeable.

| Kind | What a difference means |
|---|---|
| **Same conversations, different access** | One set of stored transcripts scored at two tiers. The agent is identical on both sides *by construction*, so every difference is what the missing access cost. |
| **Different runs** | Two separate assessments. A difference can be the access, the agent behaving differently that time, or both — and at low attempt counts the second usually dominates. |

Only the first supports the claim "this is what black-box access costs you."

```
tierReplay(storedAssessment):
    white = score(storedAssessment, withhold: none)
    black = score(storedAssessment, withhold: [end_state, traces, tool_calls])
    return comparison(white, black, marker: SAME_CONVERSATIONS)
```

The comparison view **states which kind it is showing, detected from the scorecards rather
than assumed.** A pair counts as the same conversations only if both sides carry a
tier-replay marker naming each other **and their transcript digests match.**

### Why that check is not pedantry

Two separate runs once had the same agent scoring as having completed the task at white
box — its records showed the action — and as having failed it at black box, where the
conversation judged had ended in an escalation.

Both scorings were right about their own run. Presenting that gap as the cost of black-box
access would have been **inventing a finding**.

---

## Standards Traceability

Every metric maps to the control clauses it provides evidence toward, across published
frameworks: the OWASP Agentic Top 10, NIST AI RMF, ISO/IEC 42001, the EU AI Act, and
service-quality standards for contact handling.

Three rules govern the mapping:

1. **A scorecard says "evidence toward" a control. It never says "compliant with."** No
   test suite can make a compliance claim, and a tool that does is selling something.
2. **Every mapping carries a `reviewed` flag, and mappings are drafts until reviewed.** The
   scorecard footer states the mapping status.
3. **Unverifiable mappings are listed as unverifiable.** Seven of thirty-seven cannot be
   independently checked because the standards texts are paywalled. They are named rather
   than quietly asserted.

### Coverage of a framework is reported, not assumed

A security pillar that ran probes against six of ten categories in a framework reports
six of ten. It does not report a clean result against the framework.

**A clean security pillar on a never-probed target must read as "never probed", not as
"nothing found."** Those are different claims and only one of them is true.
