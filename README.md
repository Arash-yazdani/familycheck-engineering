# FamilyCheck — engineering case study

A production voice AI agent that places real phone calls, classifies what it hears, and alerts a human when something is wrong.

**The source is private.** This repo is the case study: architecture, the decisions behind it, what the evaluation numbers actually did over five months, and the two pieces I extracted and open sourced.

Built solo — architecture, backend, frontend, evals, telephony compliance — across 12 sprints.

| | |
|---|---|
| **Live** | [family-checker.com](https://family-checker.com) · private beta |
| **Stack** | Vapi · GPT-4.1 · Deepgram Flux (STT) · Cartesia Sonic 3.5 (TTS) · Twilio · Stripe |
| **Code** | FastAPI backend + React/TypeScript frontend, ~18K LOC, 90+ commits |
| **Open sourced** | [crisp-eval](https://github.com/Arash-yazdani/crisp-eval) · [webhook-outbox-kit](https://github.com/Arash-yazdani/webhook-outbox-kit) |

---

## What it does

An agent calls a patient at a scheduled time, has a short conversation, and classifies the outcome into one of six states:

```
confirmed · deferred · denied · concern · unclear · no-answer
```

If the outcome warrants it, a caregiver gets an SMS within seconds.

The interesting engineering is not the call. It's everything around the call: proving the outcome was recorded faithfully, guaranteeing the alert actually goes out, and knowing whether any of it is working.

### Why six classes and not five

`unclear` is the important one. Most people building an LLM classifier ship a set of confident labels and then spend months confused about why production behaves badly. An explicit abstain path means the system can say "I don't know" instead of picking the nearest confident answer — and "I don't know" routes to a human, which is the correct behaviour when the alternative is guessing about someone's health.

Designing the abstain path first, rather than bolting it on after the first bad incident, is the single decision I'd repeat on any classification system where being wrong is expensive.

---

## Architecture decisions

### Delivery must survive the process dying

The handler has to do two things when a call ends: record the outcome, and alert the caregiver. As two separate operations there's a window where one succeeds and the other doesn't — the database says "missed dose" and nobody is ever told.

The fix is a **transactional outbox**: the intent to send is written into the same database, in the same transaction, as the business fact. One commit. A separate worker drains the queue and retries until delivery succeeds, with exponential backoff, dead-letter quarantine for messages that will never send, and a dedupe key derived from the business fact so a retry can't produce a second text.

Extracted and open sourced as **[webhook-outbox-kit](https://github.com/Arash-yazdani/webhook-outbox-kit)** — 41 tests, no dependencies.

### Inbound webhooks fail closed, across three different schemes

Three vendors post into the system and each signs differently:

| Vendor | Scheme | Signs | Replay-resistant |
|---|---|---|---|
| Voice platform | shared-secret header | nothing | no |
| Twilio | HMAC-SHA1 | URL + sorted form params | no |
| Stripe | HMAC-SHA256 | `timestamp` + raw body | **yes** |

Only one signs a timestamp, so only one resists replay on its own. There is no single generic verifier that handles all three correctly, and the details that break you are specific: Twilio signs the full URL, so a TLS-terminating proxy silently invalidates every signature; Stripe signs raw bytes, so re-serialising parsed JSON fails every time; Stripe's header carries multiple `v1=` values during key rotation, and checking only the first takes an outage.

All verification fails closed. There is no "log a warning and continue" branch, because that branch is the vulnerability.

Also in [webhook-outbox-kit](https://github.com/Arash-yazdani/webhook-outbox-kit).

### Consent is a separate agent with a different definition of correct

The consent call runs as its own assistant with its own tool. It is not a mode of the main agent, because its quality bar is a different question entirely: not "did the turn pass" but **"was consent freely given, and recorded faithfully per channel?"**

Consequences that fell out of that:

- Recorded voice consent **is** the opt-in. An earlier design gated activation behind a "reply YES" step, which carriers reject as consent-as-a-condition.
- Calls and texts are asked separately and tagged separately. A patient who says yes to calls and no to texts stays active for voice and is **never** texted — enforced at a single chokepoint rather than checked at each send site.
- A declined call sets a multi-day block so the system never re-dials a "no."

The general lesson: when a second agent's definition of success is a different question, it needs its own evaluation, not a stretched version of the first one's.

### Trust properties built in, not bolted on

- Verbal consent gates all outreach
- The assistant refuses to disclose phone numbers, account identifiers, or internal IDs — enforced as a test assertion, not as prompt text
- A deterministic emergency path, added after a live call exposed the gap
- Traces are PII-redacted before they leave the system, because patient data does not belong in an observability tool
- TCR-verified A2P 10DLC registration, live in production

The pattern worth naming: **a guardrail that isn't wired to something that fails a build is a wish.** Every safety property above is bound to a check that blocks a release.

---

## Evaluation: what the numbers actually did

Full write-up: **[docs/EVAL-PROGRESSION.md](docs/EVAL-PROGRESSION.md)**

The short version, because it's the least flattering and most useful part:

My first suite went from 71% to 100%. Then I made the suite five times harder and the score fell to 56%.

**The second number was worth more.**

| | |
|---|---|
| First suite (10 scenarios) | 71% → 100% turn-pass |
| Second suite (50 adversarial scenarios) | 83% → 87% turn-pass · 56% → 66% scenario-pass |
| Injection resistance | **30% → 80%** |
| Edge / safety scenarios | 60% → 80% |
| PII leak | **0%, held throughout** |
| Escalation recall | **100%, held** |
| Live production judging | 100% of answered calls, 6 dimensions, **~0.4¢/call** |
| Judge-vs-human agreement | **AC1 0.958 / 0.843 / 1.000** across 3 dimensions |

A 100% pass rate on a suite you wrote, graded by a judge you wrote, means the suite is saturated — not that the agent is correct. The 50-scenario suite is where the real work started.

The scenario taxonomy behind it — **CRISP**: Common, Realistic, Invalid/malicious, Specific/edge, Performance — plus turn-pass vs scenario-pass reporting, deterministic safety keys, and judge-vs-human calibration, is open sourced as **[crisp-eval](https://github.com/Arash-yazdani/crisp-eval)**.

### Grading the judge

Every number above is a judge grading an agent. **FamilyCheck's deployed judge is same-family — GPT-4o grading GPT-4.1** — so rather than assert it was fine, I hand-labelled all 53 scenarios blind and measured.

Observed agreement **96.0% / 86.3% / 100%**, Gwet's AC1 **0.958 / 0.843 / 1.000**. Where the two differ the judge is **lenient, one-sidedly** — 7 of 9 disagreements are the judge passing a call the human failed. Cohen's kappa went *negative* on the same tables, which is the kappa paradox on real data and exactly why AC1 was pre-registered as primary.

The both-fail cell is empty in every dimension, so this validates the judge's *pass* behaviour and barely touches its *fail* behaviour. Single rater who built the system, synthetic transcripts, lenient tie-break — **upper bounds, stated as such**.

**Study: [docs/agreement-study-2026-07-30.md](docs/agreement-study-2026-07-30.md)** · raw label pairs: [`docs/agreement/pairs.csv`](docs/agreement/pairs.csv)

### One decision that cost a cycle to learn

I A/B'd a cheaper model against GPT-4.1 and **kept the expensive one.** The cheap model lost the keyword-matching reliability the safety rubric depends on. The eval is what made that visible; without it the cost saving would have looked free.

---

## What's here and what isn't

**Here:** architecture, design decisions, evaluation methodology, the results, and the two extracted libraries.

**Not here:** the system prompts, the 50 scenarios, the rubric wording, and the product source. Those encode the domain judgment that took five months to build, and the method is separable from them.

If you want to evaluate the engineering, the two open-source repos are real code with real tests and I'd start there.

---

## Related

- **[crisp-eval](https://github.com/Arash-yazdani/crisp-eval)** — LLM-as-judge harness: CRISP taxonomy, judge-family checks, deterministic safety keys, judge-vs-human agreement statistics, failure mining. 42 tests, no dependencies.
- **[webhook-outbox-kit](https://github.com/Arash-yazdani/webhook-outbox-kit)** — fail-closed webhook verification across three vendor schemes, transactional outbox with dedupe and dead-letter quarantine. 41 tests, no dependencies.

Questions welcome: [LinkedIn](https://www.linkedin.com/in/arashyazdanimba) · [modeo.io](https://modeo.io)
