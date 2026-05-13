---
# Canonical source: examples/skrpt-builder/sources/skrpt-authoring-reference.md
# If updating, sync both copies
type: source
id: skrpt-authoring-reference
title: Skrpt Authoring Reference
description: "Complete reference for skrpt structure: node types, connections, variables, execution model, manifest format"
tags: [Reference, Authoring]
connections: []
metadata:
  source_type: reference
---

## Skrpt Structure

A skrpt is a directory containing a manifest (`skrptiq.yaml`), optional `tags.yaml`, and node files organised in typed subdirectories.

```
my-skrpt/
├── skrptiq.yaml        # Package manifest
├── tags.yaml            # Tag definitions (optional)
├── skills/              # Capability definitions
├── prompts/             # Prompt templates
├── workflows/           # Orchestration (usually one)
├── services/            # External service declarations
├── sources/             # Reference material
├── documents/           # Longer-form knowledge
└── assets/              # Templates, criteria, config
```

## Node Types

### Skill

A capability the workflow invokes. Each skill does one thing well.

```yaml
---
type: skill
id: analyse-data        # lowercase, hyphens only
title: Analyse Data
description: "Interprets structured data and produces insights"
tags: [Production]
connections:
  - target: llm-service
    type: runs_on
  - target: data-source
    type: references    # optional: reference material
---
```

Body describes: capability, when to use, what it does (numbered steps), inputs table, outputs.

**Gate steps** add `metadata: { gate: true }` — execution pauses for user input. The user's response becomes the step output.

### Prompt

The instructions that drive a skill. Every non-utility skill needs one prompt with a `derived_from` connection pointing back to the skill.

```yaml
---
type: prompt
id: analyse-data-prompt
title: Analyse Data
description: "Interprets data and produces insights"
tags: [Production]
inputs:                  # declare if body uses {{input.*}}
  dataset:
    label: "Dataset"
    description: "The data to analyse"
    example: "Q4 sales figures"
    required: true
    type: text
connections:
  - target: analyse-data
    type: derived_from   # REQUIRED — links prompt to its skill
metadata:
  output_format: structured   # or: markdown, json, plain
  prompt_type: task           # or: analysis, generation
---
```

Body contains the actual prompt: purpose section, then the prompt instructions with variable references.

### Workflow

Orchestrates skills into a pipeline. Usually one per skrpt.

```yaml
---
type: workflow
id: my-pipeline
title: My Pipeline
description: "Does the thing"
tags: [Production]
connections:
  - target: skill-one
    type: uses
  - target: skill-two
    type: uses
  - target: llm-service
    type: runs_on
metadata:
  estimated_duration: "10-30 seconds"
  trigger: manual
output_step: "final-skill"
composite_steps:
  - "skill-one"
  - "skill-two"
  - "final-skill"
execution:
  - skill: "skill-one"
    prompt: "skill-one-prompt"
    step_type: "generation"
  - skill: "skill-two"
    prompt: "skill-two-prompt"
    step_type: "synthesis"
  - skill: "final-skill"
    prompt: "final-prompt"
    step_type: "content"
---
```

Body describes: overview, pipeline stages, inputs table, outputs table, setup instructions.

### Service

An external service the skrpt depends on. Most skrpts need `llm-service`. MCP and API services declare auth metadata.

```yaml
---
type: service
id: llm-service
title: LLM Service
description: "Language model service"
tags: [Production]
connections: []
metadata:
  serviceType: llm          # or: api, mcp
  auth_type: api_key        # see valid values below
  env_var: OPENAI_API_KEY   # required if auth_type is not none
---
```

Valid `auth_type` values: `api_key`, `api_key_optional`, `api_token`, `personal_access_token`, `bot_token`, `oauth`, `bearer_token`, `none`.

### Source

Reference material the LLM reads during execution. API schemas, style guides, knowledge bases.

```yaml
---
type: source
id: my-reference
title: My Reference
description: "Reference material for the pipeline"
tags: [Reference]
connections: []
---
```

### Document

Longer-form knowledge content. Guides, manuals, domain knowledge.

```yaml
---
type: document
id: my-guide
title: My Guide
description: "Domain knowledge for the pipeline"
tags: []
connections: []
---
```

### Asset

Templates, criteria cards, configuration files.

```yaml
---
type: asset
id: quality-criteria
title: Quality Criteria
description: "Criteria card for review steps"
tags: []
connections: []
metadata:
  format: yaml    # or: json, csv, html, text
---
```

## Connection Types

| Type | Meaning | Common usage |
|------|---------|-------------|
| `uses` | Workflow uses a skill | workflow → skill |
| `derived_from` | Prompt is derived from a skill | prompt → skill (REQUIRED) |
| `runs_on` | Skill/workflow runs on a service | skill → service |
| `references` | References material | skill → source/document/asset |
| `depends_on` | Cannot function without | skill → service |
| `requires` | Prerequisite | workflow → skill |
| `tagged` | Categorised by | any → tag |

## Variable System

### User Input

`{{input.variable_name}}` — collected in the execution form before the workflow runs. Every `{{input.*}}` variable in a prompt body MUST have a matching entry in the prompt's `inputs` frontmatter.

### Step Output

`{{steps.previous.output}}` — output of the positionally previous step.
`{{steps.Step Title.output}}` — output of a named step (use the skill's Title Case title).

### Loop Variables

**for_each mode:**
- `{{loop.item}}` — current item from the input list
- `{{loop.index}}` — current iteration index (0-based)
- `{{loop.total}}` — total items in the list

**until_pass mode:**
- `{{loop.lastOutput}}` — the previous iteration's output
- `{{loop.lastReview}}` — the verifier's last feedback

**After loop (post-loop steps):**
- `{{loop.<id>.results}}` — aggregated results (for_each)
- `{{loop.<id>.final}}` — final approved output (until_pass)
- `{{loop.<id>.status}}` — pass, fail, or max_iterations_reached

### Step Context

`{{step.context.param_name}}` — author-configured context values set in the execution block. Not user-facing.

## Step Types

| Type | Behaviour | Output contract |
|------|-----------|----------------|
| `generation` | Creates from scratch | Full artefact |
| `content` | Refines/transforms | Complete revised document (not deltas) |
| `review` | Evaluates and reports | Evaluation report |
| `synthesis` | Condenses, restructures | Condensed artefact |
| `validation` | Pass/fail verdict | `{status: "pass"\|"continue"\|"fail", reason, instructions}` |

## Execution Block

The `execution` array in workflow frontmatter defines the pipeline. Order = data flow.

```yaml
execution:
  # Sequential step
  - skill: "step-one"
    prompt: "step-one-prompt"    # which prompt drives this step
    step_type: "generation"
    context:                      # optional author-configured params
      tone: "formal"

  # Parallel block — branches run simultaneously
  - parallel:
    - skill: "review-a"
      prompt: "review-a-prompt"
      step_type: "review"
    - skill: "review-b"
      prompt: "review-b-prompt"
      step_type: "review"

  # Sequential step after parallel
  - skill: "final-step"
    prompt: "final-prompt"
    step_type: "synthesis"
```

Rules:
- Every execution entry must have `skill` and `step_type`
- Every non-utility skill should have `prompt` referencing a prompt node
- Parallel blocks need 2+ steps
- No nested parallel blocks

## Loops

Declared in **workflow frontmatter** (NOT in skrptiq.yaml).

### until_pass

Iterates until the verifier returns `{status: "pass"}`.

```yaml
loops:
  - id: "refinement"
    mode: "until_pass"
    steps: ["drafting", "review"]
    verifier: "review"           # must be in steps list
    maxIterations: 5
    freshContextPerIteration: true
```

The verifier skill must produce: `{status: "pass"|"continue"|"fail", reason: "...", instructions: "..."}`. Gate steps can serve as verifiers — the user decides pass/continue.

### for_each

Iterates over a list provided by the user.

```yaml
loops:
  - id: "batch"
    mode: "for_each"
    steps: ["process-item", "validate-item"]
    maxIterations: 20
```

The input list comes from a user input variable (e.g. JSON array or one item per line).

## Manifest (skrptiq.yaml)

```yaml
name: "my-skrpt"                    # lowercase, hyphens, required
display_name: "My Skrpt"            # optional, defaults to name
description: "What it does"         # required for Hub
version: "1.0.0"                    # semver, required
engine: ">=1.0.0"

author:
  name: "skrptiq"
  url: "https://hub.skrptiq.ai"

licence: "MIT"                      # SPDX identifier, required

category: "developer"               # Hub category slug
tags: ["tag1", "tag2"]

requires:
  services: ["llm-service"]         # service node IDs
  permissions: []                    # e.g. network:example.com
  data_handling: []                  # e.g. source-code, credentials

contents:                            # must match actual file counts
  skills: 3
  prompts: 3
  workflows: 1
  services: 1
  sources: 0
  documents: 0
  assets: 0
  total: 8
```

### Naming Rules

- `name`: lowercase, hyphens, no leading hyphen. Pattern: `[a-z0-9][a-z0-9-]*`
- Node `id`: same pattern as name
- Node `type`: must match its directory (skill in skills/, prompt in prompts/, etc.)
- No duplicate node IDs within a skrpt
