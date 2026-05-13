---
type: skill
id: node-generation
title: Node Generation
description: "Generates all node files (skills, prompts, workflow, manifest) from an approved design specification"
tags: [Production, Builder, Stage-3]
connections:
  - target: llm-service
    type: runs_on
  - target: skrpt-authoring-reference
    type: references
metadata:
  complexity: high
  avg_tokens: 4000
---

## Capability

Takes a structured design specification (from Stage 2) and generates every file in the skrpt package: skills, prompts, workflow, services, sources, manifest, and tags. Each file follows the exact format defined in the Skrpt Authoring Reference.

## What It Does

1. **Parses the design** — reads the structured JSON/YAML design document and extracts every node's ID, title, type, connections, and purpose
2. **Generates skill files** — creates each skill's frontmatter (type, id, title, description, tags, connections, metadata) and body (capability, what it does, inputs, outputs)
3. **Generates prompt files** — creates each prompt's frontmatter (including `inputs` for first-step prompts and `derived_from` connection) and body (purpose, prompt instructions with correct variable references)
4. **Generates workflow file** — creates the workflow frontmatter (connections, execution block, loops, output_step, composite_steps) and body (overview, pipeline stages, inputs table, outputs table)
5. **Generates manifest** — creates `skrptiq.yaml` with correct identity, discovery, requires, and contents counts
6. **Generates tags** — creates `tags.yaml` with appropriate tag definitions
7. **Generates service files** — creates any service declarations beyond the standard `llm-service`

## Output

Complete file set as fenced code blocks, each labelled with its path (e.g. `skills/my-skill.md`). Every file is copy-ready — the user can save them directly into a workspace.

## Limitations

The generator produces structurally valid files but cannot guarantee the prompt content will produce optimal AI output. The prompts are functional starting points — the user should refine them after testing.
