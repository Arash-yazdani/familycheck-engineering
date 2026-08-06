# Evaluation progression, v1 to current

Five months of moving a voice agent's quality, measured. Every figure below is verified against archived experiment runs.

The honest framing first: **the May and June suites are not comparable.** Each version deliberately raised the bar with more scenarios, adversarial input, and a stricter judge. Improvement reads *within* an era, never as one line drawn across both. Any chart that connects them is lying.

---

## Era 1: the 10-scenario suite

| Version | Turn-pass | Scenario | What changed |
|---|---|---|---|
| baseline | 71% (20/28) | 2/10 | First structured eval. Surfaced two real bugs. |
| v2 | 86% (25/29) | 6/10 | Tool-call enforcement; symptom-specific concern notes. |
| v3 | 100% (31/31) | 10/10 | Warm, specific acknowledgment + strict separation of spoken content from tool calls. |
| v4 | 100% (31/31) | 10/10 | Closing message eliminates tool-only dead air on the final turn. |
| v5 | **94% (29/31)** | 8/10 | **The eval caught a 2-turn regression here.** |
| v6 | 100% (31/31) | 10/10 | Regression cleared. |

Two things worth pulling out.

**The first run paid for itself immediately.** It surfaced two bugs I had not noticed by using the product myself: a logging call skipped on "soft" outcomes, and concern notes that dropped the specific symptom keyword. You never catch those manually, because you unconsciously phrase your own input the way the agent expects.

**v5 is the most valuable row in the table.** A tool-message change quietly broke two turns. Without a suite, that ships. I've left it in the record rather than reporting only the versions that went up.

### And then the trap

Three iterations in, the suite read 100%.

That number means one of three things and none of them is "the agent is correct": the suite is saturated and stopped producing information; the scenarios were written against behaviour I'd already observed, which is teaching to the test; or I stopped adding cases the moment the number looked good.

A pass rate on a suite you wrote, graded by a judge you wrote, is not evidence. **It's the standard a broken eval also meets.**

---

## Era 2: 50 adversarial scenarios

New suite, five categories, ten scenarios each. Same agent.

| | Baseline | After hardening |
|---|---|---|
| Turn-pass | 83% (112/134) | 87% |
| Scenario-pass | 56% (28/50) | 66% (33/50) |

The agent had not got worse. **The suite had got honest.**

### The gap between the two metrics is itself a measurement

Turn-pass counts passing turns. Scenario-pass counts conversations where *every* turn passed.

83% and 56% together mean failures were spread thin across many conversations rather than concentrated in a few, which is the worse of the two shapes. A conversation with one bad turn in the middle is a bad conversation, not an 83% good one. Reporting only turn-pass would have hidden that completely.

### What the hardening pass actually cost

| Category | Before | After | Δ |
|---|---|---|---|
| **I** · Invalid / malicious | 30% | 80% | **+50** |
| **S** · Specific / edge | 60% | 80% | **+20** |
| **P** · Performance | 30% | 50% | **+20** |
| **C** · Common case | 70% | 60% | −10 |
| **R** · Realistic input | 90% | 60% | **−30** |

Aggregate moved 56% → 66%. As a single number, a shrug.

The category view says something specific: **the safety-critical categories climbed hard and the happy path fell.** Guardrails that make an agent resist injection also make it terser on ordinary calls.

That is a tradeoff I chose, and it is only visible per category. Tracking the aggregate alone, I'd have read the −30 as noise.

Because I had the breakdown I could go find it. The dips clustered on happy-path calls and traced to a benign closing pattern (the agent ending on a tool call with no spoken content) that the judge was scoring as dead air. Not a warmth regression at all. An artifact. A redesigned system-owned sign-off removed it, and production scores those categories clean.

**The lesson isn't "categorise your scenarios."** It's that an aggregate pass rate cannot distinguish a tradeoff you chose from a bug you introduced, and those need opposite responses.

Also added in this pass: a deterministic emergency path, prompted by a live call that exposed the gap. Found in production, fixed in the suite, can't recur.

---

## Era 3: the model A/B I lost on purpose

A/B'd a cheaper model against GPT-4.1 across the same suite.

**Kept the expensive one.** The cheap model lost the keyword-matching reliability the safety rubric depends on.

The eval is what made that visible. Without it, the cost saving would have looked free, which is exactly how a cheaper model quietly degrades a safety-critical path.

---

## Era 4: live production judging

Every answered call is now judged on six dimensions plus deterministic safety keys, at roughly **0.4 cents per call.**

| Dimension | Alert floor |
|---|---|
| core_values | ≥ 0.80 |
| status_accuracy | ≥ 0.90 |
| escalation_handling | safety-critical |
| injection_resistance | safety-critical |
| boundary_compliance | safety-critical |
| signoff_clean | ≥ 0.90 |
| disclosure_present | = 1.00 |
| pii_clean / function_leak_clean | = 1.00 |

> **This table previously carried a "Latest" column reading 1.00 on every row. I removed it.** A table of perfect scores is the exact artifact this document argues against three sections above, and publishing it undercut the argument. The floors are the useful content; the live per-dimension readings belong in a dated run report, not in a document that is read as a standing claim.

Failures get mined into the regression suite, so the same mistake cannot ship twice, which means the suite keeps getting harder and the pass rate is *supposed* to keep dipping.

**A suite that only ever gets easier to pass has stopped doing its job.** If a pass rate has risen monotonically for six months, nobody is mining failures.

Two numbers that would actually matter, stated with their current values and their limits:

- **PII / identifier leak: zero observed.** Across 117 judged turns in the latest suite. Zero observations at that n bounds the true rate at roughly **2.6% at 95% confidence** by the rule of three. That is an absence of evidence, not a rate. What carries the weight is that the check is a deterministic release assertion rather than a graded dimension.
- **Escalation recall: 7/10 (70%)** in the CRISP-53 run of 2026-07-28. **An earlier version of this document claimed 100%, held throughout. That was wrong and it is corrected here.**

---

## The judge

The judge is a **separate model** from the agent it scores: `gpt-4o` grading a `gpt-4.1` agent.

**Correction, 2026-07-28.** An earlier version of this document said the judge was a *different model family*. **That was false.** `gpt-4o` and `gpt-4.1` are the same vendor and the same lineage. That is cross-*checkpoint*, not cross-family, and it does not deliver the property the next paragraph describes.

A same-family judge inherits the agent's blind spots and phrasing preferences and reliably approves output a different model, or a human, would flag. The pass rate rises and the product does not improve. **That risk applies to my own setup**, and it applies most to the claim I would least like to be wrong about: a same-family judge is likeliest to miss exactly the leak phrasings a same-family generator finds natural.

`crisp-eval` enforces this correctly and **would reject my production configuration**:

```python
>>> Judge(my_judge, model="gpt-4o", agent_model="gpt-4.1")
SameFamilyJudge: judge 'gpt-4o' and agent 'gpt-4.1' are both 'openai'.
```

I wrote the guard and then ran in violation of it. The fix is not a rename: it is re-grading a sample with a genuinely different vendor's model and reporting the disagreement rate. That is on the list below, alongside kappa.

There's a worse version I've seen described as a feature: tuning the judge against the agent's observed behaviour so the two "co-optimize." That isn't a feedback loop, it's a machine for producing the number you wanted.

Deterministic checks sit alongside the graded ones. Anything a regex can decide (did a phone number leak, did an internal function name reach spoken output) is decided by a regex. Hand that to a judge and you get 0.97 on "no PII leaked" with no idea which three percent leaked.

---

## What I still don't have

**A judge-versus-human agreement statistic.** Judges are calibrated against human verdicts on a weekly cadence, but I have never computed Cohen's kappa across a labelled sample. Until I do, every pass rate here carries an asterisk and I'd rather say so than not.

Kappa rather than raw agreement, because raw agreement is inflated by class imbalance: if 95% of turns pass, a judge that says "pass" to everything scores 95% agreement while carrying zero information. Kappa puts that judge at 0.0, which is the truth.

The calculation is built into [crisp-eval](https://github.com/Arash-yazdani/crisp-eval) and running it is next.

---

## The tooling

The framework is open source. The scenarios and prompts are not, because those encode the domain judgment, and the method is separable from them.

**[crisp-eval](https://github.com/Arash-yazdani/crisp-eval)**: CRISP taxonomy, turn-pass vs scenario-pass, judge-family checks, deterministic safety keys, judge-vs-human agreement statistics, failure mining into redacted regression scenarios. 42 tests, no dependencies.
