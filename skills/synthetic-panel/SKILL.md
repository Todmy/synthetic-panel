---
name: synthetic-panel
description: Use when you need feedback from multiple perspectives before publishing — test content, code, proposals, or copy against synthetic audience personas across situational contexts. Triggers on words like evaluate, panel, feedback, test draft, review proposal, audience reaction.
argument-hint: "<content text, file path, or 'setup' to reconfigure>"
allowed-tools:
  - Agent
  - Read
  - Write
  - Glob
  - Bash
---

# Synthetic Panel

Evaluate content through a synthetic audience: **personas × contexts × metrics** → parallel agents → scored report.

Works for any domain: social media posts, code reviews, feature proposals, ad copy, presentations, job descriptions — anything multiple people will judge.

If `panel/data/config.json` exists, read `config.name` and use it as the panel identity in all output.

---

## Routing

Check `$ARGUMENTS` and file state to decide what to do:

1. **No config exists** (Glob `panel/data/config.json` returns nothing) → **Init Wizard** below
2. **`$ARGUMENTS` = "setup"** → **Init Wizard** below (reconfigure)
3. **`$ARGUMENTS` has content + config exists** → **Run Evaluation** (read `panel-engine.md` in this directory for Steps 1-7)
4. **No arguments + config exists** → Print help: `"/panel <content or file path>" to evaluate. "/panel setup" to reconfigure.`

---

## Init Wizard

Run this when panel data doesn't exist or user requests setup. Ask one question at a time, wait for answer.

### W1. Panel Name

```
Give your panel a name (appears in reports).
Examples: "Social Media Audience", "Backend Code Review", "SaaS Feature Board"

Name:
```

### W2. Domain

```
What are you evaluating?

  1. Social media posts (LinkedIn, Twitter/X, newsletters)
  2. Code (PRs, architecture decisions)
  3. Product features / proposals
  4. Marketing copy (ads, emails, landing pages)
  5. Custom — I'll describe my use case
```

### W3. Generate Data

**If preset (1-4):** Read [engine.md](engine.md) → Presets section for the selected domain. Generate all 4 JSON files to `panel/data/` with full schemas. Create `panel/evaluations/` directory.

**If custom (5):** Ask conversationally:
1. "Describe what you're evaluating in one sentence." → `content_type`
2. "Who evaluates this? List 3-8 roles/types of people." → generate `personas.json`
3. "What situations affect their reaction? List 3-8 scenarios." → generate `contexts.json`
4. "What matters most? (clarity, accuracy, persuasiveness, impact...)" → generate `metrics.json`
5. "What should the two headline scores be called?" → suggest defaults, user can rename

Generate `config.json` with: name, domain, content_type, framing (2-3 sentence evaluator prompt), action_weights (metric→impact mapping), attention_metrics (first 2 metrics), score_names, score_interpretation (5 ranges).

All JSON schemas and generation rules are in [engine.md](engine.md) → Init Wizard Data Schemas section.

### W4. Confirm

```
✓ Panel "{name}" configured for {domain}
  {N} personas, {M} contexts, {K} metrics

  Run /panel <your content> to start evaluating.
```

If user included content in the original call, proceed to evaluation.

---

## Evaluation Engine

When routing decides to evaluate content, read [engine.md](engine.md) for the complete evaluation pipeline (Steps 1-7):

1. **Parse Input** — detect file path vs raw text, validate length
2. **Load Data** — read config + 4 JSON files, display preflight
3. **Build Prompts** — construct agent prompts with anti-bias framing from config
4. **Spawn Agents** — 5 parallel agents, progress reporting
5. **Aggregate** — action-value formula (weights from config), hook gate, log scaling
6. **Report** — Markdown scorecard to `panel/evaluations/`, terminal summary
7. **Converse** — discuss results, revise, calibrate, manage personas/contexts
