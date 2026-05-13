---
type: prompt
id: generate-nodes
title: Generate Nodes
description: "Generates all skrpt node files from a structured design specification"
tags: [Production, Builder, Stage-3]
inputs:
  approved_design:
    label: "Approved Design"
    description: "The structured design specification from Stage 2 (Skrpt Design). Paste the design output — it should contain node inventory, execution plan, connections, and input/output contract in structured format."
    example: "{ nodes: [{ type: 'skill', id: 'analyse-data', title: 'Analyse Data', ... }], execution: [...], manifest: { name: 'my-skrpt', ... } }"
    required: true
    type: longtext
connections:
  - target: node-generation
    type: derived_from
metadata:
  output_format: markdown
  prompt_type: generation
---

## Purpose

Drives the node generation step in the build loop. Takes a structured design specification and generates every file in the skrpt package.

**Context technique:** This prompt expects structured data (JSON or YAML), not free-text prose. Parse the design document as a specification — each node entry becomes a file. Do not re-interpret or paraphrase the architecture. Generate directly from the specification.

## Prompt

You are a skrpt generator. You have a structured design specification and must produce every file in the skrpt package, following the exact format from the Skrpt Authoring Reference.

### Design Specification

{{input.approved_design}}

### Previous Build (if revising)

{{loop.lastOutput}}

### Review Feedback (if revising)

{{loop.lastReview}}

If a previous build and review feedback are provided above, **revise the files to address every point in the feedback.** Do not regenerate from scratch unless the feedback requires it.

If no previous build is provided (first iteration), generate all files from the design specification.

---

Generate every file in the package. For each file, output a fenced code block with the file path as the label:

### File Generation Rules

**Skills:**
- Frontmatter: `type: skill`, `id`, `title`, `description`, `tags`, `connections` (including `runs_on: llm-service`), `metadata`
- Gate skills: add `metadata: { gate: true }`
- Body sections: Capability, What It Does (numbered steps), Output
- Use British English throughout

**Prompts:**
- Frontmatter: `type: prompt`, `id`, `title`, `description`, `tags`, `connections` (including `derived_from: <skill-id>`), `metadata`
- First-step prompts: declare `inputs` frontmatter for every `{{input.field_name}}` used in the body
- Later prompts: reference upstream output via `{{steps.Step Title.output}}`
- Body: Purpose section, then the actual prompt with role instruction and numbered steps
- Use British English throughout

**Workflow:**
- Frontmatter: `type: workflow`, `id`, `title`, `description`, `tags`, `connections` (all `uses` edges + `runs_on`), `metadata`, `output_step`, `composite_steps`, `execution` block, `loops` (if applicable)
- Body: Overview, Pipeline Stages (one subsection per step), Inputs table, Outputs table
- Use British English throughout

**Manifest (skrptiq.yaml):**
- All required fields: name, display_name, description, version, engine, author, licence, category, tags, requires, contents
- `contents` counts MUST match the actual number of files generated

**Tags (tags.yaml):**
- One entry per tag used in the package

**Service:**
- Use the canonical `llm-service` unless additional services are needed

### Output Format

Output each file as:

````
```yaml
# skills/my-skill.md
---
type: skill
...
---

Content here.
```
````

Generate ALL files. Do not skip any node from the design specification.

## Formatting Rules

- Use British English throughout
- Every prompt must have exactly one `derived_from` connection
- Every skill must have a `runs_on` connection
- Every `{{input.field_name}}` must have a matching `inputs` entry
- IDs must be lowercase with hyphens only
- No duplicate IDs
