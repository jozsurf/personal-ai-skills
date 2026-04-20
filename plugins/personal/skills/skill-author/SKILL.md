---
name: skill-author
description: >-
  Guide for writing high-quality skill documents and agent definitions consumed by AI agents.
  Use when creating a new skill, reviewing an existing skill for quality, restructuring a skill
  that has grown too large, or deciding whether a problem warrants a skill vs inline instructions.
  Covers front-matter schema, structural patterns, prose style, fast-path design, reference-file
  splitting, and the relationship between skills and agent definitions.
---

# Skill Author

A guide for designing, writing, and maintaining skill documents and agent definitions for AI agents.

## What Is a Skill?

A skill is a **self-contained instruction document** that an AI agent loads on demand to gain focused expertise. Skills are not general-purpose prompts — each skill encodes one domain of knowledge or one repeatable workflow.

**Skill** vs **Agent definition**:

| Concept | File pattern | Purpose |
|---------|-------------|---------|
| Skill | `SKILL.md` inside a named folder | Reusable domain knowledge or workflow, loaded by name |
| Agent | `*.agent.md` | A complete agent persona that composes skills, defines tools, and drives a top-level task |

## When to Write a Skill

Write a skill when:
- The same knowledge or workflow recurs across multiple tasks or agents
- The instructions are too long or specialised to embed inline in every agent that needs them
- You want to test and iterate the instructions in isolation from the agent that uses them
- The content is repo-agnostic and could be reused across different codebases

Do **not** write a skill when:
- The instructions are truly one-off — embed them directly in the agent
- The content is just a cheat sheet or reference table with no procedural logic — use a plain reference file instead
- The skill would simply forward to another skill without adding value

## File Layout

```
.agents/skills/<skill-name>/
    SKILL.md               # Required — the skill itself
    references/            # Optional — supplementary files loaded selectively
        <topic>.md
        <repo>/            # Repo-specific variants of a reference
            <topic>.md
```

Rules:
- The folder name must match the `name` field in front matter, using kebab-case.
- `SKILL.md` is the only required file. Add a `references/` subdirectory only when the skill would otherwise become too large to scan, or when content differs by repo.
- Never put repo-specific details (file paths, branch names, product constants) directly in `SKILL.md`. Move them to `references/<repo>/<topic>.md` and have the skill load them conditionally.

## Front Matter Schema

Every `SKILL.md` must begin with a YAML front matter block:

```yaml
---
name: <kebab-case-name>
description: >-
  <One to three sentences. State the domain or workflow, list explicit trigger
  conditions with "Use when..." or "Apply when...", and call out any important
  boundaries. Agents use this description to decide whether to load the skill.>
---
```

**Optional front-matter fields** (used in agent definitions, not plain skills):

```yaml
argument-hint: <prompt shown to user when invoking the agent>
model: <model-id>            # e.g. claude-opus-4.6
tools: [tool1, tool2, ...]   # allowlist of tools the agent may use
user-invocable: false        # hide from user-facing skill list
```

### Writing the Description

The description is the **sole mechanism** the agent runtime uses to decide whether to load a skill. When the agent receives a task, it reads every available skill's description and decides which (if any) to invoke. The skill body is never read before the description fires.

This means a poorly written description causes one of two failures:

| Failure mode | Cause | Effect |
|---|---|---|
| **Under-triggering** | Description is too vague or too narrow | Skill never loads; agent works without the knowledge it needs |
| **Over-triggering** | Description is too broad | Skill loads on unrelated tasks; wastes tokens, may confuse the agent |

Both are real problems. Optimize the description to fire precisely.

#### Positive triggers

State the concrete scenarios where the skill MUST load. Use "Use when…" or "Apply when…" followed by specific, unambiguous conditions:

- Name the specific entities, tools, or workflows involved (e.g. `ediprod/get-issue-details`, `ISS-type work items`, `CusCodeData`, `ColumnLayoutBuilder`)
- Use action verbs that match what users actually ask (e.g. "adding", "debugging", "refactoring", "tracing")
- List multiple triggers if the skill covers related scenarios — but keep each trigger specific

#### Negative boundaries

State the conditions where the skill must NOT load, when there's a plausible overlap with another skill or a common misuse:

```
# Good — no boundary needed (name is unambiguous)
"Use when adding a new boolean registry item to FreightConfigurationRegistry."

# Good — boundary prevents confusion with a related skill
"Use when implementing CusCodeData types (CY_ tables). For CusSupportingInfo
types (CSI_ tables), use the cus-supporting-info skill instead."
```

Only add a boundary when the overlap is real and likely to cause mis-loading. Do not add hypothetical boundaries.

#### Two-sided description test

Before finalising a description, apply this test:

1. **Would a passing task fire it?** — Take the canonical input for this skill. Does the description match? If not, it under-triggers.
2. **Would an adjacent task also fire it?** — Take a task meant for a related skill. Would this description also match? If yes, it over-triggers. Add a boundary or tighten the trigger.

#### Description checklist

- [ ] Names the domain or workflow in the first sentence
- [ ] Contains at least one specific "Use when…" trigger (not vague — names entities/tools/actions)
- [ ] Includes a negative boundary if there is a plausible overlap with another skill
- [ ] Is no longer than ~4 sentences — longer content belongs in the skill body
- [ ] Does **not** repeat the skill name verbatim as the first word

Example — **weak** (under-triggers and over-triggers simultaneously):
> "Skill for investigating issues."

Example — **strong** (precise positive trigger + negative boundary):
> "Systematic root cause analysis for ISS-type work items. Use when triaging an ISS work item and you have the related issue occurrence details from ediprod/get-issue-details. Handles exception message provenance, deduplication, branching model context, threading/lifecycle analysis, and runtime awareness. Do not use for non-ISS work items — for general code investigation use the customs-investigation skill."

#### Verifying triggering with evalcases

The description cannot be considered correct until evalcases confirm both sides:

```yaml
# Positive — skill must fire
- id: triggers-on-iss-triage
  assertions:
    - type: skill-trigger
      skill: iss-investigation
      should_trigger: true

# Negative — skill must NOT fire on unrelated input
- id: does-not-trigger-on-gui-task
  input: "Add a new ZTextBox to the declaration header panel"
  assertions:
    - type: skill-trigger
      skill: iss-investigation
      should_trigger: false
```

A skill with only `should_trigger: true` tests is half-verified. Add at least one `should_trigger: false` test whenever the description has a negative boundary.

## Structural Patterns

### Mandatory H1

The first line after front matter must be an H1 heading that names the skill in human-readable form.

```markdown
# My Skill Name
```

### Standard Section Order

Not every skill needs every section. Use this order when sections are present:

1. **What Is This / Goal** — one-paragraph orientation (optional if the name is self-evident)
2. **When to Use / When NOT to Use** — explicit trigger and anti-trigger lists
3. **Inputs** — what the skill expects to receive (data, tool results, context)
4. **Workflow / Steps** — the procedural core (see Workflow Patterns below)
5. **Output** — what the skill produces and in what format
6. **Important Rules / Constraints** — non-negotiable behaviors, safety guardrails
7. **Maintenance** — authoring conventions, where repo-specific content belongs

### Workflow Patterns

#### Linear numbered steps

Use for workflows where every step must be run in order:

```markdown
## Steps

### 1. Locate the Fix Site
...

### 2. Read Surrounding Code
...
```

#### Lettered sub-steps (fast paths and checks)

Use suffixes `a`, `b`, `c` for optional checks or fast paths within a numbered step. The suffix implies "this is a variant or adjunct to the parent step, not a new top-level step":

```markdown
### 1. Validate Input
...

### 1a. Fast Path: Known Pattern
If the input matches a known pattern, skip steps 2–4 and jump to step 5.

### 1b. Pre-Search Knowledge Base
Before deep analysis, search for prior incidents...
```

#### Phase-based workflows

Use phases when a workflow has natural gates where work may stop or branch:

```markdown
## Phase 0: Pre-flight
Validate environment variables. Stop if any are missing.

## Phase 1: Retrieval
Gather all inputs before beginning analysis.

## Phase 2: Investigation
...

## Phase 2.5: Deep Investigation
Perform only when Phase 2 depth assessment says deep investigation is warranted.
```

#### Checklist pattern

Use when the agent must work through a list of independent checks:

```markdown
## Investigation Checklist

Work through these checks sequentially. Each check either resolves the issue or narrows the hypothesis.

### 1. Exception Type/Message Consistency
...

### 2. Known Pattern Quick-Check
...
```

#### Decision table / lookup table

Use tables for mappings, tool references, or branching decisions:

```markdown
| Condition | Action |
|-----------|--------|
| Input has X | Skip to step 5 |
| Input has Y | Run full workflow |
```

## Skill Size and Context Cost

Every skill loaded by an agent consumes **context window tokens**. Larger skills cost more tokens, leave less room for actual work, and make the agent slower and more expensive to run. Treat token budget as a first-class constraint.

### Size targets

| Skill type | Target size | Hard limit |
|---|---|---|
| Focused procedural skill | ≤ 200 lines | 400 lines |
| Broad reference skill (tool guides, API docs) | ≤ 400 lines | 600 lines |
| Any single reference file | ≤ 150 lines | 250 lines |

If a SKILL.md exceeds its target, look for content to extract to `references/` or remove entirely.

### The preloading trap

A common mistake is writing skills that load all their content upfront — enumerating every known pattern, every edge case, every reference table — regardless of whether the current task needs any of it.

**Why this is harmful**:
- The agent reads everything, pays the token cost, then uses 10% of it
- The relevant signal is buried in noise, reducing instruction-following quality
- Large skill files accumulate stale content that nobody audits because "it might be needed someday"

**The correct model — load only what the task needs**:

```markdown
### 1a. Known Exception Pattern Quick-Check

Check `references/known-exception-patterns.md` ONLY if the exception type
matches one of these categories: FileNotFound, InvalidOperation, Deadlock.
Skip this step entirely for other exception types.
```

The skill **instructs** the agent on which reference to load and **when** — it does not load everything speculatively. The agent reads the reference only if that branch is reached.

### Conditional loading patterns

**Pattern 1 — load on match**:
```markdown
If the exception is a SQL error (deadlock, timeout, constraint violation),
read `references/sql-error-patterns.md`. Otherwise skip.
```

**Pattern 2 — load on repo**:
```markdown
For repo-specific branch naming conventions, read
`references/{repo}/branching-model.md` (e.g. `references/cargowise/branching-model.md`).
```

**Pattern 3 — load on depth**:
```markdown
If deep investigation is warranted (assessed in Step 4), read
`references/depth-examples.md` for annotated examples.
Skip for standard triage.
```

**Pattern 4 — named escape hatch**:
```markdown
For the full search strategy including cross-branch search and NuGet packages,
read `references/exception-provenance.md`. For most cases the inline steps below are sufficient.
```

### When to split into a reference file

Split content into `references/` when **all three** of these are true:
1. The content is more than ~30 lines
2. It is only needed in specific scenarios (not every invocation)
3. It is stable reference data, repo-specific detail, or deep examples — not core logic

Do **not** split content that the skill always needs. Fragmentation that requires loading multiple files on every invocation is no better than a large SKILL.md.

**Maintenance rule** — always state this at the top of the skill if it has references:
```markdown
## Maintenance: Repo-Specific References

This skill is repo-agnostic. All repo-specific details (file paths, constant names,
branch naming conventions, known patterns) belong in `references/{repo}/` files.
```

## Prose Style

- **Voice**: Imperative. "Retrieve the work item." Not "You should retrieve the work item."
- **Tense**: Present tense throughout.
- **Sentence length**: Short. One instruction per sentence.
- **Emphasis**: Use **bold** for key terms on first use, warnings, and non-negotiable rules. Do not bold entire sentences.
- **Code**: Wrap all code, commands, tool names, file paths, and identifiers in backticks.
- **Lists vs prose**: Use bullet lists for enumerations of 3+ items. Use prose for 1–2 items.
- **Tables**: Use for reference data with 2+ columns. Prefer tables over nested bullet lists.
- **Negation**: State what NOT to do when the wrong action is plausible and has real consequences. Example: "Do NOT use `git log` — it doesn't support path filtering."
- **Fast paths first**: When a shortcut can skip most of the workflow, state it early (before the full workflow), not at the end.

## Anti-Patterns to Avoid

| Anti-pattern | Problem | Fix |
|---|---|---|
| Repo-specific paths in SKILL.md | Breaks reuse across codebases | Move to `references/{repo}/` |
| Skill that just says "load another skill" | No added value | Inline the logic or compose in the agent |
| Steps that are always skipped | Dead weight | Remove or make conditional with an explicit trigger |
| Description too vague — under-triggers | Skill never loads; agent works blind | Add specific entities, tools, and action verbs to trigger phrase |
| Description too broad — over-triggers | Skill loads on unrelated tasks; wastes tokens | Add negative boundary; tighten trigger to specific conditions |
| No `should_trigger: false` evalcase when boundary exists | Over-triggering goes undetected | Add negative evalcase for each explicit "do NOT use when" condition |
| SKILL.md > 400 lines with no reference splitting | Burns context tokens; buries signal in noise | Extract conditionally-loaded content to `references/` |
| Loading all reference files at skill start | Token cost paid even when content never used | Use conditional loading patterns — load only when the branch is reached |
| Exhaustive "might be needed" content | Stale, unaudited, never used | Remove it; add it back when a real task needs it |
| Using passive voice ("the code should be read") | Ambiguous who acts | Use imperative ("Read the code") |
| Embedding agent-level orchestration in a skill | Confuses skill with agent | Agent files drive orchestration; skills provide knowledge |
| Repeating front-matter description in H1 | Redundant | H1 is a human-readable label, not a repeat of the description |

## Evalcases

Every skill must be backed by at least one evalcase file. Evalcases are the automated tests that verify the skill actually produces the outputs it promises — without them, skill regressions are invisible.

### File location and naming

```
WTG.AI.Prompts/evals/<domain>/<skill-name>.eval.yaml
```

Examples:
- `evals/development/code-research.eval.yaml`
- `evals/cargowise/cw-layout.eval.yaml`
- `evals/cargowise-customs/customs-investigation.eval.yaml`

The eval file lives in `WTG.AI.Prompts`, **not** alongside the skill. Match the domain folder to the product area the skill belongs to.

### Eval file structure

```yaml
description: <One sentence — what this eval file tests>
workspace: .templates/eval-workspace-setup.yaml   # optional — include if skill needs a repo context

tests:
  - id: <kebab-case-test-id>
    criteria: <One sentence — what a passing response demonstrates>
    input: "<user prompt that exercises the skill>"
    assertions:
      - <assertion>
```

To load the skill as a system-level input (rather than relying on the agent to trigger it):

```yaml
input:
  - role: system
    content:
      - type: file
        value: /plugins/<domain>/skills/<skill-name>/SKILL.md

tests:
  - id: ...
    input: "<user prompt>"
    assertions: ...
```

### Assertion types

| Type | When to use | Example |
|------|-------------|---------|
| Plain string | Conceptual checks — readable, flexible | `"References or loads the cw-layout skill"` |
| `type: contains` | Exact value must appear in output | `value: "ColumnLayoutBuilder<T, TBag>"` |
| `type: contains-all` | Multiple values all required | `value: ["CombineCategories", "ResString.GetMultilingualString"]` |
| `type: contains` + `negate: true` | Value must NOT appear | `value: "ResString"`, `negate: true` |
| `type: regex` | Pattern match (GUIDs, version strings) | `value: "[0-9a-fA-F]{8}-..."` |
| `type: skill-trigger` | Verifies skill was loaded at all | `skill: cw-layout`, `should_trigger: true` |
| `type: rubrics` | Weighted, multi-criteria judgment | See rubric pattern below |

**Rubric pattern** — use when output quality matters more than exact text:

```yaml
assertions:
  - type: rubrics
    criteria:
      - id: produces-structured-output
        outcome: Produces a summary with X, Y, Z sections
        weight: 2.0
        required: true
      - id: routes-correctly
        outcome: Routes the defect to the correct team
        weight: 1.5
        required: false
```

### What to test

Each eval file should cover:

1. **Happy path** — skill triggers on the canonical input and produces the expected output
2. **Key technical outputs** — specific class names, method signatures, or constants that must appear (use `type: contains`)
3. **Anti-patterns** — things the skill explicitly says NOT to do must not appear (use `negate: true`)
4. **Fast paths** — when a shortcut applies, verify the agent takes it and skips the full workflow
5. **Skill trigger** — at least one test verifies the skill is actually loaded (`type: skill-trigger`)
6. **Disambiguation** — if the description has explicit "do NOT use when" conditions, test that the skill does NOT trigger on those inputs (`should_trigger: false`)

### Minimum coverage rule

A skill is not considered complete until it has evalcases covering **at least**:
- One happy-path test
- One `type: contains` test for a non-trivial technical output
- One `type: skill-trigger` test

Skills that touch "do NOT" rules or anti-patterns should additionally have at least one negation test.

### Keeping evals honest

- **Evals should fail before the skill is written** — write the eval for the behavior you intend, then write the skill to make it pass.
- **Avoid tautological evals** — testing that the output "contains the word investigation" proves nothing. Test specific facts from the skill's knowledge.
- **Update evals when the skill changes** — if a step is removed or a class name changes, the eval must change with it. Stale evals that always pass are worse than no evals.

## Skill vs Agent Composition

Skills are **knowledge providers**. Agents are **orchestrators** that compose skills:

```
agent.md
  ├── Phase 1: Retrieval
  │     └── invokes: ediprod-mcp skill (for tool usage patterns)
  ├── Phase 2: Investigation
  │     └── invokes: iss-investigation skill (for root cause checklist)
  └── Phase 3: Fix Proposal
        └── invokes: fix-proposal skill (for code change workflow)
```

**In an agent, reference a skill by name** and give it context:
```markdown
3. **Root Cause Analysis**: Use the `iss-investigation` skill to analyse the exception
   details from Step 1b. Provide the skill with: exception type, message, stack trace,
   and version information.
```

**Never** put phase-level orchestration (environment validation, user confirmation gates,
multi-step data retrieval) inside a skill. That belongs in the agent.

## Quality Checklist

Before publishing a skill, verify:

- [ ] Front matter has `name` and `description`
- [ ] Description contains specific positive triggers (entity names, tool names, action verbs)
- [ ] Description includes a negative boundary if there is plausible overlap with another skill
- [ ] Two-sided description test applied (passing task fires it; adjacent task does not)
- [ ] H1 heading is present immediately after front matter
- [ ] No repo-specific paths, branch names, or product constants in SKILL.md
- [ ] SKILL.md is within the size target (≤ 200 lines for procedural, ≤ 400 for reference)
- [ ] All reference files use conditional loading — the skill tells the agent *when* to load each one
- [ ] No speculative "might be needed someday" content
- [ ] Workflow steps use imperative voice
- [ ] Fast paths appear before full workflows
- [ ] Non-obvious "do NOT" rules are stated explicitly
- [ ] Maintenance section explains where repo-specific additions belong (if references/ exists)
- [ ] At least one `.eval.yaml` exists in `WTG.AI.Prompts/evals/<domain>/` for this skill
- [ ] Evalcases cover happy path, at least one `type: contains` technical output, and a `type: skill-trigger` check
- [ ] If the skill has "do NOT" rules, at least one negation eval exists
- [ ] Evals were run and pass before the skill is considered complete
