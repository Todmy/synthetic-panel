# Feature Specification: Generalize /panel Skill

**Feature Branch**: `003-generalize-panel`
**Created**: 2026-03-25
**Status**: Draft
**Input**: Generalize /panel into domain-agnostic synthetic evaluation panel with self-configuring init, presets, and one-file installation.
**Prior Art**: Branch `001-synthetic-audience-panel` — working LinkedIn panel with Spearman 0.951 calibration. `specs/002-generalize-panel/competitive-research.md` — 20+ competitors reviewed.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - First Run Init Wizard (Priority: P1)

A new user copies `panel.md` into their project's `.claude/commands/` and runs `/panel`. The skill detects no `panel/data/` directory exists and enters an interactive init wizard. The wizard asks what they're evaluating, who evaluates it, and what situations affect reactions. It then generates starter personas, contexts, metrics, and config — ready for immediate use.

**Why this priority**: Without init, the tool requires manual JSON creation which kills adoption. Init is the door.

**Independent Test**: Delete `panel/data/`, run `/panel`, answer the wizard prompts. Verify `panel/data/config.json`, `personas.json`, `contexts.json`, `metrics.json` are created with coherent content matching the user's answers.

**Acceptance Scenarios**:

1. **Given** `panel/data/` does not exist, **When** user runs `/panel`, **Then** the skill prints "No panel data found. Let's set up your evaluation panel." and starts the wizard.
2. **Given** the wizard is active, **When** user describes their domain (e.g., "product feature proposals for my SaaS team"), **Then** the skill generates 5-8 personas matching that domain (e.g., CTO, Product Manager, End User, Sales Rep, Support Engineer).
3. **Given** the wizard is active, **When** user describes situations ("budget freeze, post-incident, competitor just shipped"), **Then** the skill generates 5-8 contexts with appropriate modifiers (attention_level, emotional_state).
4. **Given** init is complete, **When** user runs `/panel "my content"`, **Then** evaluation runs with the generated personas and contexts.

---

### User Story 2 - Preset System (Priority: P1)

The user can choose from built-in presets during init instead of describing their domain from scratch. Presets provide complete starter configs for common use cases. The user can also start from a preset and customize.

**Why this priority**: Presets dramatically reduce time-to-first-evaluation and show the tool's versatility.

**Independent Test**: Run `/panel` init, choose "LinkedIn Content" preset. Verify it creates the same 20 personas, 20 contexts, 20 metrics as the current LinkedIn panel. Then re-init with "Code Review" preset and verify different personas/contexts/metrics are generated.

**Acceptance Scenarios**:

1. **Given** init wizard is active, **When** user is asked "What are you evaluating?", **Then** the wizard offers: preset choices (LinkedIn Content, Code Review, Product Feedback, Ad Copy) + "Custom — I'll describe my use case."
2. **Given** user selects "Code Review" preset, **Then** the skill creates personas (Senior Dev, Junior Dev, Tech Lead, Security Engineer, DevOps), contexts (before release, tech debt sprint, new team member, deadline pressure, code freeze), and metrics (readability, bug_risk, maintainability, test_coverage_need, architectural_fit, security_concerns).
3. **Given** user selects a preset then says "add a persona: QA Engineer", **Then** the persona is added to the generated set.

---

### User Story 3 - Domain-Agnostic Evaluation (Priority: P1)

The skill runs evaluations for ANY domain — not just LinkedIn posts. The agent prompt, aggregation formula, and report format adapt based on `config.json` settings. The core matrix engine (N personas × M contexts × K metrics → parallel agents → aggregate → report) is unchanged.

**Why this priority**: This is the core refactoring. Without it, the tool is LinkedIn-only.

**Independent Test**: Set up a "Code Review" panel, run `/panel` with a PR description. Verify the scorecard shows code-relevant metrics (readability, bug_risk) instead of LinkedIn metrics (hook_stop, ego_engagement).

**Acceptance Scenarios**:

1. **Given** a Code Review config, **When** the user submits a PR description, **Then** evaluators simulate code reviewers (not LinkedIn readers), and the scorecard shows code-quality metrics.
2. **Given** a Product Feedback config, **When** the user submits a feature proposal, **Then** evaluators simulate stakeholders (CTO, PM, End User), and the scorecard shows approval_likelihood, implementation_concerns, business_value.
3. **Given** any config, **When** evaluation completes, **Then** the Reach Score and Buyer Score are computed using action weights from `config.json`, not hardcoded LinkedIn weights.

---

### User Story 4 - Reconfiguration (Priority: P2)

The user can re-run setup to change their panel configuration, add/remove presets, or switch domains entirely. The skill can also self-configure incrementally — add a persona, change a metric, swap a preset.

**Why this priority**: Users need to iterate on their panel setup. First config is rarely perfect.

**Independent Test**: Start with LinkedIn preset, run `/panel setup` to switch to Product Feedback. Verify all data files are regenerated.

**Acceptance Scenarios**:

1. **Given** a configured panel, **When** user says "switch to code review preset", **Then** the skill regenerates all data files for the new domain, warns about losing current calibration data.
2. **Given** a configured panel, **When** user says "add a metric: technical_debt_risk", **Then** the skill adds it to metrics.json with appropriate type, anchors, and range.

---

### User Story 5 - One-File Installation (Priority: P2)

A new user can install the panel with a single command. No dependencies, no CLI tools, no package managers. Just copy one file and run.

**Why this priority**: Low friction = adoption. Every extra step loses users.

**Independent Test**: In a fresh project with Claude Code, run the curl install command. Verify `/panel` appears in skill list and init wizard works.

**Acceptance Scenarios**:

1. **Given** a project with Claude Code and no panel installed, **When** user runs `mkdir -p .claude/commands && curl -o .claude/commands/panel.md <URL>`, **Then** `/panel` becomes available as a slash command.
2. **Given** the installed skill, **When** user runs `/panel`, **Then** init wizard starts (no additional setup required).

---

### Edge Cases

- What happens if user runs `/panel "content"` before init? Skill detects missing data and enters init wizard instead of crashing.
- What happens if `config.json` exists but `personas.json` doesn't? Skill reports which files are missing and offers to regenerate.
- What happens if user switches presets after calibration? Warn that calibration data is invalidated, offer to keep or clear it.
- What happens if the user's domain doesn't match any preset? Custom init flow handles any domain via conversational persona/context/metric generation.
- What happens if user wants to mix presets (e.g., LinkedIn metrics + Code Review personas)? Allow it — presets are starting points, not constraints.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: The skill MUST be a single file (`.claude/commands/panel.md`) that self-configures on first run. No external installer, CLI, or dependencies.
- **FR-002**: On first run (when `panel/data/` does not exist), the skill MUST enter an interactive init wizard that creates all required data files.
- **FR-003**: The init wizard MUST offer preset choices: LinkedIn Content, Code Review, Product Feature Feedback, Ad Copy, and Custom.
- **FR-004**: Each preset MUST generate: `config.json`, `personas.json`, `contexts.json`, `metrics.json`. Persona count: 5-20. Context count: 5-20. Metric count: 8-20.
- **FR-005**: `config.json` MUST contain: `domain` (name), `content_type` (what's being evaluated), `framing` (system prompt for agents), `action_weights` (metric-to-impact mapping for scoring formula), `score_names` (what to call the two headline scores), `score_interpretation` (range descriptions).
- **FR-006**: The agent prompt template MUST read framing, metric definitions, and persona structure from config/data files — no hardcoded domain-specific text in the template.
- **FR-007**: The aggregation formula MUST read `action_weights` from `config.json` to compute scores. The formula structure (action_value × hook_gate × log_scaling) stays the same; the weights and metric names are configurable.
- **FR-008**: The skill MUST support `/panel setup` to re-run init wizard or reconfigure the panel.
- **FR-009**: All features from the original LinkedIn panel MUST continue working: parallel agents, calibration, comparison, history, persona/context CRUD, conversational follow-up.
- **FR-010**: The skill MUST detect which mode to enter based on arguments: no args + no data = init; no args + data exists = show help; content arg = evaluate.
- **FR-011**: Presets MUST be embedded in the skill file itself (not external files) so that one-file installation works.

### Key Entities

- **Config**: Domain configuration. Contains: domain name, content type description, agent framing text, action weights (metric→multiplier map), score names (primary + secondary), score interpretation ranges. Stored in `panel/data/config.json`.
- **Preset**: A built-in starter config embedded in the skill. Contains complete config + starter personas + contexts + metrics for a specific domain. User selects during init. 4 presets built-in.
- **Persona**: (unchanged from 001) Synthetic evaluator with demographic, psychographic, and behavioral attributes. Schema same but field names are domain-agnostic (e.g., `evaluation_behavior` instead of `linkedin_behavior`).
- **Context**: (unchanged) Situational modifier with attention_level, emotional_state, and domain-appropriate modifiers.
- **Metric**: (unchanged) Evaluation dimension with type (score/text/label), range, and anchors. Metric IDs are domain-specific (e.g., `readability` for code, `hook_stop` for LinkedIn).

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: User can go from zero to first evaluation in under 5 minutes (install + init + evaluate).
- **SC-002**: LinkedIn preset produces identical results to the current panel (same personas, contexts, metrics, same Spearman correlation on calibration dataset).
- **SC-003**: Code Review preset produces coherent evaluations when given a PR description — metrics are code-relevant, personas behave as code reviewers.
- **SC-004**: Switching between presets works without manual file editing.
- **SC-005**: The skill file is self-contained — one `curl` command installs a fully functional panel.
- **SC-006**: All 7 conversational follow-up features from the original panel work in any domain (discuss, revise, calibrate, accuracy check, persona mgmt, context mgmt, history).
