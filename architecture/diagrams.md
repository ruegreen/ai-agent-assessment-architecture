# Architecture Diagrams

Detailed diagrams for the AI agent assessment platform. All diagrams describe design and
control flow; none describe implementation internals.

---

## 1. High-Level System Architecture

```mermaid
graph TB
    subgraph Config["Configuration — all data, schema-validated"]
        RUBRIC["Rubric"]
        PROFILE["Profiles (extends:)"]
        SCEN["Scenario Suites"]
        PERS["Personas"]
        MISSION["Mission<br/>target + ownership + consent"]
    end

    subgraph Plan["Run Planner"]
        FILTER["Filter by tier"]
        EXPAND["Expand pass^k"]
        CAP["Cost cap"]
    end

    subgraph Exec["Execution"]
        RUNNER["Runner"]
        SIM["Simulated User<br/>(model A)"]
        FID["Persona Fidelity Check"]
        ADAPT["Target Adapters"]
    end

    EV[("Evidence Store<br/>transcripts · audio · traces · end state")]

    subgraph Score["Scorers"]
        DET["deterministic"]
        SC["state_check"]
        JUDGE["llm_judge (model B)"]
        AUD["audio"]
        RT["redteam"]
        ATT["attested"]
    end

    subgraph Agg["Aggregator"]
        NORM["Renormalize around<br/>not_assessed / excused"]
        GATES["Gates · caps · status floors"]
        COV["Coverage · provisional<br/>· headline floor"]
    end

    REPORT["Scorecard<br/>HTML · JSON · PDF<br/>+ traceability matrix"]

    RUBRIC --> PROFILE --> Plan
    SCEN & PERS & MISSION --> Plan
    Plan --> RUNNER
    SIM <--> RUNNER <--> ADAPT
    SIM --> FID
    FID -->|"broke character:<br/>excluded + reported"| EV
    RUNNER --> EV --> Score --> Agg --> REPORT
```

---

## 2. From a Check to a Number

```mermaid
flowchart TD
    CHECK["Check<br/>value 0–1 · status<br/>evidence_refs · rationale"]
    CHECK --> METRIC["Metric<br/>weighted mean of checks<br/>status from objective + thresholds"]
    METRIC --> FLOOR{"Scorer pinned<br/>a status floor?"}
    FLOOR -->|yes| PIN["Status cannot be softened<br/>by aggregation"]
    FLOOR -->|no| CAT
    PIN --> CAT["Category<br/>weighted mean of ASSESSED metrics"]
    CAT --> PILLAR["Pillar<br/>weighted mean of assessed categories"]
    PILLAR --> OVERALL["Overall<br/>weighted mean of assessed pillars"]
    OVERALL --> GATE{"Any gate failed?"}
    GATE -->|yes| CAPPED["Cap overall or pillar<br/>at stated ceiling · raise flag"]
    GATE -->|no| COVCHECK
    CAPPED --> COVCHECK{"Coverage check"}
    COVCHECK -->|"below headline_floor"| WITHHOLD["WITHHOLD overall + grade<br/>everywhere.<br/>raw_overall withheld too."]
    COVCHECK -->|"below minimum_coverage"| PROV["Mark PROVISIONAL"]
    COVCHECK -->|ok| BAND["Grade band"]
    PROV --> BAND
```

---

## 3. Four Kinds of Absence

```mermaid
flowchart TD
    Q["A metric produced no score"] --> WHY{"Why?"}
    WHY -->|"tier forbids it · no evidence ·<br/>judge disabled"| NA["not_assessed"]
    WHY -->|"we looked, question<br/>did not apply"| EX["excused"]
    WHY -->|"attempt made, it failed"| ER["errored"]
    WHY -->|"nothing was ever put<br/>in front of the detector"| UN["untested"]

    NA --> NAC["counts AGAINST coverage<br/>excluded from average"]
    EX --> EXC["counts TOWARD coverage<br/>excluded from average<br/>named with its reason"]
    ER --> ERC["a failed attempt,<br/>not an absence of evidence"]
    UN --> UNC["NOT 'passing'<br/>see instrument validation"]
```

> Collapsing any two of these produced a measurable, directional error in the headline
> score — and in every case the error flattered somebody. Merging `excused` into
> `not_assessed` removed a *failing* metric from the coverage denominator, which raises the
> pillar score and reads as the agent improving.

---

## 4. Tiers and Evidence Source

```mermaid
graph TB
    subgraph Tiers["Access Tiers"]
        W["white<br/>traces · tools · state"]
        G["gray<br/>channel + platform transcripts"]
        B["black<br/>the conversation only"]
    end

    subgraph Sources["Evidence Source — weakest wins"]
        S1["read from the target's own records"]
        S2["inferred from the conversation"]
        S3["observed by attacking it"]
        S4["stated by the operator, not verified"]
    end

    W --> S1 & S2 & S3
    G --> S2 & S3
    B --> S2 & S3
    B --> S4
    B -.->|"structurally impossible"| S1

    S1 & S2 & S3 & S4 --> WEAK["Metric reports the<br/>WEAKEST source among its checks"]
```

A metric fed partly by transcript inference is **not** a records read, however many of its
other checks were. And a black-box assessment cannot claim a records source at all — the
adapter has no state channel, so no check can return one.

---

## 5. Comparing Two Scorecards

Only one of these two comparisons supports the claim *"this is what black-box access costs
you."*

```mermaid
flowchart TD
    PAIR["Two scorecards"] --> CHECK{"Both carry tier-replay markers<br/>naming each other<br/>AND transcript digests match?"}
    CHECK -->|yes| SAME["SAME CONVERSATIONS<br/>Agent identical by construction.<br/>Every difference = cost of missing access."]
    CHECK -->|no| DIFF["DIFFERENT RUNS<br/>Difference = access, OR the agent<br/>behaving differently, OR both.<br/>At low attempt counts the<br/>second usually dominates."]
    SAME --> STATE["Comparison view states which kind<br/>it is showing — detected, not assumed"]
    DIFF --> STATE
```

> Two separate runs once had the same agent scoring as having completed the task at white
> box, where its records showed the action, and as having failed it at black box, where the
> conversation judged had ended in an escalation. Both scorings were right about their own
> run. Presenting that gap as the cost of black-box access would have been inventing a
> finding.

---

## 6. Judge Characterisation — Two Independent Faults

```mermaid
graph TB
    JUDGE["LLM Judge"]

    subgraph Cal["Calibration — agrees with a PERSON?"]
        BLIND["Blind human labels<br/>(labeller never sees judge's band)"]
        CEIL["Human-to-human kappa<br/>= the CEILING"]
        JK["Judge-to-human kappa"]
        HEAD["headroom = ceiling − judge"]
    end

    subgraph Con["Consistency — agrees with ITSELF?"]
        REPEAT["Same transcripts, same prompt,<br/>same model, same temperature<br/>× 3"]
        VAR["Per-check variance<br/>and flip rate"]
    end

    JUDGE --> Cal & Con
    BLIND --> CEIL & JK --> HEAD
    REPEAT --> VAR

    HEAD -->|"near zero"| RUBRIC["The RUBRIC needs work,<br/>not the judge"]
    VAR -->|"high"| RUBRIC
    JUDGE -->|"uncalibrated"| PROV["Scorecard cannot be<br/>non-provisional"]
```

**Neither measure substitutes for the other.** A judge can be perfectly consistent and
consistently wrong, and it can be well calibrated on average while deciding individual
cases close to a coin-flip — the average hides that, and only consistency shows it.

Measured directly: re-scoring stored assessments with nothing changed but the request moved
0.2 to 1.4 check results per conversation.

---

## 7. Ownership and the Run Decision

```mermaid
flowchart TD
    START["Mission submitted"] --> OWN{"Ownership declared?"}
    OWN -->|no| R1["REFUSE — no default, ever"]
    OWN -->|yes| KIND{"Which kind?"}

    KIND -->|own| OWNPATH["Full assessment permitted"]
    KIND -->|authorized| CONSENT{"Consent record<br/>complete and unexpired?"}
    KIND -->|public_unauthorized| PUB["Experience only<br/>volume too low to grade"]

    CONSENT -->|"missing a field"| R2["REFUSE — a record missing<br/>a field is not a weaker record"]
    CONSENT -->|expired| R3["REFUSE — consent expired"]
    CONSENT -->|ok| AUTHPATH["Assessment permitted"]

    AUTHPATH --> PROBE{"Probing requested?"}
    PUB --> NOPROBE["NEVER probed. No override."]
    PROBE -->|"separate written<br/>authorization present"| YESPROBE["Probing permitted"]
    PROBE -->|"no separate authorization"| NOPROBE2["Probing refused"]

    OWNPATH --> HANDOFF{"Handoff policy"}
    AUTHPATH --> SAO["stop_at_offer — MANDATORY"]
    PUB --> SAO
    HANDOFF -->|"named unstaffed<br/>test queue confirmed"| ALLOW["allow_transfer available"]
    HANDOFF -->|otherwise| SAO
```

---

## 8. Stop at Offer

```mermaid
sequenceDiagram
    participant S as Simulated Caller
    participant A as Target Agent
    participant R as Runner
    participant E as Evidence

    S->>A: conversation proceeds
    A->>S: "Would you like me to transfer you?"
    R->>E: record TRANSFER_OFFERED
    R->>R: end conversation
    Note over R: The offer WAS the measurement.<br/>Escalation-on-request: SCORED.<br/>Context-transfer: not_assessed,<br/>reason = "policy is stop_at_offer"

    alt transfer fires without confirmation
        R->>R: HALT — send nothing further
        R->>E: record FINDING + transcript
        Note over R: A live queue now holds a test<br/>conversation. Continuing to talk<br/>into it is the only way to make<br/>it worse.
    end
```

---

## 9. Voice Turn-Taking Measurement

```mermaid
graph TB
    subgraph Stream["Full-duplex bidirectional stream"]
        T1["Track: caller (ours)"]
        T2["Track: agent (target)"]
        CLK["ONE monotonic clock"]
    end

    T1 & T2 --> CLK

    CLK --> M1["Response latency =<br/>agentFirstAudio − callerSpeechEnd<br/>− <b>our TTS first-byte delay</b>"]
    CLK --> M2["Barge-in =<br/>playback stop − caller speech start"]
    CLK --> M3["Talk-over =<br/>intersection of the two<br/>speech interval sets"]

    M3 -.->|"impossible on a<br/>mixed single track"| T2

    subgraph Quality["Speech quality"]
        REL["predicted MOS:<br/>objective = relative<br/>vs same-path baseline"]
        ABS["absolute MOS:<br/>white-box, pre-telephony<br/>— a SEPARATE metric id"]
        NOBASE["no baseline yet<br/>→ not_assessed"]
    end

    REL --> NOBASE
```

> Our own synthesized-speech first-byte delay sits **inside** the interval that would
> otherwise be attributed to the target, so it is measured and subtracted. Reporting it as
> the agent's latency is a measurement error, not a conservative estimate.

---

## 10. Instrument Validation — Positive Controls

```mermaid
flowchart TD
    RANGE["Test range: deliberately flawed agent"] --> V["vulnerable variant<br/>(flaws present)"]
    RANGE --> H["hardened variant<br/>(same flaws removed)"]

    V --> PC{"Planted flaw exercised<br/>AND caught?"}
    PC -->|yes| WORKS["Positive control WORKED —<br/>measure demonstrably discriminates"]
    PC -->|no| UNTESTED["Metric is UNTESTED,<br/>not passing"]

    H --> FIRED{"Metric fired on<br/>hardened variant?"}
    FIRED -->|"yes, and control worked"| FALSE["FALSE FINDING<br/>our bug — fails strict gate"]
    FIRED -->|"yes, control never ran"| OBS["OBSERVATION about the<br/>hardened agent — usually true,<br/>does not fail the gate"]
    FIRED -->|no| CLEAN["Clean — <b>only meaningful if</b><br/>the positive control worked"]

    WORKS --> CLEAN
    UNTESTED -.->|"must not render<br/>the same as clean"| CLEAN
```

> *"We looked and found nothing"* and *"nothing was ever put in front of it"* must not
> render the same way. A range report showing forty metrics clean when only twelve had a
> flaw put in front of them is the most dangerous possible output, because it reads as
> thorough.
