# Assessment Ethics Pattern

The platform is pointed at agents the operator does not own. You cannot rate-limit someone
else's chat widget, put a test queue behind their contact centre, or recall a conversation
that reached a person.

**So the tool's own behaviour is the protection.** These are refusals in code, not
conventions in a runbook.

---

## Ownership Is Declared, and There Is No Default

| Kind | What it is | What it permits |
|---|---|---|
| `own` | Our agent, our tenant | Everything, including a completed transfer into a **named** test queue |
| `authorized` | Someone else's agent, with a recorded consent record | Assessment; probing only under **separate written** security-testing authorization |
| `public_unauthorized` | An agent we can reach and have no permission to test | Experience only, very low volume. **Never probed. No override.** |

```
plan(mission):
    if mission.ownership is None:
        refuse("ownership is not declared")          # no default, ever
    ...
```

**Why there is no default:** the safe-sounding assumption — *"it's probably ours"* — is the
one that causes the harm. A mission that does not say cannot run.

Note that `authorized` permits *assessment* but not *probing*. Permission to be measured
and permission to be attacked are different permissions, and conflating them is how a
consented assessment becomes an unauthorised penetration test.

---

## A Consent Record Is a Record, Not a Checkbox

```
ConsentRecord
  grantedBy      name AND role of the person who permitted it
  grantedAt      when
  expiresAt      until when
  scope          what was permitted
  method         how it was given
```

**Every field is required**, because a record missing one is not a weaker record:

- without a name, **there is nobody who gave it**
- without dates, it **cannot be shown to still apply**
- without knowing how it was given, **there is nothing to produce later**

```
if consent.expiresAt < now():
    refuse("consent expired " + consent.expiresAt)
```

Consent expires, and an expired record refuses the run rather than warning about it.

### What travels and what does not

Ownership appears on **every** scorecard: the kind, the organisation where there is one,
whether the evidence is third-party, the handoff policy in force, the retention period, and
whether probes were permitted at all.

**The consent's own wording does not travel.** A scope can name their people, their systems
and their ticket numbers — and a scorecard is a document that gets forwarded.

> A reader who does not know whose data they are holding cannot handle it correctly.

---

## A Test Conversation Never Reaches a Live Human

`stop_at_offer` is the default and is **mandatory** for `authorized` and
`public_unauthorized`.

```
onTransferOffer(conversation):
    record(conversation, evidence: TRANSFER_OFFERED)
    end(conversation)                                 # the offer was the measurement
```

**Why the offer is sufficient:** whether the agent escalates when asked is answered by
whether it offered. An agent that says *"would you like me to transfer you?"* has
escalated. Whether the transfer then completes is a property of the platform, not of the
agent's judgement.

Confirming it adds nothing except a person on the other end, waiting to help a customer who
does not exist.

### The metrics this changes, and how

| Metric | Under `stop_at_offer` |
|---|---|
| Escalation on request | **Scored.** The offer answers it. |
| Context transfer across handoff | **`not_assessed`**, reason: *"handoff policy is stop_at_offer"* |

`not_assessed`, **never failed.** The reason is our policy, not the agent's behaviour, and
scoring it as a failure would blame the agent for a conversation we chose to end.

The caveat names the policy explicitly rather than leaving an unexplained blank, so a
reader is not left to infer a gap in the agent from a gap in the report.

### This is not a gap to be closed by relaxing the policy

The obvious response — allow the transfer, measure the thing — trades a metric for a
person's time. On our own agent, with a declared test queue that has nobody behind it, that
trade is available and `allow_transfer` makes it. On anybody else's it is not.

**No scorecard is worth a customer waiting behind a conversation we started as a test.**

`allow_transfer` requires a **named** queue the operator has confirmed is unstaffed. Named
rather than ticked, so whoever reads the mission later can go and check the claim.

### If a transfer fires unconfirmed

```
onUnconfirmedTransfer(conversation):
    halt()                                   # send nothing further
    record(FINDING, "transferred without confirmation", transcript)
```

Stop immediately. Whatever queue it landed in now has a live conversation in it, and the
only thing that makes that worse is continuing to talk into it.

---

## Pacing Loosens Only as the Claim Strengthens

Defaults get gentler as the claim on the target gets weaker: fewer attempts and longer
delays for `authorized`; `public_unauthorized` is deliberately **too low a volume to
grade** — enough to characterise an experience, not to score one.

That is an intentional refusal to produce a number. A confident score on an agent nobody
gave permission to assess is the output most likely to be misused.

---

## The Target's Cost Is Not Our Cost

**Every cost estimate shows the target-side conversation count, labelled as usage billed to
the agent's owner.**

Running an assessment against someone else's agent spends their money — model tokens,
platform minutes, possibly per-interaction licensing. Showing only our own spend presents
the assessment as cheaper than it is and hides a cost we imposed on somebody else.

---

## Evidence Belongs to the Organisation It Came From

| Control | Rule |
|---|---|
| Retention | Per-mission, and short by default |
| Separation | Missions kept separated per organisation |
| Marking | Exports marked confidential, with the organisation's name |
| Disclosure | **A security finding on someone else's agent is marked for disclosure to its owner only** |

Transcripts and retrieved knowledge-base content from an `authorized` or
`public_unauthorized` mission are that organisation's confidential material, not ours,
whatever our assessment concluded about it.

---

## Why All Of This Is in the Code

The governing observation: **a rule with no mechanism behind it is one distracted moment
from being broken** — and here the thing on the other side of that moment is a real
customer waiting for help.

Each of the above is a refusal at a boundary rather than a line in a runbook:

- Undeclared ownership refuses the run.
- Expired consent refuses the run.
- `public_unauthorized` refuses probing, with no override flag.
- An unnamed test queue refuses `allow_transfer`.
- An unconfirmed transfer halts the conversation.

The one genuine operating rule that remained unmechanised — never dialling while another
tool might hold the shared outbound number — was converted into a check for the same
reason, after it became clear that a shared number cannot be in two calls at once and the
second attempt fails rather than queueing.
