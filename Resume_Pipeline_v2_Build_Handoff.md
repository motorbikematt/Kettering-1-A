# Build Handoff — Resume Pipeline v2

Paste this as the opening message of a fresh **Opus 4.8** session inside the **"Background and Employment history"** Claude Project (so WHD v4 and the v1 FINAL prompts are in project knowledge), with **two folders selected**:

1. `D:\vibe\career-diagnostic-pipeline` — the public repo (code plane)
2. `D:\vibe\Personal Background and Employment History` — Matt's private data (data plane)

---

You are executing an approved build plan. Do not redesign it. Where the plan and your judgment conflict on architecture, the plan wins; raise the conflict, don't silently deviate.

## Project intent — governs every decision

This is a **public open-source project** (MIT, GitHub: `motorbikematt/career-diagnostic-pipeline`). The goal is that other job seekers can install and use this skill, not just the author. Consequences:

- The skill source is **user-agnostic**: no personal names, employers, file contents, or paths baked into prompts, code, or examples. The author is one user of his own tool.
- The skill locates its data plane via **configuration** (a config file or first-run prompt storing the data-plane path), never a hardcoded path.
- Personal data **never enters the repo** — not gitignored-but-present, physically absent. Two-plane separation below is the enforcement mechanism.
- Synthetic examples (fake candidate, fake JD) are the only fixtures that may live in the repo.

## Two-plane layout

```
CODE PLANE — career-diagnostic-pipeline (public repo)
  skill/        v2 skill: SKILL.md, subagent contracts, Python helpers,
                YAML schemas, report + docx templates
  prompts/      v1 FINAL prompts (existing; canonical invariants source)
  docs/         existing docs; update to v2 after Phase 4
  examples/     synthetic end-to-end fixture (fake candidate + fake JD)
  archive/      superseded artifacts (JSX orchestrator moves here)

DATA PLANE — Personal Background and Employment History (private folder)
  pipeline/whd/       canonical writable WHD (copied from project knowledge)
  pipeline/runs/      one folder per job application
  pipeline/fixtures/  base resume, past v1 runs for Phase 7
```

## Read first

1. `Resume_Pipeline_v2_Plan_of_Work.md` — the full design (in the data-plane folder). Sections §2 (architecture), §7 (Python/compute split), and §9 (contract template, screening-blindness enforcement) are binding.
2. `prompts/` in the repo — the canonical v1 prompts. Diff each against the project-knowledge copies; report any body-text drift before building invariants blocks from them.
3. Project knowledge: `Reyes_WorkHistory_v4.md` (data-plane seed) and `Pipeline-System-Summary.md` in `docs/` (best statement of design rationale).

## Decisions already made — do not re-open

- **Runtime:** Cowork skill, invoked as `/resume-fit`. Package via skill-creator as a .skill file.
- **Docx target:** default ATS-safe template (single column, standard fonts, no tables/text boxes/headers-footers). User approves a sample render in Phase 5 before it's locked.
- **Gate 1:** Gap Brief with the fixed three-option interrogation (stop / proceed with gaps on record / contest with evidence → WHD micro-interview → recompute). Never an autonomous numeric cutoff.
- **Subagent prompts:** five-section contract template — role / verbatim invariants / input manifest / output schema / refusal conditions. Compress only ritual and handoff scaffolding.
- **Screening blindness:** enforced by dispatch manifest + no file-read tools for the screening subagent + canary token in WHD front-matter with deterministic post-run scan. Prompt instruction is framing only.
- **Runtime model routing:** Sonnet-class for research and fit subagents; Opus-class for screening and synthesis.
- **Exception-driven interrogation:** triggers per plan §2 fire only on fundamental mismatches; ~3 question rounds per gate, overflow to punch list.
- **Repo disposition:** reuse, restructure contents, evict personal data. Do not start a fresh repo.

## Task 0 — repo preparation (inventory already done; do not re-audit)

Repo contents, verified before this handoff: six v1 prompts under persona names (verbatim vs. FINAL uploads plus metadata headers); two docs; TODO backlog; superseded JSX orchestrator (abbreviated prompts, hardcoded model, browser-side API calls); MIT license; README. Working tree shows four files "modified" — pure CRLF conversion, no content change (verified with `--ignore-cr-at-eol`).

Steps, in order:

1. Add `.gitattributes` (`* text=auto`), normalize line endings, commit. Clears the noise permanently.
2. Move `ui/career-pipeline-orchestrator.jsx` to `archive/` with a one-line deprecation note. Do not delete.
3. Mark `TODO.md` "Immediate" and "Work History Builder" sections as superseded by the v2 plan; leave Post-v1.0 items as roadmap.
4. Create `skill/` and `examples/` scaffolding. **No data-plane directories in the repo.**
5. Create `pipeline/whd|runs|fixtures/` in the data-plane folder.
6. Commit with a descriptive message. Never push without explicit user instruction.

## Build order and checkpoints

Execute plan §8 Phases 1–4 only. Checkpoint with the user at the end of each phase — show the artifacts, get an explicit go before the next. Phases 5–6 (finishing loop, WHD reconciliation) begin only after Phases 1–4 have been validated on a live JD.

- **Phase 1:** WHD restructure (index, stable anchors, changelog, canary token) applied to a copy of `Reyes_WorkHistory_v4.md` placed in `pipeline/whd/`. Also produce `skill/templates/whd-template.md` — the same structure, blank, for other users; the WHD schema is a public artifact even though Matt's WHD is not.
- **Phase 2:** Skill scaffold + Python helpers (score math, ATS scan, Gate 1 tally, YAML validation, plumbing) + data-plane config mechanism. Helpers are deterministic — unit-test with pytest before wiring. Build the synthetic `examples/` fixture here; use it, not Matt's data, for helper tests.
- **Phase 3:** Port Stage 0/1/2 into subagent contracts; wire parallel dispatch, Gap Brief, interrogation triggers, blindness enforcement. Integration-test against the synthetic fixture first, then against Matt's real resume + a real JD.
- **Phase 4:** Synthesis + report.md / appendix.md templates. End-to-end run on a real JD is the phase checkpoint.

## Public-release work (backlog — do not build now, do not forget)

For strangers to benefit, three things must exist that Phases 1–4 don't cover: an onboarding mode porting the Pre-Stage Career Documentarian interview into the skill (new users have no WHD); a README rewrite describing v2 install and use; and a pass stripping any incidental personal references from docs. Log these in `TODO.md` under a "v2 public release" heading during Task 0; schedule after Phase 7 validation.

## Inputs arriving from the user

- **Base resume:** `C:\Users\motorbikematt\OneDrive\Documents\Personal\Employment Stuff\Generic\Reyes_Matthew_AI-Product.docx` → user copies into `pipeline/fixtures/`. Needed from Phase 3 (real-data integration test); request at the Phase 2 checkpoint, not before.
- **Past v1 runs** (2–3 JDs + stage outputs): Phase 7 fixtures. If they appear in `pipeline/fixtures/`, leave untouched.
- **Repo visibility:** confirm public vs. private with the user at Task 0. The two-plane design assumes public; if private, nothing changes structurally.

## Working norms

- Code and anything generic → repo. Anything derived from Matt's history, resumes, or target jobs → data plane. When in doubt, data plane.
- No unrequested work: no README polish beyond the backlog entry, no speculative features, no Phase 5–6 preamble.
- Commit at each phase checkpoint with a descriptive message. Never push without explicit user instruction.
- When blocked or when a plan instruction is ambiguous, ask one precise question; do not guess and proceed.
