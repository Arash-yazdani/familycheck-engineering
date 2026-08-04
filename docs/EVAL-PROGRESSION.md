# Evaluation progression, v1 to current

Five months of moving a voice agent's quality, measured. Every figure below is verified against archived experiment runs.

The honest framing first: **the May and June suites are not comparable.** Each version deliberately raised the bar with more scenarios, adversarial input, and a stricter judge. Improvement reads *within* an era, never as one line drawn across both. Any chart that connects them is lying.

---

## Era 1 — the 10-scenario suite

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

## Era 2 — 50 adversarial scenarios

New suite, five categories, ten scenarios each. Same agent.

| | Baseline | After hardening |
|---|---|---|
| Turn-pass | 83% (112/134) | 87% |
| Scenario-pass | 56% (28/50) | 66% (33/50) |

The agent had not got worse. **The suite had got honest.**

### The gap between the two metrics is itself a measurement

Turn-pass counts passing turns. Scenario-pass counts conversations where *every* turn passed.

83% and 56% together mean failures were spread thin across many conversations rather than concentrated in a few — the worse of the two shapes. A conversation with one bad turn in the middle is a bad conversation, not an 83% good one. Reporting only turn-pass would have hidden that completely.

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

Because I had the breakdown I could go find it. The dips clustered on happy-path calls and traced to a benign closing pattern — the agent ending on a tool call with no spoken content — that the judge was scoring as dead air. Not a warmth regression at all. An artifact. A redesigned system-owned sign-off removed it, and production scores those categories clean.

**The lesson isn't "categorise your scenarios."** It's that an aggregate pass rate cannot distinguish a tradeoff you chose from a bug you introduced, and those need opposite responses.

Also added in this pass: a deterministic emergency path, prompted by a live call that exposed the gap. Found in production, fixed in the suite, can't recur.

---

## Era 3 — the model A/B I lost on purpose

A/B'd a cheaper model against GPT-4.1 across the same suite.

**Kept the expensive one.** The cheap model lost the keyword-matching reliability the safety rubric depends on.

The eval is what made that visible. Without it, the cost saving would have looked free — which is exactly how a cheaper model quietly degrades a safety-critical path.

---

## Era 4 — live production judging

Every answered call is now judged on six dimensions plus deterministic safety keys, at roughly **0.4 cents per call.**

| Dimension | Latest | Alert floor |
|---|---|---|
| core_values | 1.00 | ≥ 0.80 |
| status_accuracy | 1.00 | ≥ 0.90 |
| escalation_handling | 1.00 | safety-critical |
| injection_resistance | 1.00 | safety-critical |
| boundary_compliance | 1.00 | safety-critical |
| signoff_clean | 1.00 | ≥ 0.90 |
| disclosure_present | 1.00 | = 1.00 |
| pii_clean / function_leak_clean | 1.00 | = 1.00 |

Failures get mined into the regression suite, so the same mistake cannot ship twice — which means the suite keeps getting harder and the pass rate is *supposed* to keep dipping.

**A suite that only ever gets easier to pass has stopped doing its job.** If a pass rate has risen monotonically for six months, nobody is mining failures.

Held throughout, across every era: **0% PII leak. 100% escalation recall.** Those are the two numbers that would actually matter.

---

## The judge

**The deployed judge is same-family: GPT-4o grading GPT-4.1.** Same vendor, same
lineage. That is a real limitation of this system and I'd rather name it than
imply otherwise.

The concern with same-family judging is that the judge inherits the agent's
blind spots and phrasing preferences, and approves output a different model — or
a human — would flag. The pass rate rises and the product does not improve.

So rather than assert the judge is fine, I measured it against a human on a
labelled sample. That is the [agreement study](agreement-study-2026-07-30.md)
below: the shipped judge agrees with a human rater at **AC1 0.958 / 0.843 /
1.000** across three dimensions, and where the two differ the judge is
**lenient**, not random — which is the direction a same-family judge is
theoretically expected to fail in, now with a number on it.

There's a worse version I've seen described as a feature: tuning the judge against the agent's observed behaviour so the two "co-optimize." That isn't a feedback loop, it's a machine for producing the number you wanted.

Deterministic checks sit alongside the graded ones. Anything a regex can decide — did a phone number leak, did an internal function name reach spoken output — is decided by a regex. Hand that to a judge and you get 0.97 on "no PII leaked" with no idea which three percent leaked.

---

## Era 5 — grading the judge

Every number above is a judge grading an agent with nobody grading the judge.
On 2026-07-30 I hand-labelled all 53 scenarios blind and compared.

**Full study: [docs/agreement-study-2026-07-30.md](agreement-study-2026-07-30.md)** ·
raw label pairs: [`docs/agreement/pairs.csv`](agreement/pairs.csv)

| Dimension | n | observed agreement | Gwet's AC1 [95% CI] |
|---|---|---|---|
| core_values | 50 | 96.0% | **0.958** [0.890, 1.000] |
| status_accuracy | 51 | 86.3% | **0.843** [0.703, 0.938] |
| escalation_handling | 49 | 100% | **1.000** [1.000, 1.000] |

Three things this produced that the pass rates could not.

**Kappa did the opposite of what I predicted.** What this doc said before the
study ran, verbatim:

> Kappa rather than raw agreement, because raw agreement is inflated by class
> imbalance: if 95% of turns pass, a judge that says "pass" to everything scores
> 95% agreement while carrying zero information. Kappa puts that judge at 0.0,
> which is the truth.

The reasoning is right and the conclusion is wrong. On real data the imbalance
was severe enough to break kappa itself: at 96–98% pass on both margins the
chance-agreement term exceeds 0.96, kappa's denominator collapses, and it lands
**negative** (−0.020, −0.035) with up to 13% of bootstrap replicates undefined —
on the same tables that yield AC1 0.958. That is the Feinstein–Cicchetti
paradox, and it is why Gwet's AC1 was pre-registered as primary before any
labelling started. Quoted rather than quietly deleted.

**The judge is lenient, one-sidedly.** In 7 of 9 disagreements the judge passed
a call the human failed — concentrated on `logged_status` for adversarial
probes. The human sided with the suite's expectation against the model's
logging, which turns the long-standing "I-category at 60%" from a
suite-vs-model argument into a human-validated defect.

**It validates the judge's pass behaviour and almost nothing else.** The
both-fail cell is **empty in every dimension** — the two raters never once
agreed a call failed. With failure that rare, this cannot distinguish a
slightly-lenient judge from one blind to a rare failure mode. A
failure-enriched replication is the follow-up, not a victory lap.

Single rater, who also built the system and wrote the rubric, on synthetic
transcripts, with a tie-break that biases toward agreement. **These are upper
bounds**, and the study says so in its own limitations section.

---

## The tooling

The framework is open source. The scenarios and prompts are not — those encode the domain judgment, and the method is separable from them.

**[crisp-eval](https://github.com/Arash-yazdani/crisp-eval)** — CRISP taxonomy, turn-pass vs scenario-pass, cross-family judge enforcement, deterministic safety keys, kappa calibration, failure mining into redacted regression scenarios. 42 tests, no dependencies.
