# How Scoring Works

## The Problem

Not all audience reactions are equal. One comment drives more distribution than five likes. One repost reaches more people than ten saves. A flat average across metrics misses this entirely.

## Action-Value Formula

Each metric has a configurable weight in `config.json`:

```json
{
  "action_weights": {
    "like_probability": 1,
    "comment_probability": 5,
    "repost_probability": 10,
    "save_probability": 2
  }
}
```

Per persona, we compute:

```
action_value = Σ(metric_score × weight)
```

## Hook Gate

People can't engage with content they didn't read. If attention metrics score below a threshold, action potential is suppressed:

```
hook_quality = mean(attention_metrics)
hook_gate = min(1.0, hook_quality / threshold)
gated_action = action_value × hook_gate
```

A persona with `hook_stop: 2` (didn't stop scrolling) gets their action_value cut in half, regardless of how good the content would be IF they read it.

## Predicted Reach

Audience-weighted sum of gated actions:

```
predicted_reach = Σ(persona.audience_share × persona.gated_action)
```

## Log Scaling

First few engaged personas matter most. Diminishing returns model:

```
score = k × ln(1 + predicted_reach)
```

The constant `k` (default: 29.2) is calibrated so that a strong evaluation (raw score ~10) maps to ~70/100.

## Two Scores

- **Primary Score** — all personas, audience-share weighted. "How will the broad audience react?"
- **Secondary Score** — only key decision-maker tiers, normalized by their share. "How will the people who matter most react?"

## Calibration

Feed real performance data after publishing:

```
"This post got 45K views, 89 reactions, 23 comments"
```

The panel computes Spearman rank correlation between synthetic scores and real performance. Tested on 12 LinkedIn posts: **r = 0.951** (11 of 12 ranked within ±1 position).

## Why Log Scale?

Linear scoring punishes niche content. A post that deeply resonates with 3 personas (30% of your audience) but is ignored by the rest would score ~30/100 on a linear scale. With log scaling, those 3 deeply engaged personas push the score to 60-70 — correctly reflecting that niche-but-deep content often outperforms broad-but-shallow content in real distribution.
