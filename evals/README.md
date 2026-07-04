# Skill Evals

How this repo measures whether its skills actually work: that they **trigger** when they should, **stay distinct** from each other, and **change agent behavior** the way each skill promises.

## Prior art (and what we adopted)

There is no single settled community standard for evaluating `SKILL.md` skills, but two approaches lead:

- **Anthropic's skill-creator v2** defines a per-skill `evals.json` (prompt + `expectations[]`, graded from the transcript) plus trigger-accuracy testing of descriptions against sample prompts. We adopt its [`evals.json` schema](https://github.com/anthropics/skills/tree/main/skills/skill-creator) **verbatim** for our behavioral tier, so any of its tooling (`run_eval.py`, benchmark comparisons, the eval viewer) works against our eval files unmodified.
- **Superpowers** (obra) tests skills with bash + `claude -p` + prompt fixtures and grader scripts. Our behavioral runner follows the same headless-`claude` pattern, with the grading rubric drawn from `expectations[]`.

What neither provides is a **deterministic, CI-safe** check for a multi-skill *catalog* — does each skill's description carry the vocabulary users actually say, and do two skills' descriptions collide? That's Tier 2 below, and it's this repo's addition.

## The three tiers

| Tier | What it checks | Runs | Cost |
|---|---|---|---|
| 1. Structural | Frontmatter, naming, required sections, command parity | CI (`validate-skills.js`, `validate-commands.js`) | Free |
| 2. Trigger & routing | Positive prompts rank their skill top-k; negative prompts don't; no two descriptions near-collide | CI (`run-evals.js`) | Free |
| 3. Behavioral | An agent following the skill satisfies its `expectations[]` | On demand (`run-evals.js --behavioral`) | Tokens |

Tier 2 is a **lexical approximation** of routing (stemmed TF-IDF over descriptions). It cannot judge semantics — that's Tier 3's job — but it catches the two failure modes that dominate real trigger bugs: a description missing the vocabulary users say (false negative), and an over-broad description that outranks the right skill (false positive). A Tier-2 failure usually means *fix the description*, not the eval.

## Running

```bash
# Tier 2 — deterministic, runs in CI
node scripts/run-evals.js

# Tier 3 — behavioral, runs each eval through headless claude, then grades it
node scripts/run-evals.js --behavioral test-driven-development            # spends tokens
node scripts/run-evals.js --behavioral test-driven-development --dry-run  # prints commands only
```

Tier 3 writes grading output to `evals/results/` (gitignored), in skill-creator's `grading.json` shape.

## Eval case format

One file per skill: `evals/cases/<skill-name>.json`.

```json
{
  "skill_name": "test-driven-development",
  "trigger": {
    "positive": [
      { "prompt": "Write a failing test for this bug before fixing it", "top_k": 3 }
    ],
    "negative": [
      { "prompt": "Update the architecture diagram in the docs" }
    ]
  },
  "evals": [
    {
      "id": 1,
      "prompt": "Fix the reported rounding bug in the invoice totals, test-first.",
      "expected_output": "A failing test demonstrating the bug, a minimal fix turning it green, full suite passing",
      "expectations": [
        "A failing test is written and shown failing before the fix",
        "The implementation is the minimum needed to pass",
        "The full suite is run after the fix to catch regressions"
      ]
    }
  ]
}
```

- `evals[]` is skill-creator's schema exactly (`id`, `prompt`, `expected_output`, optional `files[]`, `expectations[]`). Expectations are verifiable statements a grader checks against the transcript — behaviors, not phrasings.
- `trigger` is this repo's extension. `positive` prompts are realistic user asks that should route here (`top_k` defaults to 3; tighten to 1 for a skill's signature ask). `negative` prompts belong to a *different* skill; this skill must not rank first for them.

**Writing good trigger prompts:** paraphrase how users actually talk; don't copy the description (that's gaming the eval). If a realistic prompt can't rank because the description lacks its vocabulary, that is a real finding — improve the description.

## Adding a skill

Every skill ships with an eval file. When you add `skills/<name>/`, add `evals/cases/<name>.json` with at least 3 positive triggers, 2 negative triggers, and 1 behavioral eval. The runner currently reports missing eval files as warnings; this will become an error once the backlog of in-flight skill PRs clears.

## Metrics to watch

The Tier-2 run prints a **trigger rank-1 rate** (share of positive prompts that rank their skill first, not merely top-k). It isn't gated, but falling numbers mean descriptions are drifting toward each other. The collision check errors at ≥75% pairwise description similarity and warns at ≥50%.
