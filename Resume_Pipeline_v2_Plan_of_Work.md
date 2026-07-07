# Resume Pipeline v2 — Plan of Work

**Goal:** Collapse the five-prompt, two-model, copy-paste pipeline into a single Cowork/Claude Code skill invoked with a job description, producing one concise report and — when warranted — a submission-ready .docx, with the Work History Document (WHD) improving as a side effect of each run.

**Design constraint:** Preserve the three properties that make v1 work — the honesty guardrails (Stretch/Hard No classification), the adversarial blindness of the screening stress test (it must see only what a recruiter sees, never the WHD), and the conservative default when employer evidence is thin. Everything else is negotiable.

---

## 1. Stage consolidation review

What each v1 stage contributes, and what happens to it in v2:

| v1 Stage | Core value | v2 disposition |
|---|---|---|
| Pre-Stage (WHD) | Standing source of truth + voice sample | Kept as standing asset; restructured (§5) with a summary index and changelog. No longer manually re-run — patched in-workflow (§6). |
| Stage 0 (SCD) | Employer intelligence, shadow requirements, conservative default | Kept; becomes a **research subagent** running in parallel. Web search replaces "paste an official source." |
| Stage 1 (Micro Fit) | Gap Map, Recoverable Gaps, scored math | Kept; becomes a **fit subagent** running in parallel with research. Gap Map is the pipeline's most differentiated output — untouched. |
| Stage 2 (Macro) | Six-second/three-minute stress test, elimination logic | Kept but compressed; becomes a **screening subagent**, sequential after fit. Its WHD-blindness is preserved by construction: the subagent's context simply never includes the WHD. |
| Stage 3 (Triage) | Worth-It verdict, prescriptions, honesty check | Absorbed into the **orchestrator's synthesis step**. No longer a separate handoff — it is the report. |
| Stage 4 (Draft) | Voice-calibrated draft | Replaced by an **interactive finishing loop** (§4). Tag-laden markdown is eliminated as the user-facing artifact. |

**Redundancies eliminated:**

- **JD parsed once.** v1 has Stages 1 and 2 each re-derive requirements, archetypes, and personas from the raw JD. v2 parses the JD into a canonical requirements schema (hard/preferred, keywords, seniority signals, recruiter-persona cues) at intake, and every downstream step consumes the schema, not the raw JD. This is the single largest token saving.
- **Archetype determined once.** Seeker Archetype, JD Archetype, and Verified Archetype Target are currently computed in three places and reconciled in a fourth (Stage 3 Delta Assessment). v2 computes each once and reconciles once, in synthesis.
- **Handoff blocks eliminated.** "Immutable Handoff Variables" exist because v1 crosses chat boundaries. v2 writes compact structured artifacts (YAML) to a per-run folder; the orchestrator passes each subagent only the artifacts it needs. No re-narration of prior outputs.
- **Ritual eliminated.** "Ensure Using Sonnet/Opus" announcements, per-stage input guards, and "paste Stage N next" instructions all disappear. Model routing is an orchestrator parameter (cheap model for research/fit extraction, strong model for screening simulation and synthesis); input validation happens once at intake.

## 2. Orchestrated architecture

One skill: `/resume-fit <JD file, pasted text, or URL>` with optional flags (`--draft` to proceed to finishing, `--quick` for score-only).

```
Phase A  INTAKE (orchestrator, one pass)
         Validate: JD present, current resume located, WHD located.
         Parse JD → requirements.yaml (canonical schema).
         Create run folder: runs/<company>-<role>-<date>/

Phase B  PARALLEL (two subagents, cheap model)
         ├─ Research subagent: web search on employer → scd.yaml
         │    (business cycle, buyer anxiety, shadow requirements,
         │     evidence confidence; conservative default preserved)
         └─ Fit subagent: requirements.yaml + resume + WHD → gapmap.yaml
              (Gap Map, Recoverable Gaps, score math, ATS keywords)

Phase C  GATE 1 — early exit (Gap Brief, not a score cutoff)
         A zero-token Python step tallies gapmap.yaml: any hard
         requirement that is None on the resume AND absent from the
         WHD is an *unrecoverable gap*. If a categorical trip-rule
         fires (e.g., ≥2 unrecoverable hard gaps, or more than a
         third of hard requirements unrecoverable), the orchestrator
         writes a Gap Brief: each unrecoverable gap, what filling it
         would actually require (experience you don't have vs. a
         credential vs. pure repositioning), and a one-line magnitude
         verdict. User confirms stop or overrides. The numeric score
         appears as a diagnostic line, never as the decision rule —
         the reasoning is the gate.
         Interrogation format (fixed): the Gap Brief is presented as
         one structured question with exactly three options —
         (a) Stop, archive the Gap Brief to the run folder;
         (b) Proceed anyway, gaps acknowledged (recorded in the
             report's Open Questions so the decision is visible);
         (c) Contest a gap — "I have evidence for X" — which routes
             immediately into the Phase H micro-interview: evidence
             is captured, the WHD is patched, and the gap tally is
             recomputed before proceeding.
         Option (c) is what makes propose/dispose trustworthy: an
         override is never a shrug, it is either an accepted risk or
         new evidence on the record.

Phase D  SCREENING (one subagent, strong model)
         Inputs: resume + requirements.yaml + gapmap summary + scd.yaml.
         Explicitly NOT the WHD. → screen.yaml
         (Gate 1/Gate 2 triggers, elimination logic, HM acceptance risk)

Phase E  SYNTHESIS (orchestrator, strong model)
         All artifacts + WHD → Worth-It verdict, prescriptions,
         honesty check, information gaps → report.md (§3)

Phase F  GATE 2 — user decision
         Report presented. AskUserQuestion: proceed to draft / stop /
         resolve information gaps first.

Phase G  FINISHING LOOP (interactive, §4) → resume_final.docx
Phase H  WHD RECONCILIATION (interactive, §6) → WHD patched
```

Token-efficiency principles: subagents receive minimal context (the fit subagent never sees the SCD; the research subagent never sees the resume); artifacts are structured YAML, not prose; the orchestrator reads artifact summaries, pulling full detail only where synthesis requires it; early-exit gates prevent spend on doomed applications.

Wall-clock effect: v1 is five sequential chat sessions with manual model switching. v2 is one invocation; Phase B runs research and fit concurrently, and for a "Do Not Pursue" JD the whole run ends in roughly the time v1 spent on Stage 0 alone.

**Exception-driven interrogation (cross-cutting rule).** Grill-me-style questioning is not confined to the finishing loop. The orchestrator carries a standing escalation rule: when any phase surfaces a fundamental mismatch or contradiction, pause and interrogate the user rather than carry the ambiguity forward into more expensive phases. Defined triggers:

- *Phase B (fit):* Seeker Archetype and JD Archetype are structurally incompatible (not merely distant) — e.g., IC resume against a people-leadership mandate. Ask whether repositioning is intended before screening simulates the wrong candidate.
- *Phase B (research):* SCD evidence contradicts the resume's positioning (e.g., company in efficiency mode, resume leads with growth-spend narratives), or research surfaces something that changes whether the user wants the job at all (layoffs, acquisition, leadership exodus). Surface it now, not in the report.
- *Phase E (synthesis):* The honesty check finds a load-bearing stretch — a claim that, if withdrawn, flips the Worth-It verdict. Confront it before the report is written, not after a draft exists.

Each trigger is one question round, subject to the same per-gate cap as the finishing loop (§9). The principle: interview on exception, not batch-then-interview — a mismatch discovered at Phase B costs one question; the same mismatch discovered in a finished draft costs a rewrite.

## 3. The unified report

One `report.md` per run replaces four stage outputs. Verdict-first, one to two pages, ~600–900 words. Structure:

1. **Verdict line** — Apply / Apply with edits / Do not pursue, with Worth-It tier and scope of work in the same sentence.
2. **Numbers strip** — paper score, recoverable gaps count, escalation likelihood, evidence confidence. Four numbers, one line each.
3. **The two triggers** — first friction trigger and first escalation trigger, verbatim from screening. These are the report's most actionable diagnostic and lead the analysis section.
4. **Prescriptions table** — Edit | Why | Source (WHD section). Reframe/Add/Remove/Reorder collapsed into one table. Every Recoverable Gap gets a row (v1's mandatory rule preserved).
5. **Honesty lines** — "You can honestly present as ___. You cannot claim ___." Two sentences, no elaboration.
6. **Shadow requirements** — from the SCD, only those not already covered by a prescription row.
7. **Open questions** — information gaps blocking editing, if any.

Everything else — score math, full Gap Map, persona reasoning, competitive comparison — is written to `appendix.md` in the run folder for inspection, never rendered in the report. The v1 principle "the score derives from the map" survives: the map exists and is auditable; it just isn't the headline.

## 4. Grill-me finishing loop (Stage 4 replacement)

v1's failure mode: Stage 4 emits a markdown draft strewn with [CHANGED]/[CANDIDATE TO SUPPLY]/[NOT INTEGRATED] tags that the user must manually resolve, strip, and reformat. v2 inverts this — the system interviews the user until the document is clean, then renders it.

1. **Silent draft.** Voice calibration and drafting rules carry over unchanged from v1 Stage 4 (voice profile from the WHD Voice Sample, no unflagged rewrites, honesty constraints, no fabrication). The tagged draft is generated but never shown as the deliverable — it becomes the finishing loop's worklist.
2. **Interrogation rounds.** Issues are batched by type and pushed through AskUserQuestion, highest-stakes first:
   - *Supply rounds* — each [CANDIDATE TO SUPPLY]: what's needed, why, one question. User answers in their own words; the answer is inserted near-verbatim (light tense/format conformance only), which sidesteps most voice-drift risk.
   - *Keyword rounds* — each [NOT INTEGRATED]: offer 2–3 candidate-voice integration options plus "skip — leave it out."
   - *Voice rounds* — each VOICE NOTE / low-confidence [CHANGED] line: show original vs. rewrite, user picks or dictates a third phrasing.
   - *Stretch confrontations* — any prescription touching a Stage 3 stretch claim: the user explicitly accepts the honest framing or drops the line. No silent softening.
3. **Exit criteria.** The loop does not terminate until zero tags remain and the voice audit passes. If the user stalls, the draft saves with an explicit NOT SUBMITTABLE banner and a list of what remains.
4. **Render.** Clean markdown → .docx via the docx skill against a fixed resume template (single column, ATS-safe fonts, no tables/text boxes/headers that parsers mangle). Deliverables: `resume_final.docx` + `resume_final.md` (source of truth) in the run folder.

## 5. WHD restructure (enabler)

Before building the skill, restructure the existing WHD once:

- **Front-matter index** — one line per role (company, title, dates, 3–5 capability tags) so subagents can retrieve selectively instead of ingesting the full document.
- **Stable section anchors** — each role/project gets an ID (`role-3.project-2`) so prescriptions and patches cite machine-resolvable locations.
- **Changelog section** — date-stamped entries recording what changed and which application run prompted it.
- **Voice Sample untouched** — remains the calibration source; the reconciliation loop never edits it.

## 6. WHD reconciliation loop (fifth consideration)

v1 flags an honesty discrepancy or a thin spot and leaves you to re-run the Pre-Stage interview by hand. v2 closes the loop inside the run, in Phase H:

1. **Capture.** During synthesis and the finishing loop, the orchestrator accumulates a patch queue: new facts the user supplied (metrics, projects, scope details), corrections ("that was 8 reports, not 12"), evidence surfaced for items previously marked Partial, and information-gap answers.
2. **Classify.** Each item is tagged *WHD-worthy* (durable fact about your history) or *application-specific* (framing for this employer only). Only the former is proposed as a patch.
3. **Micro-interview on discrepancies.** For each Stretch or Hard No the triage flagged: one targeted question — "Do you have real evidence for X?" If yes, the evidence is captured into the relevant WHD section, and X may upgrade from Stretch to Genuine in future runs. If no, a `hard-no: X (confirmed <date>)` marker is written so future runs don't re-litigate it. This is the honesty check compounding instead of repeating.
4. **Propose, don't apply.** Patches are presented as a diff against the anchored WHD sections (AskUserQuestion per patch or batch-approve). Approved patches are applied with changelog entries; rejected ones are dropped. The WHD's validation-pass ethos ("is anything overstated?") is preserved because the user ratifies every edit.
5. **Full Pre-Stage retained** for genuinely new roles or major life changes — reconciliation handles increments, not rebuilds.

Net effect: every application run makes the standing asset more complete and more honest, and the Recoverable Gaps count should trend downward across runs — a measurable signal the system is working.

## 7. Local compute and API strategy

The skill remains the orchestrator — it provides the interactive question UX, subagent dispatch, and automatic prompt caching for free. But a meaningful slice of the pipeline is deterministic and should never spend a token. Ship these as Python scripts inside the skill, invoked via bash:

- **Score arithmetic** — the model classifies each requirement (Match/Partial/None); Python computes the weighted score from gapmap.yaml. Eliminates LLM arithmetic errors and the tokens spent "showing the math."
- **ATS keyword scan** — exact-match term frequency between JD and resume is string matching, not judgment. Python produces the missing-keyword list; the model only adjudicates terminology mismatches (semantic, e.g., "Product Ops" vs. "Program Management").
- **Gate 1 tally** — unrecoverable-gap counting and trip-rule evaluation (§2 Phase C).
- **Plumbing** — run-folder scaffolding, YAML schema validation, WHD anchor resolution, patch application and changelog writes, finishing-loop exit check (tag count must reach zero), resume version diffs, docx render.

**Optional semantic prefilter (Phase 3.5).** For a large WHD, an embeddings pass (local sentence-transformers, or an embeddings API call — both near-free relative to generation) can shortlist the 3–5 most relevant WHD sections per requirement. The fit subagent then adjudicates candidates instead of reading the whole document. Defer until the WHD is big enough to hurt; the §5 front-matter index may be sufficient.

**API-level efficiency, if a direct-API path is ever built:**

- *Prompt caching* — structure every call so the WHD + resume + JD sit in a shared cached prefix; all subagent calls after the first read the prefix at the cached-input discount. This is the largest single lever, and it's what a Python orchestrator would buy you that the skill already approximates.
- *Model routing* — already in the design: cheap model (Haiku-class, or Gemini Flash) for extraction and research, strong model only for screening simulation and synthesis.
- *Batch API* — irrelevant for a single interactive run, but enables a **bulk triage mode**: queue 10–20 saved JDs overnight at the batch discount, wake up to a ranked list of Gap Briefs, and run the full interactive pipeline only on survivors. This is the strongest argument for a direct-API companion script.
- *Gemini specifically* — search-grounded generation is a genuine fit for Stage 0 employer research, and Flash pricing on long contexts is attractive. Cost: a second vendor means two SDKs, two key managements, and less consistent structured output. Recommendation: stay single-vendor unless Phase 7 validation shows research quality lagging. Verify current pricing for both vendors at build time; rates change faster than this plan.

Decision rule: skill + embedded Python helpers is the default architecture. A standalone Python/Agent-SDK orchestrator is justified only if bulk triage becomes the primary usage pattern.

## 8. Build sequence

| Phase | Work | Depends on | Est. effort |
|---|---|---|---|
| 1 | WHD restructure (§5): index, anchors, changelog | — | 1 session |
| 2 | Skill scaffold via skill-creator: intake, JD→requirements.yaml parser, run-folder convention, artifact schemas (requirements/scd/gapmap/screen YAML), Python helpers (score math, ATS scan, plumbing per §7) | 1 | 1–2 sessions |
| 3 | Subagent prompts: port Stage 0/1/2 FINAL prompts into the subagent contract template (§9 — verbatim invariants block, input manifest, output schema, refusal conditions); wire parallel dispatch, Gate 1 Gap Brief (tally script + trip rules), exception-driven interrogation triggers, and screening-blindness enforcement (tool restriction + canary check) | 2 | 1–2 sessions |
| 4 | Synthesis + report: fold Stage 3 logic into orchestrator; report.md and appendix.md templates | 3 | 1 session |
| 5 | Finishing loop (§4) + docx template and render | 4 | 1–2 sessions |
| 6 | WHD reconciliation (§6): patch queue, micro-interview, diff-approve | 5 | 1 session |
| 7 | Validation: replay 2–3 JDs previously run through v1; compare verdicts, prescriptions, and honesty calls against v1 outputs; measure token spend and wall-clock; tune Gate 1 trip rules | 3–6 | 1 session |
| 8 (opt) | Bulk triage mode: direct-API batch script producing ranked Gap Briefs across saved JDs (§7) | 7 | 1–2 sessions |

Phases 1–4 alone deliver the core win (one command → one report, no copy-paste). Phases 5–6 deliver the submittable document and the compounding WHD. Ship 1–4 first and run it on a live application before building 5–6.

## 9. Risks and open questions

- **Gate 1 trip rules** still need calibration, but as categorical rules (how many unrecoverable hard gaps justify stopping) rather than a score threshold. Start permissive — the gate proposes, the user disposes, via the fixed three-option interrogation format in §2 Phase C (stop / proceed with gaps on the record / contest with evidence). Tighten the trip rules once a few Gap Briefs have been seen in practice. The false-precision trap from the v1 user guide applies doubly to an autonomous numeric cutoff; the Gap Brief design exists to avoid it.
- **Subagent drift — mitigated by a contract template.** Every subagent definition follows one fixed template with five sections: (1) Role, one line; (2) **Invariants block** — the calibrated behavioral rules lifted *verbatim* from the v1 FINAL prompts and marked do-not-paraphrase (the conservative SCD default, the "score the resume alone, note WHD evidence separately" rule, the "structured approximations, not predictions" framing, the no-fabrication rules); (3) Input manifest — the exact artifacts this agent receives, and nothing else; (4) Output schema — required YAML keys, validated by Python on return, so structural drift fails loudly at dispatch rather than silently downstream; (5) Refusal conditions. Compression is applied only to v1's narrative, ritual, and handoff scaffolding — never to the invariants. Phase 7's replay comparison then catches behavioral drift the template didn't prevent; the replay outputs become a small golden-fixture set for regression whenever a subagent prompt is later edited.
- **Screening-blindness is a code problem, with prompt as backstop.** Enforcement layers, in order of authority: (1) *dispatch construction* — the orchestrator builds the screening subagent's input from an explicit manifest that omits the WHD; (2) *tool restriction* — the screening subagent gets no file-read tools; all inputs arrive inline, so it cannot go find the WHD even if it "wanted" to; (3) *canary check* — a unique token embedded in the WHD front-matter; deterministic post-run scan of screen.yaml fails the run if the canary appears; (4) prompt instruction ("you see only what the recruiter sees") — retained for framing quality, but never relied on for enforcement. An LLM instruction is a request; a missing capability is a guarantee.
- **Voice risk doesn't vanish.** The finishing loop reduces AI-voice drift by sourcing answers from the user live, but the v1 warning stands: read the final draft aloud before submitting. The report should carry a one-line reminder.
- **Interactivity budget.** With exception-driven interrogation (§2) now running throughout the pipeline, the cap matters more: ~3 question rounds per gate (Phase B/C/E triggers and each finishing-loop type). Overflow items go to a punch list rather than blocking progress. The distinction to preserve: early-phase triggers fire only on *fundamental* mismatches (archetype incompatibility, load-bearing stretch claims, disqualifying research findings) — routine gaps and phrasing issues wait for their designated phase, or the pipeline degenerates into a continuous interview.
