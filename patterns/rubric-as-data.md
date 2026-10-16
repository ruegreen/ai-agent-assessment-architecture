# Rubric as Data Pattern

Every category, metric, weight, threshold, gate and standards mapping lives in YAML and is
validated against a JSON Schema on load. No scoring decision is expressed in code.

---

## Why This Is the Foundation

"Vendor-neutral" is easy to claim and hard to mean. It means something here only because a
different industry, a different risk appetite or a different regulator is a **profile
overlay** rather than a fork of the tool.

If any weight, threshold or gate lived in code, every customer would get a branch.

---

## The Metric Record

```
Metric
  id                  snake_case, stable, NEVER renamed — deprecated instead
  scorer              deterministic | state_check | llm_judge | audio | redteam | attested
  objective           maximize | minimize | target | relative
  thresholds          pass/warn bands, or target + tolerance, or a baseline reference
  weight              within its category
  applies_to_tiers    [white, gray, black] — which access levels can measure it
  evidence            what artifact backs the value
  standards[]         control clauses this provides evidence toward — or explicit []
```

Four requirements enforced at load:

1. **Every metric declares its tiers.** A metric that does not say which access levels it
   applies to cannot be correctly skipped, so it cannot be honestly scored.
2. **Every metric declares `standards` or an explicit empty list.** Silence is ambiguous
   between "maps to nothing" and "nobody checked."
3. **IDs are permanent.** A renamed metric breaks every stored scorecard's comparability.
   Deprecation is the only exit.
4. **Every scorer returns the same shape:** `value`, `score`, `status`, `evidence_refs`,
   `rationale`. A scorer that cannot produce a rationale cannot be audited.

---

## Objectives Are Not All Thresholds

```
status(metric, value):
    match metric.objective:
        maximize  -> band(value, metric.pass_at, metric.warn_at)
        minimize  -> compare(value, metric.target, metric.limit)
        target    -> within(value, metric.target, metric.tolerance)   # humanlike
        relative  -> compare(value, baseline(metric.reference))       # same-path
```

`target` exists because **humanlike is a target, not a maximum.** An agent can be too
effusive, too casual, too eager. Scoring those dimensions with `maximize` instructs an
agent to become a caricature.

`relative` exists because some measures are meaningless in absolute terms — see
[voice-measurement.md](voice-measurement.md).

---

## Profile Inheritance

```
resolve(profileId, registry):
    chain = []                       # profile -> parent -> ... -> base
    while profileId:
        p = registry[profileId]; chain.append(p); profileId = p.extends
    merged = {}
    for p in reversed(chain):
        merged = mergeById(merged, p)          # partial overlay, keyed by id
    validate(merged, SCHEMA)                   # the MERGED result, not the fragments
    normalizeWeights(merged)
    return merged
```

Two details that are easy to get wrong:

- **Validate the merged result, not each fragment.** A profile is a partial overlay and is
  invalid on its own by design. Validating fragments either rejects every legal profile or
  requires a second, weaker schema that then drifts.
- **Gates merge by id**, so a profile can add a gate *or disable an inherited one*. Without
  id-keyed gate merging, a profile can only ever be stricter than its parent, which is not
  how risk appetite actually varies.

---

## Gates, Caps and Status Floors

Three mechanisms, deliberately distinct.

| Mechanism | Effect | Set by |
|---|---|---|
| **Gate** | A pass/fail requirement on one metric. Failing caps the overall or a pillar at a stated ceiling and raises a flag | The rubric |
| **Cap** | The ceiling a failed gate imposes | The rubric |
| **Status floor** | A scorer pins a status the aggregation **must not soften** | The scorer |

The status floor is the subtle one. A count metric's own band might call one finding a
warning — but one *confirmed critical* security finding is a failure, and the scorer knows
that in a way the band cannot express. So the scorer pins the floor and says it did.

Gates are also what make the tool a governance instrument rather than a score generator.
A disclosure gate capping the overall at 49 and flagging critical is not a scoring
preference; it is a statement that nothing else on the scorecard matters if that failed.

---

## What Stays in Code

Not everything belongs in data. The boundary is the question *"would a customer reasonably
configure this?"*

| In data | In code |
|---|---|
| Categories, metrics, weights | How a weighted mean renormalizes around `not_assessed` |
| Thresholds, objectives, tolerances | The four objective comparison functions |
| Gates, caps, grade bands | The coverage floor machinery and the headline withholding rule |
| Standards mappings | That a scorecard says "evidence toward", never "compliant with" |
| Tier applicability | That a black-box run cannot claim a records source |

**The right-hand column is not configurable, and that is deliberate.** A rubric that could
switch off the coverage floor, or relabel attested evidence as tested, would let a customer
configure away the honesty the tool exists to provide.
