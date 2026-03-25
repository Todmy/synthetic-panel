# Synthetic Panel

Evaluate anything through a synthetic audience before it goes live. Zero dependencies. Self-configuring.

Synthetic Panel creates a matrix of **personas × contexts × metrics** — simulating how different people react to your content in different situations. It runs inside [Claude Code](https://docs.anthropic.com/en/docs/claude-code), spawns parallel evaluation agents, and produces a scored report with actionable risks and strengths.

## Install

Clone the skill into your project's `.claude/skills/` directory:

```bash
git clone https://github.com/Todmy/synthetic-panel.git .claude/skills/synthetic-panel
```

Or for personal use across all projects:

```bash
git clone https://github.com/Todmy/synthetic-panel.git ~/.claude/skills/synthetic-panel
```

The skill appears as `/synthetic-panel` in Claude Code.

## First Run

```
/panel
```

The skill detects missing data and starts an interactive setup:

```
What are you evaluating?

  1. Social media posts (LinkedIn, Twitter/X, newsletters)
  2. Code (PRs, architecture decisions)
  3. Product features / proposals
  4. Marketing copy (ads, emails, landing pages)
  5. Custom — I'll describe my use case
```

Pick a preset or describe your domain. The wizard generates personas, contexts, and metrics — ready in under a minute.

## Use Cases

### Content & Social Media

Test social media posts, newsletter drafts, or blog articles before publishing. The panel simulates your actual audience segments — who stops scrolling, who comments, who shares, who ignores.

```
/panel "Full-stack developer goes outdated. Now, there's something new..."

Reach Score: 67/100 | Buyer Score: 86/100

STRENGTHS:
  Sr Dev→Product (7%) — ego 4.5. Will comment about outsource reality.
  Dev Advocate (3%) — Will repost with their own take.
RISKS:
  HR Manager (3%) — Developer jargon excludes them entirely.
```

### Code Review

Simulate how your team would review a PR. Different reviewers catch different things — a security engineer spots vulnerabilities, a junior dev flags readability, an architect questions design choices.

```
/panel "PR: Refactor auth middleware to support OAuth2 + SAML..."

Quality Score: 58/100 | Risk Score: 72/100

STRENGTHS:
  Tech Lead — architectural_fit 7.2. Clean separation of concerns.
  DevOps — deployment impact low, no infra changes needed.
RISKS:
  Security Engineer — security_concerns 8.1. Token storage needs review.
  QA — test_coverage_need 7.5. No integration tests for SAML flow.
```

### Product Feature Proposals

Before pitching a feature to stakeholders, test how different roles react. CTOs care about feasibility, PMs about roadmap fit, sales about deal enablement, support about ticket reduction.

```
/panel "Proposal: AI-powered search that learns from user behavior..."

Approval Score: 45/100 | Priority Score: 38/100

STRENGTHS:
  End User — user_impact 6.8. Solves a daily frustration.
  Sales Rep — competitive_advantage 5.9. Differentiator in demos.
RISKS:
  CTO — feasibility 3.2. "Where's the technical spec? This is a wish list."
  Support — concerns: "Who handles when AI search gives wrong results?"
```

### Ad Copy & Marketing

Test headlines, email subject lines, landing page copy, or ad variations across customer segments. Find out which message resonates before spending ad budget.

```
/panel "Stop losing deals to competitors who demo better than you."

Impact Score: 52/100 | Conversion Score: 61/100

STRENGTHS:
  VP Sales persona — emotional_resonance 7.1. "This is my exact problem."
  Startup founder — click_intent 6.3. Wants to see the product.
RISKS:
  Enterprise buyer — trust_signal 2.8. "Sounds like every other SaaS pitch."
```

### Pitch Decks & Presentations

Test how investors, board members, or conference attendees would react to your slides or talking points.

### Internal Communications

Before sending a company-wide announcement (reorg, policy change, new tool rollout), simulate how different teams react.

### Job Descriptions

Test whether your job posting attracts the right candidates or scares them away. Simulate how senior devs, career changers, and passive candidates perceive it.

### Any Evaluation Scenario

If multiple people will judge your work, the panel can simulate them first. Define your personas, contexts, and metrics — the engine handles the rest.

## How It Works

**1. Parallel Agents** — Your personas are split into batches of 4. Each batch runs as a parallel Claude Code agent, evaluating content across all contexts simultaneously. A full 20×20 panel completes in ~2 minutes.

**2. Action-Value Scoring** — Not all engagement is equal. Each metric has a configurable weight reflecting its real-world impact:

```
action_value = Σ(metric × weight)
```

For social media: a comment (×5) drives more reach than a like (×1). For code review: requesting changes (×8) is a stronger signal than approving (×1). You define the weights.

**3. Hook Gate** — Evaluators can't engage with content they didn't read. If attention metrics score below threshold, action scores are proportionally suppressed:

```
hook_gate = min(1.0, attention / threshold)
```

**4. Log Scaling** — First few engaged personas matter most. Diminishing returns beyond the core audience:

```
score = k × ln(1 + predicted_reach)
```

**5. Calibration** — Feed real performance data to track accuracy:

```
"This post got 45K views, 89 reactions, 23 comments"
```

Tested on 12 social media posts: **Spearman rank correlation 0.951** (11 of 12 posts ranked within ±1 of real performance).

## After Evaluation

Everything is conversational. No separate commands needed:

```
"Why did the CTO score low?"           → analyzes evaluation data
"Here's a revised version: ..."         → re-evaluates + shows comparison
"Add real metrics: 45K views, 89 likes" → calibrates against reality
"How accurate is the panel?"            → shows correlation report
"Add a persona: QA Engineer"            → updates panel config
"Switch to code review preset"          → reconfigures the panel
"Show history"                          → lists past evaluations
```

## Reconfigure Anytime

```
/panel setup
```

Switch presets, add personas, change metrics, adjust weights.

## File Structure

```
.claude/skills/synthetic-panel/  ← the skill (cloned from this repo)
├── SKILL.md                     ← routing + init wizard (~500 words)
└── engine.md                    ← evaluation engine + presets (loaded on demand)

panel/                           ← auto-generated by init wizard
├── data/
│   ├── config.json              ← domain settings, weights, score names
│   ├── personas.json            ← evaluator profiles
│   ├── contexts.json            ← situational modifiers
│   └── metrics.json             ← evaluation dimensions with anchors
└── evaluations/                 ← scorecard reports (auto-generated)
```

All data files are plain JSON — edit them directly or through conversation.

## Requirements

- [Claude Code](https://docs.anthropic.com/en/docs/claude-code)
- That's it.

## License

MIT
