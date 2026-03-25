# Synthetic Panel — Engine Reference

This file contains the evaluation engine (Steps 1-7), data schemas for init wizard, and embedded presets.
Referenced by the main `panel.md` skill. Do not load this file unless performing an evaluation or init.

---

## Init Wizard Data Schemas

The init wizard in `panel.md` references these schemas when generating JSON files.


1. `panel/data/config.json` — copy the config template from the matching preset. Set `config.name` to `PANEL_NAME`.
2. `panel/data/personas.json` — generate a JSON array with the personas listed in the preset. Each persona MUST have ALL required fields:
   ```json
   {
     "id": "kebab-case-id",
     "name": "Human Readable Name",
     "audience_share": 0.XX,
     "tier": "core_engager|strategic_reader|passive|edge",
     "tier_label": "Tier N: Label",
     "demographics": {"age_range": "XX-XX", "location": "...", "occupation": "..."},
     "psychographics": {
       "big_five": {"openness": 0.X, "conscientiousness": 0.X, "extraversion": 0.X, "agreeableness": 0.X, "neuroticism": 0.X},
       "values": ["...", "..."],
       "decision_style": "..."
     },
     "engagement_trigger": "The question or anxiety that makes this persona pay attention",
     "evaluation_behavior": {
       "type": "active_engager|passive_reader|lurker|amplifier",
       "pattern": "Detailed behavioral description of how this persona interacts with content",
       "response_style": "How they express feedback (terse, detailed, sarcastic, supportive...)"
     }
   }
   ```
   Audience shares MUST sum to 1.00. Assign tiers based on how actively each persona engages (core_engager = will act, strategic_reader = evaluates carefully, passive = skims, edge = usually ignores).

3. `panel/data/contexts.json` — generate a JSON array with contexts from the preset. Each context MUST have:
   ```json
   {
     "id": "kebab-case-id",
     "name": "Human Readable Name",
     "description": "2-3 sentences describing the situation and HOW it changes evaluation behavior",
     "modifiers": {
       "attention_level": 0.0-1.0,
       "emotional_state": "descriptive text",
       "evaluation_speed": "slow|medium|fast",
       "quality_threshold": "low|medium|high"
     }
   }
   ```

4. `panel/data/metrics.json` — generate a JSON array with metrics from the preset. Each metric MUST have:
   ```json
   {
     "id": "snake_case_id",
     "category": "attention|emotion|action|quality|impact",
     "name": "Human Readable Name",
     "description": "What this measures. Include guidance for the evaluator.",
     "type": "score|text|label",
     "range": [0, 10],
     "anchors": {
       "0": "What 0 looks like (the default/common case)",
       "5": "What 5 looks like (above average)",
       "10": "What 10 looks like (exceptional, rare)"
     }
   }
   ```
   For `text` type: replace `range`/`anchors` with `"format": "description of expected text"`.
   For `label` type: replace `range`/`anchors` with `"options": ["option1", "option2", ...]`.

After writing all 4 files, use `mkdir -p panel/evaluations` to create the output directory.

Confirm:
```
✓ Panel "{PANEL_NAME}" configured for {domain}
  {N} personas, {M} contexts, {K} metrics
  Files: panel/data/config.json, personas.json, contexts.json, metrics.json

  Run /panel <your content> to start evaluating.
```

**Step W3b — If custom (5):**

Ask each question, wait for answer, then next:

1. "Describe what you're evaluating in one sentence."
   → Store as `CONTENT_TYPE`. Example: "Internal RFC proposals for engineering team"

2. "Who evaluates this? List the roles/types of people who will see it (3-8 people)."
   → For each role the user lists, generate a full persona object (same schema as W3a).
   → Infer Big Five scores from the role (e.g., security engineer = high conscientiousness, low agreeableness).
   → Infer engagement_trigger from the role (e.g., CTO = "Does this save money or create risk?").
   → Distribute audience_share equally if the user doesn't specify weights. Assign tiers.

3. "What situations affect how they react? List 3-8 scenarios."
   → For each situation, generate a full context object (same schema as W3a).
   → Infer attention_level and quality_threshold from the description.

4. "What matters most in the evaluation? (e.g., clarity, technical accuracy, persuasiveness, emotional impact)"
   → Generate 8-12 metrics based on the answer. Include:
     - 2 attention metrics (always first — used for hook gate)
     - 2-3 quality/emotion metrics
     - 3-4 action metrics (define which ones get action_weights in config)
     - 1-2 text metrics (free-form response, concerns)
   → Each metric gets anchors at 0/5/10.

5. "What should the two headline scores be called? The first measures overall reception, the second measures key decision-maker reception."
   → Default suggestions based on domain. User can accept or rename.
   → Example defaults: "Quality Score" / "Approval Score", "Impact Score" / "Conversion Score"

6. Generate `config.json` with:
   - `name`: PANEL_NAME
   - `domain`: derived from content_type
   - `content_type`: from step 1
   - `framing`: generate a 2-3 sentence system prompt for evaluator agents based on the domain. Pattern: "You are a {role} evaluating {content_type}. You are {traits}. Your DEFAULT reaction is {default}."
   - `scoring.action_weights`: assign weights 1-10 to the action metrics from step 4 (higher weight = rarer but more impactful action)
   - `scoring.attention_metrics`: first 2 metric IDs from step 4
   - `score_names`: from step 5
   - `score_interpretation`: generate 5 ranges (0-25, 26-45, 46-60, 61-80, 81-100) with domain-appropriate descriptions

7. Write all 4 JSON files (same as W3a). Create `panel/evaluations/`. Confirm setup.

**Step W4 — After any init:** If the user included content in the original `/panel` call alongside setup, proceed to Step 1. Otherwise stop and wait for content.


---

## Step 1: Parse Input

The user's input is in `$ARGUMENTS`:

```text
$ARGUMENTS
```

### 1a. Handle empty input

If `$ARGUMENTS` is empty or whitespace-only:
- Ask the user: "Provide the content to evaluate — paste text directly or give a file path (.md or .txt)."
- Stop and wait for a response. Do not proceed without input.

### 1b. Detect input type

Examine `$ARGUMENTS` to determine what the user provided. Apply these rules in order:

**File path**: If `$ARGUMENTS` matches a file path pattern (starts with `/`, `./`, `../`, `~`, or ends with `.md` or `.txt`):
1. Normalize the path: expand `~` to the user's home directory, resolve `./` and `../` relative to the repository root.
2. Use the Read tool to load the file. If the file does not exist, report: "File not found: {path}. Check the path and try again." — stop and wait.
3. The file contents become `DRAFT_TEXT`.

**Quoted string**: If `$ARGUMENTS` is wrapped in quotes (single or double), strip the outer quotes and treat the inner content as `DRAFT_TEXT`.

**Raw text**: Otherwise, treat the entire `$ARGUMENTS` as `DRAFT_TEXT`.

### 1c. Validate draft length

Count the characters in `DRAFT_TEXT`.

- **Under 50 characters**: Print warning — "Draft is very short ({N} chars) — evaluation may not be meaningful." Continue.
- **Over max length from config**: Print warning using `config.content_length.warning_long` template. Continue.
- **Between 50 and 3000 characters**: No warning needed.

### 1d. Confirm input

Print the first 80 characters of `DRAFT_TEXT` followed by "..." (if longer) so the user can verify the correct text was captured. Print total character count.

Store `DRAFT_TEXT` for use in all subsequent steps.

---

## Step 2: Load Panel Data

<!-- T007: Data Loading
     WHAT: Read all 4 JSON data files that the panel depends on.
     WHY: Every subsequent step needs this data — personas for agent batching,
          contexts for the evaluation matrix, metrics-schema for the agent prompt,
          calibration for the confidence indicator on the scorecard.

     INPUTS: File paths (hardcoded):
       - panel/data/personas.json    -> store as PERSONAS (array of 20 persona objects)
       - panel/data/contexts.json    -> store as CONTEXTS (array of 20 context objects)
       - panel/data/config.json          -> store as CONFIG (domain settings, action weights, score names)
       - panel/data/metrics.json           -> store as METRICS_SCHEMA (array of metric definitions)
       - panel/data/calibration.json -> store as CALIBRATION_DATA (array of calibration records)

     OUTPUTS:
       - PERSONAS, CONTEXTS, METRICS_SCHEMA, CALIBRATION_DATA variables populated
       - Preflight summary displayed (persona count, tier distribution, calibration status, eval plan)
       - Calibration status + correlation (when 4+ paired records exist)

     ERROR HANDLING:
       - If any file is missing, tell the user which file is missing and stop.
       - If JSON is malformed, report the parse error and stop.
       - If audience_share sum is outside 0.99–1.01, warn but continue.

     IMPLEMENTATION NOTES:
       - Use Read tool for each file (4 reads, can be parallel)
       - Parse JSON from file contents
       - Validate PERSONAS audience_shares sum to ~1.00
       - Count personas per tier for display
-->

### 2a. Read and validate data files

Read all four files in parallel:

| File | Store as | Fatal if missing/malformed |
|------|----------|---------------------------|
| `panel/data/personas.json` | PERSONAS | Yes |
| `panel/data/contexts.json` | CONTEXTS | Yes |
| `panel/data/config.json` | CONFIG | Yes |
| `panel/data/metrics.json` | METRICS_SCHEMA | Yes |
| `panel/data/calibration.json` | CALIBRATION_DATA | Yes |

If any file is missing or contains invalid JSON, report which file and stop.

After loading PERSONAS, sum all `audience_share` values. If outside 0.99-1.01, warn: `"audience_share sum = {sum} (expected ~1.00). Scores may be skewed."` Continue — do not stop.

### 2b. Compute calibration status

Count records in CALIBRATION_DATA where `synthetic_scores.reach_score` is not null -> `calibrated_count`.

| calibrated_count | Status | Detail |
|------------------|--------|--------|
| 0 | **Uncalibrated** | No synthetic scores yet. This run establishes baseline. |
| 1-3 | **Insufficient** | {calibrated_count}/{total_posts} calibrated. Need 4+ for correlation. |
| 4+ | **Calibrated** | {calibrated_count} posts calibrated. |

### 2b-corr. Compute calibration correlation (4+ paired records only)

If `calibrated_count >= 4`, compute Pearson r between synthetic `reach_score` and real `weighted_score` (normalized to 0-100 via `round(100 * ws / max_ws)`) for all records that have both values.

Use a Bash Python one-liner:

```bash
python3 -c "
import json, math
d = json.load(open('panel/data/calibration.json'))
pairs = [(r['synthetic_scores']['reach_score'], r['real_metrics']['weighted_score'])
         for r in d if r.get('synthetic_scores',{}).get('reach_score') is not None
         and r.get('real_metrics',{}).get('weighted_score') is not None]
mx = max(ws for _,ws in pairs)
pairs = [(s, round(100*ws/mx)) for s,ws in pairs]
n = len(pairs)
if n < 4: print('N/A'); exit()
sx,sy = [p[0] for p in pairs],[p[1] for p in pairs]
mx2,my2 = sum(sx)/n, sum(sy)/n
num = sum((x-mx2)*(y-my2) for x,y in pairs)
den = math.sqrt(sum((x-mx2)**2 for x in sx)*sum((y-my2)**2 for y in sy))
r = round(num/den,2) if den else 0
label = 'strong' if abs(r)>=0.7 else 'moderate' if abs(r)>=0.4 else 'weak'
print(f'r={r} ({label}) — based on {n} posts')
"
```

Parse the output. Store as `CALIBRATION_CORRELATION` (e.g. `"r=0.72 (strong) — based on 6 posts"` or `null` if fewer than 4 pairs). If the Python call fails, set `CALIBRATION_CORRELATION = null` and continue.

Store `CALIBRATION_STATUS`, `CALIBRATION_DETAIL`, and `CALIBRATION_CORRELATION` for the scorecard.

### 2c. Display preflight summary

```
--- Panel Preflight ----------------------------------------

  Personas:  {N} loaded
    Core Engagers:      {count} ({share}%)
    Strategic Readers:  {count} ({share}%)
    Passive:            {count} ({share}%)
    Edge:               {count} ({share}%)
  Contexts:  {N} loaded
  Metrics:   {N} ({n_score} scored, {n_text} text, {n_label} label, {n_int} integer)

  Calibration: {CALIBRATION_STATUS} — {CALIBRATION_DETAIL}
  {if CALIBRATION_CORRELATION} Correlation: {CALIBRATION_CORRELATION}

  Plan: {personas} x {contexts} = {cells} cells, {cells x metrics} data points
        Spawning 5 agents (4 personas each), ~2 min

------------------------------------------------------------
```

`{share}%` = tier's `audience_share` sum * 100, rounded to integer. Show tiers with 0 personas as "0 (0%)".

Proceed to Step 3.

---

## Step 3: Build Agent Prompts

<!-- T008: Agent Subagent Prompt Template
     WHAT: Define the exact prompt each Agent subagent receives. This is the core
           evaluation engine — the prompt must produce reliable, calibrated scores.
     WHY: Prompt quality directly determines score accuracy. Research (R2) identified
          5 bias mitigation techniques that MUST be embedded in the prompt:
          1. Anchor examples (what 1/10 and 9/10 look like)
          2. Chain-of-thought before scoring (reasoning -> number, not number -> rationalization)
          3. Critical/skeptical system framing ("busy professional, default is scroll past")
          4. Distribution forcing (avg 4-5/10, only 10-15% above 7)
          5. Per-metric anchors from METRICS_SCHEMA (0/5/10 reference points)

     INPUTS:
       - DRAFT_TEXT (the post to evaluate)
       - Batch of 4 persona objects from PERSONAS (different per agent)
       - CONTEXTS (all 20, same for every agent)
       - METRICS_SCHEMA (all 20 metric definitions with anchors)

     OUTPUT: A prompt string (AGENT_PROMPT) that will be passed to the Agent tool.
             The prompt instructs the agent to:
             - Evaluate DRAFT_TEXT from the perspective of its 4 assigned personas
             - For each persona, evaluate across all 20 contexts
             - For each persona-context pair, score all 20 metrics
             - Return structured JSON: array of 80 evaluation objects
               (4 personas x 20 contexts), each containing:
               { persona_id, context_id, reasoning, metrics: { all 20 metric values } }

     CRITICAL PROMPT ELEMENTS:
       - System identity: "You are evaluating content from the perspective
         of {N} audience personas."
       - Skeptical framing: "You are a busy, skeptical professional. Your default is to
         scroll past. Most content is noise. A 5/10 means average, not bad."
       - Chain-of-thought: "Before scoring, explain in one sentence WHY this persona
         would react this way in this context."
       - Distribution forcing: "Average scores across all contexts should be 4-5/10.
         Only 10-15% of scores should exceed 7."
       - Output format: JSON array with exact field names matching metrics-schema IDs
       - Include full persona objects (demographics, psychographics, engagement triggers,
         evaluation_behavior) — not just names
       - Include full context objects (modifiers, attention levels) — not just names
       - Include metric anchors so the agent knows what each score level means
       - For text metrics (attention_loss_point, simulated_comment, comment_type,
         primary_emotion, author_perception), include the valid options from schema

     IMPLEMENTATION NOTES:
       - The prompt is a template — persona batch is injected per agent
       - Keep the prompt under ~8K tokens to avoid context rot in the agent
       - Metrics schema provides the anchors; embed them directly in the prompt
       - Simulated comments should be written in the persona's voice/style
-->

### 3a. Template variables

The prompt template uses these placeholders, injected at dispatch time (Step 4):

| Placeholder | Source | Per-agent or shared |
|-------------|--------|---------------------|
| `{DRAFT_TEXT}` | From Step 1 | Shared |
| `{PERSONA_BATCH_JSON}` | 4-persona slice of PERSONAS | Per-agent |
| `{CONTEXTS_JSON}` | Full CONTEXTS array | Shared |
| `{METRICS_SCHEMA_JSON}` | Full METRICS_SCHEMA array | Shared |
| `{N_PERSONAS}` | Batch size (4) | Per-agent |
| `{N_TOTAL}` | 4 x 20 = 80 | Per-agent |

### 3b. Agent prompt template

Build `AGENT_PROMPT_TEMPLATE` as the following string. Replace placeholders with actual values at dispatch time.

````
{CONFIG.framing}

You are NOT a helpful assistant giving feedback. You simulate real evaluators — busy, skeptical, self-interested. Their DEFAULT reaction is to dismiss, skip, or give minimal attention.

=== POSITIVITY BIAS WARNING ===

The #1 failure mode of LLM evaluators is positivity bias: scores cluster at 6-8 when reality clusters at 2-4. You MUST fight this actively.

SCORE RANGES — what they mean:
- 0-2/10: DEFAULT for personas where the post is outside their domain. They scroll past without reading. Common for non-target segments.
- 3-4/10: The post mildly registers — persona glances, maybe reads the hook, moves on. This is the baseline for tangentially relevant content.
- 5-6/10: ABOVE AVERAGE. The post earned a real pause, a full read, or a micro-reaction. The persona's engagement trigger was at least partially activated.
- 7-8/10: STRONG. The post directly hits the persona's identity or trigger. They read fully, feel something, consider acting. Up to 15-20% of evaluations can reach here IF the post is genuinely good for that persona.
- 9-10/10: EXCEPTIONAL. The persona is compelled to act — comment, save, screenshot. Reserve for persona-context pairs where the post perfectly intersects identity + trigger + context. Up to 5% of evaluations.

HOOK CALIBRATION — what 2/10 and 8/10 actually look like:

2/10 hooks (NORMAL — this is the baseline, not a failure):
- "5 lessons I learned about leadership this year" → Generic listicle. Thumb keeps moving.
- "Excited to share that our team just launched..." → Nobody outside the company cares.
- "AI is changing everything. Here's how to prepare." → Could be any of 10,000 posts. Scroll.

8/10 hooks (RARE — involuntary physical stop):
- "I got fired yesterday. Here's the Slack message." → Pattern interrupt + vulnerability + specificity.
- "We mass-adopted Copilot across 200 devs. Lost $400K in 3 months." → Concrete number + failure + directly relevant.
- "My salary dropped 40% when I moved from outsource to product. Best decision I ever made." → Counterintuitive + specific number.

The difference: 2/10 hooks are templates anyone could write. 8/10 hooks contain something specific, unexpected, or identity-threatening. That involuntary stop almost never happens.

=== POST DRAFT ===

{DRAFT_TEXT}

=== PERSONAS ===

Read each profile carefully — demographics, psychographics, evaluation_behavior, and especially the engagement_trigger field.

The engagement_trigger is the anxiety or question that makes this persona stop scrolling. For EVERY evaluation, explicitly check: does this post HIT this persona's engagement_trigger? Not "vaguely related" — does it poke the nerve? If not, default scores are 0-3.

Inhabit each persona before scoring. Picture their office, their screen, what tab they just closed. You are not evaluating "on behalf of" this person — you ARE this person for the duration of the evaluation.

{PERSONA_BATCH_JSON}

=== CONTEXTS ===

Each persona encounters the post in each of these 20 contexts. Context affects attention, mood, and willingness to engage. "Morning commute, half-distracted" produces very different scores from "dedicated learning time at desk."

{CONTEXTS_JSON}

=== METRICS ===

For each persona-context pair, evaluate ALL metrics below. The anchors define what 0, 5, and 10 look like — the 0-anchor is the DEFAULT for most content. Use anchors as calibration reference.

{METRICS_SCHEMA_JSON}

=== EVALUATION PROCESS ===

For each of your {N_PERSONAS} personas, iterate through all 20 contexts. For each persona-context pair:

STEP 1 — REASONING (mandatory, before ANY score):
Write the "reasoning" field FIRST. It must answer three things:
1. What is this persona physically doing in this context? (e.g., "Scrolling during a sprint review that went long, half-listening to a PM")
2. Does this post trigger the persona's engagement_trigger? Quote the trigger, give an honest yes/no. "Adjacent" or "vaguely related" = NO.
3. Most likely behavioral outcome? (Usually: "Scrolls past" or "Pauses, reads 3 lines, scrolls")

The reasoning determines the scores. Do NOT pick numbers first and rationalize after. If your reasoning says "scrolls past" but your hook_stop is 5, something is wrong — fix the score, not the reasoning.

STEP 2 — SCORE all 20 metrics based on your reasoning.

Per-metric guidance (supplements the anchors in METRICS_SCHEMA):

Scored (0-10):
- hook_stop: Default for 60%+ of content is 0-2 (thumb keeps moving).
- read_through: Most people bail after 3-4 lines even if they stopped.
- emotion_strength: Most content produces 1-3. Above 5 only if persona would still feel it an hour later.
- ego_engagement: Fires when professional identity is challenged. The #1 comment driver.
- Action metrics (from CONFIG.scoring.action_weights): Score based on how likely the persona is to take each action. Higher-weighted actions in config are rarer and more impactful.
- profile_click/follow_probability: Following from a single post is extremely rare.
- screenshot_send, tell_offline, remember_tomorrow: Harshest metrics. Most content scores 0-2.
- purchase_intent: 0 for 80%+ of pairs. Above 3 ONLY if persona is a plausible buyer AND post demonstrated specific relevant competence.

Text:
- attention_loss_point: Name the line/section where the persona bails. "never" should be uncommon.
- Text response metrics (simulated_comment or equivalent): Write AS the persona — match their voice and style. Real responses are often brief, half-formed, and self-serving. If the relevant action probability < 3, output "would not respond."
- comment_word_count: Word count of simulated_comment. 0 if "would not comment".

Label:
- primary_emotion: ONE of: curiosity, anxiety, validation, disagreement, inspiration, boredom, irritation, admiration, fear, indifference. "Indifference" and "boredom" are the most common honest answers.
- comment_type: ONE of: agreement, disagreement, personal_experience, question, ego_display, addition, would_not_comment.
- author_perception: ONE of: expert, thought_leader, provocateur, practitioner, salesman, irrelevant, bullshitter. "Irrelevant" is the most common honest answer.

=== DISTRIBUTION GATE (mandatory self-check before output) ===

After completing all evaluations, BEFORE outputting, run this check:

1. Compute mean of each scored metric across all persona-context pairs.
2. Compute overall mean across all scored metrics.
3. CHECK: Is overall mean between 2.5 and 5.0? If above 5.0, you are too generous — lower scores for non-target personas. If below 2.5, you are too harsh — raise scores for personas whose engagement_trigger is genuinely hit by the post.
4. CHECK: Are 8+ scores under 10% of all scored cells? If not, demote the weakest 8+ scores.
5. CHECK: Are 9-10 scores under 5% of all scored cells? If not, demote to 7-8.

KEY CALIBRATION RULE: Good content should produce mean scores of 3.5-5.0 across TARGET personas (those whose engagement trigger matches), while non-target personas stay at 0-2. The overall mean will be lower because non-target segments dilute it — that's correct. Don't lower target-persona scores to hit an overall mean target.

Only output after passing all checks.

=== OUTPUT FORMAT ===

Return a single JSON array with exactly {N_PERSONAS} x 20 = {N_TOTAL} objects. No text before or after the JSON. No markdown fences. Raw JSON only.

[
  {
    "persona_id": "cto_mid_tier",
    "context_id": "morning_commute",
    "reasoning": "Sunday evening, pre-week anxiety. Trigger is 'Am I stuck in outsource forever?' — this post attacks that exact nerve with 'developers who only code are obsolete.' Direct hit. Full read, considering a comment.",
    "metrics": {
      "hook_stop": 6,
      "read_through": 6,
      "attention_loss_point": "binary choice at end — felt reductive but already read everything",
      "primary_emotion": "anxiety",
      "emotion_strength": 4,
      "ego_engagement": 5,
      "like_probability": 4,
      "comment_probability": 3,
      "repost_probability": 0,
      "save_probability": 3,
      "profile_click_probability": 2,
      "follow_probability": 1,
      "simulated_comment": "Ну ок Netflix це Netflix. У аутсорсі ніхто тобі не дасть 'own problems'. Різні реальності.",
      "comment_word_count": 14,
      "comment_type": "disagreement",
      "screenshot_send": 2,
      "tell_offline": 2,
      "remember_tomorrow": 3,
      "author_perception": "provocateur",
      "purchase_intent": 0
    }
  }
]

NOTE: The example above shows TYPICAL scores (mostly 0-2). This is what a normal evaluation looks like. High scores are the exception.

Produce all {N_TOTAL} objects. Do not skip any persona-context pair. Do not abbreviate with "..." or similar. Every object must contain all 20 metric fields.
````

### 3c. Inject per-agent data

For each of the 5 agents, create a concrete prompt by replacing placeholders in `AGENT_PROMPT_TEMPLATE`:

- `{DRAFT_TEXT}` -> full `DRAFT_TEXT` from Step 1
- `{PERSONA_BATCH_JSON}` -> JSON array of 4 persona objects for this agent (all fields: id, name, tier, audience_share, demographics, psychographics, engagement_trigger, evaluation_behavior)
- `{CONTEXTS_JSON}` -> full CONTEXTS array (same for every agent)
- `{METRICS_SCHEMA_JSON}` -> full METRICS_SCHEMA array (same for every agent; includes id, name, description, type, range/options/format, anchors)
- `{N_PERSONAS}` -> `4`
- `{N_TOTAL}` -> `80`

Store the 5 constructed prompts as `AGENT_PROMPTS[1..5]` for use in Step 4.

---

## Step 4: Spawn Agents and Collect Results

<!-- T009: Parallel Agent Orchestration
     WHAT: Split PERSONAS into 5 batches of 4, spawn 5 Agent subagents in parallel,
           collect results as they complete, handle failures.
     WHY: 5 parallel agents reduce wall-clock time from ~10min to ~2min.
          Each agent evaluates 4 personas x 20 contexts = 80 evaluation cells.
          Total: 5 agents x 80 cells = 400 evaluations.
-->

### 4a. Launch

Print the launch banner listing all 5 agents with their persona names (from PERSONAS `name` field), then spawn all 5 agents in a **single response** using Agent tool with `run_in_background: true` and `AGENT_PROMPTS[1..5]` from Step 3c.

```
--- Launching Evaluation Agents ----------------------------

  Agent 1 (Personas 1-4):   {name1}, {name2}, {name3}, {name4}
  Agent 2 (Personas 5-8):   {name5}, {name6}, {name7}, {name8}
  Agent 3 (Personas 9-12):  {name9}, {name10}, {name11}, {name12}
  Agent 4 (Personas 13-16): {name13}, {name14}, {name15}, {name16}
  Agent 5 (Personas 17-20): {name17}, {name18}, {name19}, {name20}

  Total: 400 evaluations across 5 parallel agents (~2 min)

------------------------------------------------------------
```

### 4b. Progress reporting

As each background agent returns, **immediately** print a progress line. No silent gaps.

On success:
```
  Agent {done}/5 done (Personas {first}-{last}) — {batch_evals} evals | {evals_collected}/400 total
```

On failure:
```
  ✗ Agent {done}/5 FAILED (Personas {first}-{last}) — {error} | {evals_collected}/400 total
```

### 4c. JSON extraction from agent response

Agent output may contain markdown fences, explanation text, or other wrapping. Extract the JSON array using this sequence — stop at the first step that yields a valid array:

1. **Strip fences**: if response contains `` ```json `` or `` ``` ``, extract content between first opening and matching closing fence.
2. **Find array boundaries**: locate first `[` and last `]`, extract that substring (inclusive).
3. **Parse**: parse as JSON. Must be an array.

If parsing fails entirely, go to 4d (failure handling).

**Validate each object** — must have `persona_id` (matching one of this agent's 4 IDs), `context_id` (matching one of 20 context IDs), `reasoning` (string), `metrics` (object with 20 keys per METRICS_SCHEMA).

**If fewer than 80 valid objects**: identify missing persona-context pairs, insert placeholders with `"status": "evaluation_failed"` and all metrics set to `null`. Print: `"  (!) {batch_evals}/80 evals returned. {missing} cells marked failed."`

### 4d. Failure handling

Three failure modes, handled in order:

**Agent crash / timeout / no JSON at all:**
- Mark as failed. Insert 80 placeholder objects (all `"status": "evaluation_failed"`, null metrics).

**Malformed JSON (text looks like JSON but doesn't parse):**
- Attempt recovery: split on `}\s*,\s*{`, wrap fragments individually, parse each. Keep any that validate.
- Print: `"  Recovered {kept}/80 cells from malformed response."`
- Insert placeholders for the rest.

**Incomplete results (valid JSON, <80 objects):**
- Already handled in 4c — placeholders for missing pairs.

**Rules:**
- Never discard successful agents because another failed.
- Never block collection waiting for a slow agent.
- If 0/5 succeeded and 0 cells recovered: report total failure and stop.

### 4e. Completion and retry offer

Once all 5 agents have returned, print:

**All succeeded:**
```
--- All 5 agents complete. {evals_collected}/400 evaluations. Proceeding to aggregation. ---
```

**Some failed:**
```
--- {succeeded}/5 succeeded, {failed}/5 failed. {evals_collected}/400 evaluations. ---

  Failed: Agent {N} (Personas {first}-{last}), ...

  Retry failed batches? (yes/no)
```

On retry: re-spawn only failed agents (same prompts, parallel, `run_in_background: true`). Replace placeholders with real results on success. One retry round only — if retry also fails, keep placeholders and proceed.

On skip: `"Proceeding with {evals_collected}/400 evaluations. Missing cells excluded from aggregation."`

### 4f. Finalize

Store merged array as `AGENT_RESULTS` (400 objects — valid evaluations + any `"evaluation_failed"` placeholders). Compute `VALID_EVALS` and `FAILED_EVALS` counts for Step 5.

Proceed to Step 5.

---

## Step 5: Aggregate Scores

<!-- T010: Aggregation Logic
     WHAT: Compute Reach Score, Buyer Score, per-category averages, tier breakdowns,
           text metric distributions, and identify top risks/strengths.
     INPUTS: AGENT_RESULTS (400 cells), PERSONAS (audience_share, tier)
     FORMULAS: contracts/skill-interface.md
     KEY RULE: Exclude evaluation_failed cells from ALL averages. Denominator = valid cell count, never 400.
-->

### 5a. Filter failed cells

Separate `VALID_RESULTS` (cells without `status: "evaluation_failed"`) from failures. All averages below use only valid cells with per-computation denominators.

If `VALID_RESULTS` < 360 (90%), warn: "Scores based on {count}/400 evaluations ({pct}% coverage)."

### 5b. Per-persona averages

For each persona `p`: collect its valid cells, count as `N_p`.
- `N_p == 0`: persona excluded from Reach, Buyer, tiers, risks, strengths. Shows "N/A" in summary.
- `N_p > 0`: compute `PERSONA_AVERAGES[p.id][m] = mean(m across valid cells)` for each numeric metric (15 total: all `type: "score"` + `comment_word_count`). Text/label metrics excluded from numeric averages.
- `N_p < 15`: flag "Averages for {name} based on {N_p}/20 contexts."

### 5c. Reach Score (0-100)

Uses **action-value formula** — weights actions by their real distribution impact (comments > reposts > saves > likes), gates by hook quality, and applies logarithmic scaling to model diminishing returns.

**Step 1: Per-persona action value** (non-null only):
```
action_value = Σ(metric_score × CONFIG.scoring.action_weights[metric]) for each weighted metric
```
Weights reflect real platform mechanics: one comment drives more distribution than five likes.

**Step 2: Hook gate** — if the persona doesn't stop scrolling, actions can't happen:
```
hook_quality = mean(CONFIG.scoring.attention_metrics)  # e.g., mean(hook_stop, read_through) for social media
hook_gate = min(1.0, hook_quality / CONFIG.scoring.hook_gate_threshold)
gated_action = action_value × hook_gate
```

**Step 3: Predicted reach** — audience-weighted sum of gated actions:
```
predicted_reach = SUM(persona.audience_share × persona.gated_action)
```

**Step 4: Log scaling** — models diminishing returns (first 3 engaged personas matter most):
```
REACH_SCORE = round(min(100, CONFIG.scoring.log_scale_constant × ln(1 + predicted_reach)))
```
The constant 29.2 is calibrated so that raw=10 (strong post) → score=70.

**Interpretation** (one sentence based on range):
- 0-25: Weak. Few personas engage, low action intensity.
- 26-45: Moderate. Some personas read and like, few comment.
- 46-60: Strong. Multiple personas comment and save, distribution amplifies.
- 61-80: Excellent. Core audience comments actively, reposts drive new segments.
- 81-100: Exceptional. Broad engagement across tiers with strong viral actions.

### 5d. Buyer Score (0-100)

Buyer personas = `tier` in `("core_engager", "strategic_reader")`. Read tier from persona objects at runtime.

Per buyer persona (non-null only), same action-value formula:
```
buyer_action = action_value × hook_gate    (same as 5c)
buyer_reach = SUM(buyer_persona.audience_share × buyer_action)
buyer_share = SUM(buyer_persona.audience_share)
buyer_normalized = buyer_reach / buyer_share    (per-unit action for buyer segment)
BUYER_SCORE = round(min(100, 29.2 × ln(1 + buyer_normalized)))
```
Denominator normalizes because Tier 1+2 shares don't sum to 1.0. If all buyer evaluations failed: `BUYER_SCORE = "N/A"`.

**Interpretation** (same pattern as Reach): 0-15 irrelevant to buyers, 16-30 mild recognition, 31-50 attention but no action, 51-70 strong business value, 71-100 exceptional (verify).

**Gap analysis**: If `|BUYER_SCORE - REACH_SCORE| > 10`, note which direction and why (buyers-only vs. broad appeal mismatch).

### 5e. Per-category averages (0-10 scale)

Pool all valid cells. For each category, average its numeric metrics:

| Category | Metrics |
|----------|---------|
| Attention | hook_stop, read_through |
| Emotion | emotion_strength, ego_engagement |
| Action | like_probability, comment_probability, repost_probability, save_probability |
| Engagement Quality | comment_word_count |
| Virality | screenshot_send, tell_offline, remember_tomorrow |
| Brand Effect | profile_click_probability, follow_probability, purchase_intent |

Generate one-line interpretation per category. Key thresholds:
- **Attention**: 0-2 feed noise, 2-4 some pause, 4-6 good capture, 6+ exceptional.
- **Action**: 0-2 typical engagement rates, 2-4 above average, 4+ unusual.
- **Virality**: 0-1 dies on platform, 1-3 some dark-social, 3+ breaks out.
- **Brand Effect**: 0-1 no impact, 1-3 some brand building, 3+ strong.

### 5f. Text and label metric aggregation

These cannot be averaged. Aggregate by frequency across all valid cells:

- **primary_emotion**: frequency distribution, display top 3. Format: `"indifference (187), boredom (94), curiosity (52)"`
- **author_perception**: full frequency distribution across all 7 options.
- **comment_type**: full frequency distribution across all 7 options.
- **attention_loss_point**: frequency of each unique value, display top 3 most common.
- **simulated_comment**: select 3 best comments from cells where comment != "would not comment". Rules: 3 different personas, prefer different tiers, prefer longest. Store as `BEST_COMMENTS` with persona_name, context_id, comment_text.

### 5g. Tier breakdown

For each tier (core_engager, strategic_reader, passive, edge):
1. Average `reach_composite` across tier's personas (non-null only).
2. Compute tier's `audience_pct` from summed `audience_share`.
3. Find tier's strongest and weakest metric.
4. Generate tier-specific `action_note`:
   - **Core Engagers**: Are ego_engagement + comment_probability high? "Will comment — protect this" vs. "Not triggered."
   - **Strategic Readers**: Is purchase_intent > 2.0? "Buyers considering you" vs. "Doesn't demonstrate payable competence."
   - **Passive**: Is hook_stop > 3.0? "Hook works broadly" vs. "Scrolls past completely."
   - **Edge**: Any metric > 4.0? "Unexpected spike on {metric}" vs. "Disengaged. Normal."

### 5h. Top 3 risks

Rank personas by `risk_signal = audience_share x (10 - reach_composite)`. Top 3 = biggest audience losses.

For each risk, find the persona's weakest metric and a representative reasoning from AGENT_RESULTS (cell where that metric scored lowest).

**Format** — must be actionable:
```
RISK {n}: {name} ({share}%) — {weakest_metric} avg {val}/10
WHY: {one sentence from reasoning, referencing the persona's engagement_trigger}
CONSIDER: {specific change to the draft — reference the trigger, not generic "improve the hook"}
```

### 5i. Top 3 strengths

Rank personas by `strength_signal = audience_share x reach_composite`. Top 3 = biggest audience wins.

For each strength, find the persona's strongest metric and best reasoning.

**Format** — must tell author what to protect:
```
STRENGTH {n}: {name} ({share}%) — {strongest_metric} avg {val}/10
WHY: {one sentence from reasoning}
PROTECT: {what NOT to cut in revisions — specific element of the draft}
```

### 5j. Per-persona summary table

20 rows sorted by `audience_share` descending. Columns: name, tier, share%, valid_contexts (of 20), hook_stop, read_through, action_composite (mean of like/comment/repost), top_emotion, top_perception. Round to 1 decimal. "N/A" for null personas.

### 5k. Quality checks

Before proceeding:
1. **Inflation**: If unweighted mean of `reach_composite` across personas > 5.0, warn about possible score inflation.
2. **Tier skew**: If highest tier avg > 2x lowest tier avg, note the post is polarizing across segments.

Proceed to Step 6.

---

## Step 6: Generate Report and Display Summary

<!-- T011: Report Generation
     WHAT: Write a full Markdown scorecard to panel/evaluations/ and display a compact summary in the conversation.
     WHY: The file is the persistent record (history, comparison, deep-dive). The terminal summary gives immediate feedback.
     INPUTS: DRAFT_TEXT, REACH_SCORE, BUYER_SCORE, all Step 5 aggregations, AGENT_RESULTS, CALIBRATION_STATUS/DETAIL, timestamp.
     OUTPUTS: (1) Markdown file at panel/evaluations/YYYY-MM-DD-HHMMSS-{slug}.md, (2) terminal summary under 30 lines.
-->

### 6a. Generate filename

Get timestamp via Bash: `date '+%Y-%m-%d-%H%M%S'`. Build slug: first 5 words of DRAFT_TEXT, lowercased, kebab-case, max 50 chars. Ensure `panel/evaluations/` exists (`mkdir -p`).

`REPORT_PATH = panel/evaluations/{TIMESTAMP}-{slug}.md`

### 6b. Build and write report file

Use the Write tool to save a Markdown file to `REPORT_PATH` with the following sections in order. Replace all placeholders with computed values from Steps 1-5.

**Frontmatter** (YAML-like, `---` fenced). Machine-parseable for "show history":
```
---
title: {first 80 chars of DRAFT_TEXT}
date: {YYYY-MM-DD HH:MM:SS}
reach_score: {REACH_SCORE}
buyer_score: {BUYER_SCORE}
calibration_status: {uncalibrated|insufficient|calibrated}
calibration_correlation: {CALIBRATION_CORRELATION or null}
valid_cells: {VALID_EVALS}
total_cells: 400
draft_length: {char count}
---
```

**Sections** (in this order):

1. **Header**: `# Panel Evaluation: {first 80 chars}` + Date, Coverage line, and Calibration line: `**Calibration**: {CALIBRATION_STATUS}` — if `CALIBRATION_CORRELATION` is not null, display as `**Calibration**: {CALIBRATION_CORRELATION}` (e.g. `**Calibration**: r=0.72 (strong) — based on 6 posts`); otherwise show `**Calibration**: {CALIBRATION_STATUS} — {CALIBRATION_DETAIL}`.
2. **Scores**: Table with Reach/Buyer scores + interpretation (from 5c/5d ranges). Add `**Gap**` line if |BUYER - REACH| > 10.
3. **Category Averages**: Table — 6 rows (Attention, Emotion, Action, Engagement Quality, Virality, Brand Effect) with avg (0-10) + one-liner interpretation from 5e.
4. **Top 3 Risks**: Each risk as bold header `{persona} ({share}%) — {metric} avg {val}/10`, then WHY (from reasoning + engagement_trigger) and CONSIDER (specific draft change). Format from 5h.
5. **Top 3 Strengths**: Same format but WHY + PROTECT (what not to cut). Format from 5i.
6. **Tier Breakdown**: Table — 4 rows. Columns: Tier, Personas count, Audience %, Avg Reach Composite, Strongest/Weakest Metric, Action Note from 5g.
7. **Text Metric Distributions**: Primary Emotion (top 3), Author Perception (full dist), Comment Type (full dist), Attention Loss Point (top 3), Best 3 Simulated Comments (persona + context + text).
8. **Per-Persona Summary**: Dense 20-row table sorted by audience_share desc. Columns: #, Persona, Tier, Share%, Ctxs (/20), Hook, Read, Emot, Ego, Action (mean of like/comment/repost/save), Viral (mean of screenshot/tell/remember), Brand (mean of profile/follow/purchase), Top Emotion, Perception. All rounded to 1 decimal. "N/A" for null personas.
9. **Detailed Results**: One `<details>` per persona (sorted by audience_share desc). Summary line: `<strong>{name}</strong> — {tier}, {share}% | Hook {avg}, Read {avg}, Action {avg}`. Inside each:
   - Engagement Trigger (from PERSONAS)
   - Score Heatmap: 20-row table (one per context) with all numeric + label metrics, scores rounded to integers
   - Strongest/Weakest context (by reach composite)
   - Top 3 Reasoning entries (cells with highest variance from persona mean)
   - Simulated Comments table (contexts where comment_probability >= 3)
10. **Evaluated Draft**: `<details>` with full DRAFT_TEXT for later reference.
11. **Footer**: `*Generated by Synthetic Audience Panel. {VALID_EVALS}/400 cells. {FAILED_EVALS} failed.*`

### 6c. Display terminal summary

Print directly in conversation (do NOT read the file back). Under 30 lines. Every line must be a number, a risk to fix, or a strength to protect.

```
--- Panel Results ------------------------------------------
{if CALIBRATION_CORRELATION} Calibration: {CALIBRATION_CORRELATION}

Reach Score: {REACH_SCORE}/100 ({terse interpretation})
Buyer Score: {BUYER_SCORE}/100 ({terse interpretation})
{If |BUYER-REACH|>10: "Gap: {one sentence}"}

RISKS:
1. {name} ({share}%) — {plain language problem}. Consider: {specific draft change}.
2. {name} ({share}%) — {problem}. Consider: {change}.
3. {name} ({share}%) — {problem}. {Consider: or Accept: if low-value persona}.

STRENGTHS:
1. {name} ({share}%) — {metric} {val}. {Predicted behavior: "They WILL comment" / "Goes into reference folder" / "They'll share with their team"}.
2. {name} ({share}%) — {metric} {val}. {behavior}.
3. {name} ({share}%) — {metric} {val}. {behavior}.

TIERS:
  Core Engagers ({pct}%):     {action_note from 5g}
  Strategic Readers ({pct}%): {action_note}
  Passive ({pct}%):           {action_note}
  Edge ({pct}%):              {action_note}

TOP EMOTIONS: {top 3 with counts}
PERCEPTION: {top 2 with counts}
{Quality warnings from 5k, each prefixed "(!) "}

Full report: {REPORT_PATH}
------------------------------------------------------------
```

**Rules**: Risk lines say what to DO (Consider:) or explicitly accept (Accept: when share is low and fix hurts other segments). Strength lines predict concrete behavior, not vague praise. No category averages table in terminal output. No filler.

### 6d. Transition

After the summary, print:

```
What next? You can:
  - Ask about any score ("Why is hook weak for passive tier?")
  - Paste a revised draft for comparison
  - Add real metrics ("This post got 45K impressions, 89 reactions...")
  - Type "show history" for past evaluations
```

Wait for user input. Handle per Step 7.

---

## Step 7: Conversational Follow-up

After the evaluation is complete and the summary is displayed, you enter conversational mode. The author can ask follow-up questions or perform actions. Handle them based on intent:

| Author intent | Detection pattern | What to do |
|---------------|-------------------|------------|
| **Discussion** | "Why is hook weak?", "Which personas liked it?", "Explain score X" | Analyze AGENT_RESULTS data. Reference specific persona scores, context effects, and reasoning from the evaluation. Quote the reasoning field where relevant. |
| **Revised draft** | "Here's a revised version: ...", new text pasted, new file path given | Parse the new input using Step 1 rules. Re-run the full evaluation (Steps 2-6) with the new text. Then show a comparison: score deltas for REACH_SCORE/BUYER_SCORE, per-tier changes, top 3 improved personas, top 3 declined personas. Label the runs "v1" and "v2". |
| **Add calibration data** | "This post got 45K impressions, 89 reactions...", numbers with engagement terms | Handle per **7a. Calibration Input** below. |
| **Check accuracy** | "How accurate is the panel?", "correlation", "panel accuracy" | Handle per **7b. Calibration Accuracy Report** below. |
| **Persona management** | "Add a persona: ...", "Remove persona ...", "Change persona ..." | Edit `panel/data/personas.json` — add/modify/delete persona. Recalculate audience_shares so they sum to 1.00. Warn: "Editing personas invalidates existing calibration data." Confirm the change. |
| **Context management** | "Add context: ...", "Remove context: ...", "Change context ..." | Edit `panel/data/contexts.json` — add/modify/delete context. Validate required fields present. Confirm the change. |
| **History** | "Show history", "previous evaluations", "past scores" | Handle per **7f. Evaluation History** below. |
| **Anything else** | No pattern match above | Respond naturally using the evaluation data and your knowledge of the evaluation domain. If the intent is ambiguous, ask which action the user wants. |

### 7a. Calibration Input

<!-- T012: Parse real performance metrics, match/create calibration record, compute weighted_score, write calibration.json.
     Formulas: weighted_score = reactions*1 + comments*3 + saves*2 + reposts*5 (null fields excluded, not 0).
     engagement_rate = sum(non-null action fields) / impressions (null if impressions is null or no action fields). -->

When the author provides real performance metrics, follow this procedure:

#### 7a-1. Extract metrics from natural language

Scan for **numbers adjacent to metric keywords**. Handle any format: `"45K imp, 89 reactions"`, `"impressions: 45000"`, `"this got 12.4k views and 56 likes"`, pipe/comma/semicolon separators, number before or after keyword.

**Numbers**: `45000`, `45K`=45000, `12.4k`=12400, `1.2M`=1200000, `45,000`=45000, `0.8%`=0.008 (rates only).

**Keywords** (case-insensitive, match synonyms):

| Field | Accepted keywords |
|-------|-------------------|
| `impressions` | impressions, imp, imps, views, view, reach |
| `reactions` | reactions, react, reacts, likes, like |
| `comments` | comments, comm, comms, replies, reply |
| `saves` | saves, save, saved, bookmarks, bookmark |
| `reposts` | reposts, repost, shares, share, reshares |
| `engagement_rate` | engagement rate, er, eng rate (only with decimal/percentage numbers) |
| `follower_growth` | follower growth, followers, new followers |

**Rules**: Tokenize on commas/pipes/semicolons/newlines. Each segment needs both a number and a keyword; skip segments missing either. Later occurrence of same keyword wins. **Missing fields = null, not 0.** Explicit `"0 saves"` stores 0 (contributes zero to weighted_score); omitted saves = null (excluded from formula). If only impressions given with no action metrics: `weighted_score = null`, show note.

#### 7a-2. Auto-link post title

Apply in order, stop at first match:
1. **Explicit title in input**: non-metric text like `"Developer to Owner got 63K..."` or a quoted string `"'Boring AI Stack' got..."`.
2. **Current session draft exists**: if a draft was evaluated this session, auto-link to it. Don't ask "which post?" — the connection is obvious. Includes `"this post"`, `"the draft"`, or no title mention at all.
3. **No session draft**: list 5 most recent posts from calibration.json, ask user to pick.

#### 7a-3. Match and update calibration record

Read `panel/data/calibration.json`. Match by: (1) exact title (case-insensitive), (2) substring match. If multiple substring matches, list them and ask. If no match, create a new record.

**Updating**: overwrite only fields the user provided; keep existing values for unmentioned fields (allows incremental updates). Recompute `weighted_score` and `engagement_rate` from the merged state using the formulas above.

**New records**: initialize all fields as null, populate extracted values. For `post_date`, parse from input if present (any format), otherwise ask (accept "skip" for null).

**Synthetic score backfill**: if this post was evaluated in the current session, store `synthetic_scores: { reach_score: REACH_SCORE, buyer_score: BUYER_SCORE, evaluated_at: timestamp }`. This is what makes calibration work — pairing real metrics with synthetic predictions. If no evaluation this session, leave synthetic_scores unchanged.

Write to `panel/data/calibration.json` (pretty-printed, 2-space indent, trailing newline).

#### 7a-4. Confirmation mini-report

Compute: `N_REAL` = posts with non-null weighted_score, `N_TOTAL` = total records, `N_BOTH` = posts with both weighted_score and reach_score.

If N_BOTH >= 2: for each dual-scored post, `normalized_real = round(100 * weighted_score / max_weighted_score)`, `delta = abs(reach_score - normalized_real)`. Find best/worst predicted.

```
--- Calibration Updated ------------------------------------
  Post: {title} ({date or "no date"}) — {Updated|New record}
  Imp: {v|—}  React: {v|—}  Comm: {v|—}  Saves: {v|—}  Reposts: {v|—}
  Weighted: {v|—}  ER: {v%|—}  Followers: {v|—}
  {if synthetic} Synthetic: Reach {r}/100, Buyer {b}/100
  Calibration: {N_REAL}/{N_TOTAL} real, {N_BOTH} with both scores
  {if N_BOTH >= 2} Best predicted: "{title}" (off by {d} pts)
                   Worst predicted: "{title}" (off by {d} pts)
  {if N_BOTH < 4} Need {4-N_BOTH} more dual-scored post(s) for correlation.
  {if N_BOTH >= 4} Full correlation available — ask "How accurate is the panel?"
------------------------------------------------------------
```

Then return to conversational mode (Step 7 loop).

### 7b. Calibration Accuracy Report

<!-- T013: Compute and display Pearson correlation between synthetic reach_score and real weighted_score.
     Triggered by "how accurate is the panel?", "correlation", "panel accuracy", etc. -->

When the author asks about panel accuracy:

#### 7b-1. Load and filter calibration data

Read `panel/data/calibration.json`. Build paired records: keep only records where BOTH `real_metrics.weighted_score` AND `synthetic_scores.reach_score` are not null.

#### 7b-2. Check minimum sample size

If fewer than 4 paired records exist, respond:

```
Need {4-N} more calibrated post(s) for reliable correlation.
Currently have {N} post(s) with both real and synthetic scores.
Add real metrics to evaluated posts to build calibration data.
```

Return to conversational mode. Do not attempt correlation.

#### 7b-3. Compute Pearson correlation

If 4+ paired records, compute Pearson correlation coefficient using Bash:

```bash
python3 -c "import json,sys; d=json.load(open('panel/data/calibration.json')); pairs=[(r['real_metrics']['weighted_score'],r['synthetic_scores']['reach_score']) for r in d if r['real_metrics'].get('weighted_score') and r['synthetic_scores'].get('reach_score')]; n=len(pairs); mx=sum(x for x,_ in pairs)/n; my=sum(y for _,y in pairs)/n; cov=sum((x-mx)*(y-my) for x,y in pairs); sx=(sum((x-mx)**2 for x,_ in pairs))**0.5; sy=(sum((y-my)**2 for _,y in pairs))**0.5; print(round(cov/(sx*sy),3) if sx*sy else 'N/A')"
```

Store result as `CORRELATION`.

#### 7b-4. Interpret and display

Interpret the correlation:
- `>= 0.7` — **Strong**: panel predictions track real performance well
- `0.5 – 0.69` — **Moderate**: directionally useful, not precise
- `< 0.5` — **Weak**: panel needs recalibration or more data

Build a per-post comparison table sorted by real weighted_score descending. For each paired post: title, real weighted_score, synthetic reach_score, delta (synthetic minus real). Add a rank column for each score and note rank-order mismatches.

```
--- Calibration Accuracy Report ----------------------------
  Pearson r = {CORRELATION}  ({interpretation})
  Data points: {N} posts with both real and synthetic scores

  Post                     Real   Synth  Delta  Real#  Synth#
  ─────────────────────────────────────────────────────────────
  {title (25ch)}           {ws}   {rs}   {d}    {rk}   {sk}
  ...

  Rank agreement: {count}/{N} posts ranked in same order
------------------------------------------------------------
```

Return to conversational mode (Step 7 loop).

### 7c. Persona Management

<!-- T015: CRUD operations on personas.json with validation.
     Triggered by "add persona", "edit persona", "delete persona", "remove persona",
     "show personas", "list personas", "all personas", etc. -->

When the user asks to **show/list all personas**:

Read `panel/data/personas.json`. Display a summary table sorted by `audience_share` descending:

```
Name                          Share   Tier              Behavior
──────────────────────────────────────────────────────────────────
{name (30ch)}                 {share} {tier_label}      {evaluation_behavior.type}
...
Total share: {sum}
```

Return to conversational mode.

When the user asks to **add, edit, or delete a persona**:

1. Read `panel/data/personas.json` into memory.
2. Apply the requested change:
   - **Add**: construct a full persona object with all required fields (`id`, `name`, `audience_share`, `tier`, `tier_label`, `demographics` with `age_range`/`location`/`education`/`occupation`/`experience_years`/`family`, `psychographics` with `big_five` (all five traits)/`values`/`tech_adoption`/`decision_style`, `engagement_trigger`, `evaluation_behavior` with `type`/`pattern`/`scroll_speed`/`comment_style`). Ask the user for any fields they didn't specify.
   - **Edit**: locate persona by `id` or `name` (fuzzy match OK), update specified fields.
   - **Delete**: locate persona by `id` or `name`, remove from array.
3. **Validate** the resulting array:
   - All Big Five values in 0–1 range.
   - All required fields present and non-null (except `demographics.family` which may be null).
   - Sum of `audience_share` across all personas = 1.00 ± 0.01. If outside this range, warn and suggest which personas' shares to adjust to restore balance.
   - `tier` is one of: `core_engager`, `strategic_reader`, `passive`, `edge`.
   - `evaluation_behavior.type` is one of: `commenter`, `liker`, `lurker`, `sharer`.
4. Print warning: **"Modifying personas invalidates existing calibration. Recalibration recommended after changes."**
5. Write the validated array back to `panel/data/personas.json` (pretty-printed, 2-space indent).
6. Confirm: state what changed, show new persona count and audience_share sum.

Return to conversational mode (Step 7 loop).

### 7d. Context Management

<!-- T016: CRUD for panel/data/contexts.json. Triggered by "add/edit/delete context", "show all contexts", etc. -->

When the user asks to add, edit, or delete a context — or to show all contexts:

1. Read `panel/data/contexts.json`.
2. **Show all contexts** → display a summary table: name, attention_level, scroll_speed, engagement_threshold. Return to Step 7 loop.
3. **Add/edit/delete** → apply the requested change. Every context must have: `id`, `name`, `description`, `modifiers` (`attention_level` 0–1, `emotional_state` string, `scroll_speed` slow/medium/fast, `engagement_threshold` low/medium/high).
4. **Validate before writing:**
   - All required fields present, enums valid, `attention_level` between 0 and 1.
   - Internal consistency check: low attention (≤0.3) should pair with fast scroll and high threshold; high attention (≥0.7) should pair with slow scroll and low threshold. If the combination contradicts, warn the user and ask to confirm before saving.
5. Write the updated array back to `panel/data/contexts.json`.
6. Confirm the change with a one-line summary (e.g., "Added context 'Monday morning commute'.").

Return to conversational mode (Step 7 loop).

### 7e. Draft Comparison

<!-- T017: When user submits a revised draft, re-evaluate and show score deltas vs. the previous run. -->

When the "Revised draft" intent fires:

1. **Snapshot previous scores**: store `PREV_REACH = REACH_SCORE`, `PREV_BUYER = BUYER_SCORE`, `PREV_PERSONA_AVERAGES = PERSONA_AVERAGES` (deep copy), and `PREV_TIER_COMPOSITES` from the current evaluation.

2. **Re-evaluate**: parse the new input via Step 1 rules, then run the full pipeline (Steps 2-6) against the new text. All variables (REACH_SCORE, BUYER_SCORE, PERSONA_AVERAGES, AGENT_RESULTS, etc.) are overwritten with v2 values. The Step 6 report is saved as a new file (not overwriting v1).

3. **Compute deltas** after the new evaluation completes:
   - `DELTA_REACH = REACH_SCORE - PREV_REACH`
   - `DELTA_BUYER = BUYER_SCORE - PREV_BUYER`
   - Per-tier delta: for each tier, `new_avg_reach_composite - prev_avg_reach_composite`
   - Per-persona delta: for each persona, `new_reach_composite - prev_reach_composite`
   - `TOP_IMPROVED` = 3 personas with largest positive delta, sorted desc
   - `TOP_DECLINED` = 3 personas with largest negative delta (skip if no declines), sorted asc

4. **Display comparison** immediately after the v2 terminal summary (Step 6c):

```
--- Comparison: v1 → v2 ------------------------------------
Reach: {PREV_REACH} → {REACH_SCORE} ({+/-DELTA_REACH})
Buyer: {PREV_BUYER} → {BUYER_SCORE} ({+/-DELTA_BUYER})

Tiers:
  Core Engagers:     {prev} → {new} ({+/-delta})
  Strategic Readers: {prev} → {new} ({+/-delta})
  Passive:           {prev} → {new} ({+/-delta})
  Edge:              {prev} → {new} ({+/-delta})

Improved: {persona1} {+delta}, {persona2} {+delta}, {persona3} {+delta}
Declined: {persona1} {delta}, {persona2} {delta}
------------------------------------------------------------
```

5. **Save comparison report**: append a `## Comparison: v1 → v2` section to the v2 evaluation file (the one Step 6 just wrote). Include the Reach/Buyer deltas, tier deltas, and full per-persona delta table sorted by delta desc. Reference v1 report path.

Increment version labels for subsequent revisions (v2→v3, v3→v4, etc.). PREV_* snapshot always holds the immediately preceding evaluation. Return to conversational mode (Step 7 loop).

### 7f. Evaluation History

<!-- T018: Chronological table of past evaluations from saved files. -->

When the author asks "show history", "past evaluations", "previous scores", or similar:

1. Use Glob to find all `panel/evaluations/*.md` files. If none found, respond: "No saved evaluations yet." and return.
2. For each file, Read the frontmatter (~first 10 lines) and extract: `date`, `draft` (truncate to 50 chars), `reach_score`, `buyer_score`, `calibration_status`.
3. Sort by date ascending and display:

```
| Date | Draft | Reach | Buyer | Calibration |
|------|-------|-------|-------|-------------|
| 2026-03-24 14:30 | Developer to Own... | 72 | 81 | r=0.72 strong |
| 2026-03-24 15:15 | AI Stack boring... | 65 | 58 | r=0.72 strong |
```

4. If the author asks to view a specific past evaluation, Read the full file and display it.

Return to conversational mode (Step 7 loop).

---

## Presets (embedded for one-file installation)

When the init wizard selects a preset, generate data files from these templates.

### Preset 1: Social Media Content

**config.json**: domain: "Social Media Content", content_type: "Social media post draft", framing: "You are a social media feed simulator. You simulate real humans scrolling their feed — distracted, skeptical, self-interested. Default is scroll past.", action_weights: {like_probability: 1, comment_probability: 5, repost_probability: 10, save_probability: 2}, attention_metrics: ["hook_stop", "read_through"], hook_gate_threshold: 4, scores: "Reach Score" / "Buyer Score" (buyer_tiers: ["core_engager", "strategic_reader"]).

**personas**: Generate 10-20 social media audience personas (Senior Dev, Tech Lead, CTO, Recruiter, Indie Hacker, Junior Dev, Mid-Level Dev, PM, Freelance Dev, Content Creator, HR Manager, Enterprise Architect). Each with audience_share (sum to 1.0), tier, demographics, psychographics (Big Five), engagement_trigger, behavior pattern.

**contexts**: Generate 10-20 situational contexts (Monday morning, Friday evening, after AI release, layoffs wave, conference week, job hunting, burnout, quiet week, new year planning, feed oversaturated, post-viral fatigue, after AI scandal).

**metrics**: Generate 15-20 social media engagement metrics with anchors at 0/5/10 (hook_stop, read_through, attention_loss_point, primary_emotion, emotion_strength, ego_engagement, like/comment/repost/save/profile_click/follow probability, simulated_comment, comment_type, author_perception, screenshot_send, remember_tomorrow, purchase_intent).

### Preset 2: Code Review

**config.json**: domain: "Code Review", content_type: "Pull request or code change", framing: "You are a code reviewer examining a pull request. Experienced, opinionated, time-constrained. Default is 'looks fine, approve' unless something catches your eye.", action_weights: {approve_probability: 1, comment_probability: 5, request_changes_probability: 8, bookmark_probability: 2}, attention_metrics: ["initial_interest", "read_depth"], scores: "Quality Score" / "Risk Score" (buyer_tiers: ["senior_reviewer", "architect"]).

**personas**: Generate 6-8: Senior Backend Dev, Junior Dev, Tech Lead, Security Engineer, DevOps, QA, Architect, PM.

**contexts**: Generate 6-8: before_release, tech_debt_sprint, new_team_member, deadline_pressure, post_incident, code_freeze, refactoring_week, demo_day.

**metrics**: Generate 10-12: initial_interest, read_depth, readability, bug_risk, maintainability, test_coverage_need, architectural_fit, security_concerns, performance_impact, approve_probability, comment_text, request_changes_probability.

### Preset 3: Product Feature Feedback

**config.json**: domain: "Product Feature Feedback", content_type: "Feature proposal or product brief", framing: "You are a stakeholder evaluating a feature proposal. Pragmatic, budget-aware, skeptical. Default is 'interesting, but not now.'", action_weights: {support_probability: 1, champion_probability: 5, escalate_probability: 8, bookmark_probability: 2}, attention_metrics: ["interest_level", "read_depth"], scores: "Approval Score" / "Priority Score" (buyer_tiers: ["executive", "decision_maker"]).

**personas**: Generate 5-6: CTO, Product Manager, End User (enterprise), Sales Rep, Support Engineer, Designer.

**contexts**: Generate 5-6: budget_freeze, competitor_launch, quarterly_planning, post_incident, growth_phase, cost_cutting.

**metrics**: Generate 8-10: interest_level, read_depth, business_value, feasibility, user_impact, strategic_alignment, implementation_risk, support_probability, concerns_text, enthusiasm_level.

### Preset 4: Ad Copy

**config.json**: domain: "Ad Copy", content_type: "Ad, email, or landing page copy", framing: "You are a potential customer seeing this ad for the first time. Busy, skeptical, ad-blind. Default is ignore.", action_weights: {click_intent: 1, share_intent: 5, purchase_consideration: 8, bookmark_intent: 2}, attention_metrics: ["attention_grab", "read_through"], scores: "Impact Score" / "Conversion Score" (buyer_tiers: ["ideal_customer", "high_intent"]).

**personas**: Generate 6-8 customer segments based on domain (let user describe their product during init, then generate appropriate segments).

**contexts**: Generate 5-6: first_impression, retargeting, competitor_comparison, seasonal_urgency, brand_awareness, post_purchase_upsell.

**metrics**: Generate 8-10: attention_grab, read_through, message_clarity, emotional_resonance, click_intent, trust_signal, brand_fit, recall_potential, objections_text, purchase_consideration.
