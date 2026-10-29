# FinAdvisor: A Four-Agent System for Compliant Health-Insurance Lead Qualification

A multi-agent pipeline that helps licensed financial advisors work a pool of health-insurance leads end to end: interpreting lead signals, tiering and prioritizing leads, drafting a compliant first outreach email, and handling the lead's reply. Hard regulatory guardrails (no pricing, no medical claims, no Medicare misadvising, no high-pressure tactics) are enforced at every step where a message could go out.


## Pipeline

```
Lead Pool (Excel)
   -> Agent 1: Lead Intelligence
   -> Agent 2: Qualification & Scoring
   -> Agent 3: Personalization & Outreach
   -> (lead replies)
   -> Agent 4: Conversation & Booking
   -> BOOK_MEETING / NURTURE / DISQUALIFY
```

| Agent | Job |
|---|---|
| **1. Lead Intelligence** | Reads one lead's raw record and reasons about which signals matter for that person, flags compliance concerns (e.g., Medicare-primary), and recommends next actions. Uses two read-only tools: `fetch_lead_record` and `fetch_pool_context`. |
| **2. Qualification & Scoring** | Assigns a business tier (A/B/C) through holistic reasoning across Need, Fit and Urgency. A Python fast-cut assigns Tier C immediately when compliance is `BLOCK`, so no model call is spent on a disqualified lead. |
| **3. Personalization & Outreach** | Drafts one outreach email personalized to the lead's strongest signal and tone-calibrated by life event, then runs it through a compliance scanner and rewrites (up to 3 times) until it passes. |
| **4. Conversation & Booking** | Handles the lead's reply across multiple turns with persistent state, and decides `BOOK_MEETING`, `NURTURE`, `DISQUALIFY` or `CONTINUE`. Removal requests ("stop emailing me") skip straight to `DISQUALIFY`. |

A `FinAdvisorPipeline` orchestrator wires the agents together, skips Agent 3 when Agent 2 returns Tier C/BLOCK, and aggregates cost and per-iteration telemetry for the whole run.

## Design principle: Python decides rules, the LLM decides judgment

- **Python:** fetching records, computing pool statistics, checking compliance flags, scanning text for banned phrases, persisting conversation state.
- **LLM:** interpreting what a signal means for a specific person, weighing competing priorities into a tier, choosing what to personalize around, and detecting intent in an ambiguous reply.

## Compliance: three layers

1. **System-prompt guardrails** repeated in every agent.
2. **A shared Python regex scanner** (banned-pattern categories for medical claims, specific pricing, financial guarantees, Medicare misadvising and high-pressure tactics, plus a required-disclaimer check), run independently of the model that wrote the text.
3. **A rewrite loop** that revises and re-scans up to 3 times, then reports the failure rather than sending a borderline message.

## Evaluation: LLM-as-judge

An independent Judge agent (never the same call that produced the output) grades Agent 4 on a 6-dimension rubric scored 1-5: Instruction Adherence, Reasoning Transparency, Hallucination Check, Compliance Integrity, Tone Continuity and Decision Appropriateness.

- **Test suite:** 50 synthetic reply cases (5 seeds x 10 variations) covering happy paths, hesitant multi-turn replies, removal requests, adversarial compliance demands and off-topic messages.
- **Consistency Score:** 10 curated cases x 3 fresh runs each = 30 independently judged outputs.
- **Results:** mean Judge score 4.52 / 5.0; 28 PASS, 2 MARGINAL, 0 FAIL. Removal-request handling was fully stable across runs.
- **Findings:** 3 of 10 cases changed their decision across runs. Message quality stayed strong, but the point at which the agent commits to booking a meeting varied. The regex also missed the bare word "Medicare" in one reply, a gap between the written policy and the enforcement pattern.

## FinOps guardrails

Every agent has an iteration cap (`MAX_ITERATIONS = 8`) and an output-token cap per call (`MAX_OUTPUT_TOKENS = 1800`). Per-iteration telemetry (tokens, latency, tool calls) feeds a cost report using real per-token pricing.

## Team

Hanna Teshome, Prathiba Reddy, Melody Borg

## Setup

1. Install dependencies: `pip install -r requirements.txt`
2. Set your key as an environment variable: `ANTHROPIC_API_KEY`. Never commit it.
3. Run the notebook.

Lead data in this repository is synthetic.
