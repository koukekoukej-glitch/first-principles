# First Principles

A first-principles deep-analysis Skill. It breaks complex decisions (architecture choices, feasibility assessments, multi-option comparisons, risk audits) into auditable reasoning chains — matching each sub-question to a convergent or divergent strategy, then cross-validating the decomposition's completeness with an independent sub-Agent so that the strength of a conclusion never exceeds the strength of the evidence behind it.

Works with [Claude Code](https://docs.anthropic.com/en/docs/claude-code) and any LLM framework that supports custom System Prompts, file I/O, shell execution, and sub-Agent dispatch.

[中文版 README](README.md)

---

## Features

- **Two problem types, two strategies** — Convergent problems (the answer exists in the facts) get "systematic enumeration → elimination → verification." Divergent problems (the answer must be constructed) get "constraint identification → multi-candidate generation → evaluation." Misclassify a sub-question and both types fail
- **Decomposition before collection** — The decomposition defines the search space. Miss a dimension and no amount of collection can recover it. Decompose first, collect second
- **Independent sub-Agent validates decomposition completeness** — Spawn a clean-context sub-Agent, pass only the problem statement, have it independently list "what should be covered." Take the union with your own decomposition before collecting — an LLM in the same context cannot truly "ignore" what it has already written, so re-deriving in-place is formally independent but substantively identical
- **Bidirectional collection counters attention anchoring** — Each sub-question runs both ways: bottom-up (scan from existing material outward) and top-down (map from domain knowledge inward). The two directions have systematic complementary blind spots; only running them in parallel covers each other
- **Two execution modes** — **Deep Dive**: single Agent walks the full understand → decompose → collect → reason → present flow; suitable for most problems with non-trivial complexity. **Hunt**: every stage spawns sub-Agents in parallel + three-way cross-validation; reserved for "miss one and we fail"-level completeness needs (migration audits, security reviews, compliance checks)
- **Conclusion strength ≤ evidence strength** — Logically coherent does not equal factually established. No evidence → mark as assumption, don't disguise it as a conclusion. A conclusion that depends on an unverified assumption is itself an assumption
- **HTML report rendering** — Analysis output is an interactive HTML report with evidence cards, confidence tags, and collapsible reasoning chains

---

## Five Named Failure Modes

The design treats "complex analysis tends to fail" as five distinct necessary conditions (NC1–NC5) — each corresponding to a typical way "looks like analysis, actually drifting off." Every hard constraint in the skill annotates which condition it protects:

| # | Necessary Condition | What Failure Looks Like |
|---|---------------------|--------------------------|
| NC1 | Correct end-state understanding | Perfect analysis on the wrong problem (solving "how to launch on time" rather than "how to launch safely") |
| NC2 | Correct current-state understanding | Plan built on wrong premises (ignoring existing systems, redesigning from scratch what already exists) |
| NC3 | Information completeness | "Found some" mistaken for "found all"; feeling "good enough" after 5–8 discoveries |
| NC4 | Reasoning reliability | Each step in the logic chain looks right, but the chain as a whole drifts off-topic |
| NC5 | Communication failure | Conclusion is correct but the user cannot understand or execute it |

**General principle**: every step's failure mode is invisible to itself. When you misunderstand a problem you don't know you've misunderstood it; when you miss information you don't know you've missed it. So every step must have external grounding — user confirmation, sub-Agent independent derivation, self-challenge — and never trust the "looks right" feeling.

---

## Choosing a Mode

**Deep Dive** — One analyst walks the full pipeline, spawning a single sub-Agent for decomposition completeness validation, otherwise working alone.

Typical scenarios: "what's the root cause of this bug?", "how should this architecture be designed?", "is A or B the better option?", "why is this code slow?"

**Hunt** — Sub-Agents are spawned at every stage (scout, collect, analyze, adversarial review), with cross-validation and exhaustive coverage.

Typical scenarios: "migrate X system from A to B without missing any dependency", "audit the code for security issues", "evaluate which downstream systems this refactor affects"

**Default to Deep Dive when unsure** — Deep Dive can escalate to Hunt mid-flow if completeness pressure grows; Hunt cannot easily de-escalate.

---

## Workflow

```
Phase 1: Understanding
  ├─ Step 1  Understand & align            (NC1 + NC2)
  ├─ Step 2  Decompose & classify          (NC3 prerequisite)
  │   ├─ Decompose by the problem's natural structure (hypothesis space / matrix / tree / causal chain)
  │   ├─ Independent sub-Agent derivation comparison (hard gate)
  │   └─ Classify each sub-question (convergent / divergent)
  ├─ Step 3  Collect                       (NC3)
  │   ├─ Bottom-up: scan from existing material outward
  │   ├─ Top-down: map from domain knowledge inward
  │   └─ Gap audit (reuse step 2's independent checklist)
  └─ Step 4  Present cognitive baseline, request user confirmation

Phase 2: Analysis & Presentation
  ├─ Reasoning              (NC4)
  ├─ Adversarial review     (NC4)
  ├─ HTML report rendering  (NC5)
  └─ Iteration
```

Phases 1 and 2 are **loaded and executed step by step** — Phase 2 files are not read, and no conclusions are formed, until Phase 1 is complete and the user has confirmed the baseline. This gating itself protects NC1.

---

## Project Structure

```
first-principles/
├── SKILL.md                # Trigger conditions + success definition + discipline
├── derivation.md           # Design derivation (ADR; not loaded at runtime)
├── single-agent.md         # Deep Dive Phase 1
├── multi-agent.md          # Hunt Phase 1
├── phase-2-single.md       # Deep Dive Phase 2
├── phase-2-multi.md        # Hunt Phase 2
├── presentation.md         # Report presentation spec
├── report-template.html    # HTML report template
├── iteration-log.md        # Design iteration log
├── agents/                 # Sub-Agent prompts
│   ├── scout.md            # Scout (Hunt mode only)
│   ├── collector-bottom-up.md   # Bottom-up collector
│   ├── collector-top-down.md    # Top-down collector
│   ├── gap-scanner.md      # Gap scanner
│   ├── analyst.md          # Analyst
│   └── challenger.md       # Adversarial reviewer
└── sessions/               # Per-analysis working directory (generated at runtime; not tracked in this repo)
```

---

## Installation

```bash
git clone https://github.com/koukekoukej-glitch/first-principles.git ~/.claude/skills/first-principles
```

Claude Code auto-detects the trigger conditions. You can also load `SKILL.md` as a System Prompt in other frameworks.

### Triggering

Natural-language triggers:

```
> How should I design this module?
> Which is better, A or B?
> What risks does this architecture have?
> Help me decompose this problem.
> Evaluate the feasibility of X.
> /first-principles
```

Applicability criteria — the problem must satisfy at least one: ① needs decomposition into sub-questions, ② needs information collection before answering, ③ multi-option comparison, ④ requires an argument chain rather than a single answer. Problems answerable in 1–2 steps (look up a function, change a field) do not trigger.

---

## Design Principles

1. **Decomposition before collection** — A wrong decomposition cannot be fixed by collection
2. **Every step needs external grounding** — A step's failure mode is invisible to itself
3. **Independence comes from isolated attention pools** — Re-deriving in-place is not independent; you must use a clean-context sub-Agent
4. **No evidence, no conclusion** — Mark it as an assumption; don't disguise it
5. **Don't fabricate problems to fill the template** — If there's nothing the user genuinely needs to clarify, skip that section rather than inventing one for "thoroughness"

---

## License

[MIT](LICENSE)
