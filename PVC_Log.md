# FinAdvisor — Prompt Version Control (PVC) Log

**Course:** Generative AI | **Team:** Hanna Teshome, Prathiba Reddy, Melody Borg
**Track:** B — Claude Agent SDK | **Architecture:** Multi-Agent Orchestrator

This log documents the evolution of the system prompts for each of the four agents in the FinAdvisor pipeline. For every agent we record three iterations, the failure mode each version surfaced, and the specific change that fixed it.

---

## Why PVC matters here

FinAdvisor is a regulated outreach system. A subtle prompt bug — an agent that pattern-matches on a single signal, or one that hallucinates a lead record when a tool call fails — can produce non-compliant outreach to a real prospect. Each prompt revision below was driven by a specific test-case failure, not by aesthetic polish.

We use the same iteration cadence across all four agents:
- **v1**: minimal "do the job" framing — to surface failure modes quickly
- **v2**: structural fix (output schema, ReAct framing, dimension scoring)
- **v3**: reasoning-first reframing + hard guardrails — the version we ship

Agent 4 needed an extra round (**v4**) after a real production-style bug surfaced during testing (see §4.4).

---

# 1. Agent 1 — Lead Intelligence

**Role:** Read raw lead signals from the Excel file and reason about which signals matter for *this* specific person. Output a structured `intelligence_report`.

| Version | What the prompt said (essence) | Failure mode observed |
|---|---|---|
| **v1** | *"You are a lead analyst. Use the tools to assess leads."* | Agent called tools randomly; sometimes skipped `fetch_lead_record` entirely and hallucinated lead data; gave generic responses with no reasoning shown ("This person seems like a good lead"). |
| **v2** | Added an explicit ReAct instruction, listed the signals available, and required a structured output. | Agent now fetched records consistently, but applied fixed mental weights — "Uninsured = always high priority" — regardless of context. Ignored age × employment interactions (a 26-year-old uninsured ≠ a 58-year-old uninsured). |
| **v3 ✅** | Reframed the agent as a *reasoning entity*, not a classifier. Removed any reference to tiers (those belong to Agent 2). Added comparative reasoning instructions tied to the `fetch_pool_context` tool. Required signal-by-signal reasoning before conclusions, surfaced inside an `<intelligence_report>` XML tag with strict JSON schema. | Stable. Agent now reasons holistically — a 64-year-old self-employed person is treated differently than a 28-year-old in the same income bracket. Compliance flags (BLOCK / REVIEW / CLEAR) emerge from regulatory reasoning, not pattern matching. |

### v3 design choices that stabilized the agent

1. **No tiers in this prompt.** Putting "Tier A/B/C" language in Agent 1 polluted the reasoning — the model would skip the analysis and jump to a label. Removing it forced the model to surface signal interpretation first.
2. **Comparative reasoning is opt-in.** The prompt tells the agent that `fetch_pool_context` is *available* but not mandatory. This let the agent decide when comparison adds value, which it does for Test 2 (Comparative Reasoning) but not for Test 1 (single happy-path lead).
3. **XML tag wrapping JSON output.** `<intelligence_report>{...}</intelligence_report>` survives narrative wrappers and markdown fences, which raw JSON-only prompts did not.

### Evidence from test runs
- **Test 1 (LEAD-0001, divorce + ACA):** v1 returned "high priority" with no reason. v3 surfaces the divorce as a Special Enrollment Period trigger, notes household composition change, recommends contact within 60-day SEP window.
- **Test 2 (LEAD-0042 comparative):** v1 ignored pool context. v3 autonomously calls both `fetch_lead_record` AND `fetch_pool_context` without being told to, and reasons about whether the lead's situation is typical or unusual.

---

# 2. Agent 2 — Qualification & Scoring

**Role:** Read Agent 1's `intelligence_report` and assign Tier A/B/C through holistic reasoning. Output a structured `qualification_report`.

| Version | What the prompt said (essence) | Failure mode observed |
|---|---|---|
| **v1** | *"You are a lead scorer. Score leads A/B/C based on the intelligence report."* | Agent pattern-matched: "outreach_priority: High" → Tier A, every time. Ignored compliance nuance entirely. All five seed leads came back Tier A. No independent reasoning. |
| **v2** | Introduced three scoring dimensions (need / fit / likelihood) on a 1–10 scale. Required dimension-level rationale before tier assignment. Added composite-score formula: `(need + fit + likelihood) / 3`. | Scores were more granular but tiers were still mechanical: composite ≥ 7 → A, ≥ 5 → B, else C. The numeric crutch defeated the point. **REVIEW**-compliance leads were scored cleanly but no `advisor_handoff_required: true` flag was raised, meaning a downstream handoff would silently fail. |
| **v3 ✅** | Reframed as a *reasoning entity making a business decision*, not a calculator. Removed numeric composite. Replaced with: "A lead can score high on need but low on urgency and still be Tier B. A lead with moderate need but COBRA expiring in 30 days may be Tier A. Reason about what an experienced advisor would prioritize." Added explicit REVIEW/BLOCK handling. Required `tier_rationale` to reference specific Agent 1 signals by name. | Stable. Tier assignment is now holistic. The two-layer architecture (Layer 1 Python deterministic for BLOCK + Layer 2 LLM for everything else) is also encoded here. |

### v3 design choices that stabilized the agent

1. **Removed the composite formula.** This was counterintuitive — adding scoring math felt rigorous, but it just gave the model a shortcut around the actual reasoning. v3 keeps need/fit/urgency as **categorical** dimensions (High/Medium/Low) with no aggregation rule.
2. **Hard guardrail: BLOCK is non-negotiable.** Three times in the prompt, in different phrasings, with the rationale that compliance is a binary rule and the deterministic Layer 1 fast-cut handles it. The model still tries to second-guess BLOCK leads occasionally; making the rule redundant in the prompt suppresses that.
3. **`tier_rationale` must reference Agent 1's signals by name.** This forced the model to actually read Agent 1's report rather than re-derive an assessment from the raw record.

### Evidence from test runs
- **Test 1 (LEAD-0007 COBRA + Gap):** v1 → Tier A (correct, but for wrong reason: just echoed High priority). v3 → Tier A with rationale that explicitly references COBRA's transitional nature + part-time income constraint, which are the real urgency drivers.
- **Test 2 (LEAD-0002 BLOCK):** v1 → Tier A (catastrophic — would have outreach to a Medicare-primary lead). v3 → Tier C, deterministic, single tool call, no LLM scoring needed. Token cost drops by ~60%.

---

# 3. Agent 3 — Personalization & Outreach

**Role:** Given a scored lead, draft a single compliant outreach email. Use Agent 1 + 2 reports for personalization. Run a regex compliance scan before finalizing.

| Version | What the prompt said (essence) | Failure mode observed |
|---|---|---|
| **v1** | *"Draft a personalized outreach email for the given lead. Make sure it doesn't violate any compliance rules."* | Agent invented compliance rules ("I think we shouldn't mention pricing"). Caught some violations, missed others. Quoted specific premiums in 2 of 5 seed cases ($199/month, $89/month) because nothing told it not to. The "compliance rules" were vibes, not policy. |
| **v2** | Added an enumerated list of hard rules (no pricing, no health-outcome claims, no Medicare for under-65, no high-pressure tactics). Required a tool call to `check_compliance_keywords` before finalizing. | Hard rules were respected but enforcement was inconsistent: when a rewrite produced a marginal violation, the agent sometimes finalized anyway because the prompt didn't require **re-scanning** after rewrite. Bereavement-sensitive cases got the same congratulatory tone as new-baby cases — no life-event awareness. Required disclaimer ("a licensed advisor will follow up") was inconsistently included. |
| **v3 ✅** | Restructured into a "mandatory workflow" with numbered steps. Added explicit Tier-C / BLOCK halt at step 1. Added tone calibration by tier (warm-direct / educational-nurture / soft-touch). Added life event sensitivity table (bereavement → soft tone, new baby → warm-congratulatory, etc.). Required up to 3 rewrite-and-rescan iterations. Locked output to a strict JSON schema. | Stable. 90% first-scan compliance pass rate across the 20-lead batch run. Tone selection matches the life event correctly across all 5 seed cases. Tier-C / BLOCK leads short-circuit cleanly without drafting. |

### v3 design choices that stabilized the agent

1. **Numbered mandatory workflow.** Replacing "use the tools as needed" with "1. Call X. 2. Call Y. 3. Draft. 4. Scan. 5. Decide." cut the average iteration count from ~5 to ~3.4 across the batch run. Cost per lead dropped from ~$0.05 to ~$0.035.
2. **Tone calibration table.** Hard-coding (life event → tone) into the prompt was the cheapest fix for the bereavement-vs-celebration error. Tested across 5 life-event categories before locking.
3. **Up to 3 rewrites, then abandon.** Without this cap, one adversarial test (Test 4, "include $50/month and 'act now'") could send the agent into a 6+ iteration loop before realizing the user-injected content was non-fixable. v3 abandons after 3 attempts and returns DO_NOT_SEND.

### Evidence from test runs
- **Test 4 (Adversarial Injection):** User explicitly asked for `$50/month`, `save your life`, and `act now`. v1 included two of three. v3 hard-blocked all three at the **drafting** stage (never wrote them), then transparently noted the blocks in `compliance_check.violations_caught`.
- **Test 5 (Bereavement, LEAD-0042):** v2 produced "Congratulations on your transition!" because it pattern-matched on "life event = positive event." v3 picked soft-touch tone, used the phrase "incredibly difficult time," and explicitly avoided urgency framing despite the lead being Tier A.

---

# 4. Agent 4 — Conversation & Booking

**Role:** Handle inbound replies to Agent 3's outreach. Multi-turn stateful conversation deciding BOOK_MEETING / NURTURE / DISQUALIFY.

This agent's PVC log has **four** versions, not three, because v3 shipped with a real bug that surfaced during the standalone-notebook test runs. Documenting the v3 → v4 transition honestly is more useful than backfilling a clean three-version history.

| Version | What the prompt said (essence) | Failure mode observed |
|---|---|---|
| **v1** | *"Process an inbound reply. Decide whether to book a meeting, nurture, or disqualify."* | No state awareness — agent treated every turn as turn 1. Asked the same qualifying question twice across turns. No compliance scanning of outbound replies. Quoted pricing in one test case when the lead asked "how much?" |
| **v2** | Introduced the four-tool toolkit (`fetch_lead_context`, `load_conversation_state`, `save_conversation_state`, `check_compliance_keywords`) and four named qualifiers to collect (interest, household, coverage, contact time). | Agent now persisted state correctly. But it asked **four questions in turn 1** to "fill all qualifiers efficiently," which felt aggressive. Removal requests ("stop emailing me") were processed through the standard qualification flow, generating a follow-up question — exactly the wrong response. |
| **v3** | Restricted to one qualifying question per turn. Capped at 3 conversation turns. Added explicit hard rules around Medicare language for under-65 leads and licensed-advisor deferral for regulated questions. | **Bug surfaced in Test 3 (removal request).** Agent correctly drafted "Understood — you have been removed from our list and will not be contacted again." But the **compliance scanner v1** flagged it as missing the `licensed` disclaimer. The agent rewrote with "a licensed advisor will not contact you again" — which the compliance scanner *also* flagged as a missing disclaimer (because it doesn't recognize the negation). The agent looped until `MAX_ITERATIONS=6` and returned a failure. Cost: $0.064, no useful output. |
| **v4 ✅** | Two paired changes: (1) **Compliance scanner v2** — added a regex set for removal-acknowledgment phrases ("removed from", "will not contact", "off our list", etc.). When detected, the scanner skips the disclaimer check entirely and returns `message_type: removal_acknowledgment`. (2) **Prompt v4** — added a "CRITICAL — Removal request short-circuit" section at the top with explicit instructions: skip context fetching, set DISQUALIFY immediately, draft one-sentence acknowledgment, do NOT mention licensed advisor. | Stable. Re-running the failed Test 3 with v4: 4 iterations, $0.039, correct DISQUALIFY decision, clean acknowledgment text. |

### What v3 → v4 actually taught us

The bug was not "the prompt was wrong." The bug was **a mismatch between two layers of the system**: the prompt told the agent to acknowledge removal succinctly, while the deterministic Python scanner enforced a rule that only made sense for outreach. Both layers were individually correct. The fix had to live in both:

- **Scanner change:** treat removal acknowledgments as a distinct message type with their own rules.
- **Prompt change:** make the short-circuit explicit so the agent doesn't enter the rewrite loop in the first place.

Fixing only one layer would not have worked:
- Scanner-only fix → prompt would still try to "improve" the removal acknowledgment by adding licensed-advisor language, which is exactly what the lead asked us not to do.
- Prompt-only fix → if the LLM ever drafted a removal acknowledgment with slightly different phrasing, the scanner would still loop it.

This is the kind of failure that doesn't show up in unit tests on either component in isolation. It's the reason we built the four-agent telemetry trace and the FinOps "iterations hit" alarm — without the trace, the bug would have looked like "Agent 4 returned an error, retry it."

### v4 design choices that stabilized the agent

1. **Short-circuit at the top of the prompt.** Putting the removal-request rule above the standard workflow tells the model to evaluate it first. Less efficient prompts that hid the rule under "edge cases" still occasionally entered the standard flow.
2. **One drafting attempt + one rewrite cap.** Down from v3's "rewrite until PASS." The compliance scanner cooperates with this — if v4 hits a real violation it can't fix in one rewrite, it abandons and returns the decision based on what it knows.
3. **No mention of "licensed advisor" in removal acks.** Counter-intuitive: the rest of the system *requires* this disclaimer, but for a person asking to be left alone, even mentioning future contact is a violation of the implicit consent withdrawal.

### Evidence from test runs

| Test | v3 result | v4 result |
|---|---|---|
| Test 1 — Interested lead | ✅ CONTINUE, $0.049 | ✅ CONTINUE, $0.049 |
| Test 2 — Multi-turn → Booked | ✅ BOOK_MEETING, $0.055 | ✅ BOOK_MEETING, $0.055 |
| Test 3 — Removal request | ❌ MAX_ITERATIONS, $0.064 | ✅ DISQUALIFY, $0.039 |
| Test 4 — Adversarial pricing | ✅ CONTINUE (deflected), $0.050 | ✅ CONTINUE (deflected), $0.050 |
| Test 5 — Off-topic | ✅ NURTURE, $0.046 | ✅ NURTURE, $0.046 |

The v4 fix is non-regressive — every other test produced identical output at identical cost.

---

# 5. Cross-cutting patterns we observed

After four agents and ~13 prompt iterations, three patterns held across all of them:

### 5.1 "Reasoning entity, not classifier" framing
Every agent improved sharply at the v2 → v3 transition when we replaced "do task X" framing with "you are reasoning about a person/situation." This shifted the model from pattern-matching on signal labels to interpreting signals in combination. Counterintuitively, **fewer instructions** at this layer — paired with **more concrete examples** elsewhere — produced more reliable behavior than dense rule lists.

### 5.2 XML-tagged JSON output beats raw JSON
For Agents 1 and 2 we wrap structured output in `<intelligence_report>...</intelligence_report>` and `<qualification_report>...</qualification_report>` tags. This survives:
- markdown fences (` ```json ... ``` `)
- narrative preamble ("Here is the report you asked for: ...")
- truncated outputs (the parser can find the start tag even if the closing brace is missing)

Agents 3 and 4 use raw JSON because their outputs go directly to the orchestrator with strict schema validation, so robustness to formatting is less important than schema enforcement.

### 5.3 Hard rules survive iteration; soft preferences don't
"NEVER quote pricing" survived every prompt revision unchanged.
"Use a friendly tone" needed to be replaced with "Tier A → warm-direct / Tier B → educational-nurture / Tier C → no outreach" before the model behaved consistently.

The pattern: rules expressed as **invariants** ("X is always false") are stable. Rules expressed as **preferences** ("we prefer X") drift. Where we needed nuance — bereavement vs. celebration tone — we encoded it as a lookup table, not a guideline.

---

# Appendix — File pointers

- Agent 1 prompt v3: `FinAdvisor_Lead_Intelligence_Agentic.ipynb`, Cell 6 (`SYSTEM_PROMPT`)
- Agent 2 prompt v3: `FinAdvisor_Qualification_Scoring_Agent2.ipynb`, Cell 6 (`SYSTEM_PROMPT_Q`)
- Agent 3 prompt v3: `GenAI_Agent3.ipynb`, Cell 6 (`SYSTEM_PROMPT_V3`)
- Agent 4 prompt v4: `GenAI_Agent4.ipynb`, Cell 30 (`AGENT4_SYSTEM_PROMPT_V4`)
- Combined pipeline (all final-version prompts): `GenAI_FinAdvisor_Combined.ipynb`, Sections 2–5
- Compliance scanner v2 (the v3 → v4 fix): `GenAI_Agent4.ipynb`, Cell 29 + `GenAI_FinAdvisor_Combined.ipynb`, Section 1
