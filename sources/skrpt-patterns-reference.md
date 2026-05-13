---
# Canonical source: examples/skrpt-builder/sources/skrpt-patterns-reference.md
# If updating, sync both copies
type: source
id: skrpt-patterns-reference
title: Skrpt Patterns Reference
description: "Common pipeline architectures, when to use each, and anti-patterns to avoid"
tags: [Reference, Authoring]
connections: []
metadata:
  source_type: reference
---

## Pipeline Patterns

### Linear Pipeline

The most common pattern. Steps run in sequence — each receives the previous step's output.

```
Input → Step A → Step B → Step C → Output
```

**When to use:** Content generation, data transformation, report building, any workflow where each step refines the previous.

**Typical step count:** 3-5 steps.

**Skeleton:**
```yaml
execution:
  - skill: "gather"
    prompt: "gather-prompt"
    step_type: "generation"
  - skill: "analyse"
    prompt: "analyse-prompt"
    step_type: "synthesis"
  - skill: "produce"
    prompt: "produce-prompt"
    step_type: "content"
```

### Review Loop (until_pass)

Iterative refinement. A drafting step produces output, a review step evaluates it, and the loop continues until the review passes.

```
Input → [Draft → Review] (loop) → Polish → Output
```

**When to use:** Quality-sensitive content (blog posts, reports, code), anything that benefits from iterative improvement.

**Typical step count:** 2 in loop + 1-2 post-loop = 3-4 total.

**Skeleton:**
```yaml
loops:
  - id: "draft-review"
    mode: "until_pass"
    steps: ["drafting", "review"]
    verifier: "review"
    maxIterations: 5
    freshContextPerIteration: true
execution:
  - skill: "drafting"
    prompt: "draft-prompt"
    step_type: "generation"
  - skill: "review"
    prompt: "review-prompt"
    step_type: "validation"
  - skill: "polish"
    prompt: "polish-prompt"
    step_type: "content"
```

### Human-Gated Pipeline

Gate steps pause execution for user input at key decision points. The user's response becomes the step output.

```
Input → Analyse → [Gate: approve?] → Generate → [Gate: approve?] → Output
```

**When to use:** Workflows where intermediate decisions affect downstream steps. The user must validate before expensive generation.

**Typical step count:** 3-5 steps (1-2 gates).

**Key:** Gate skills need `metadata: { gate: true }` in frontmatter.

**Skeleton:**
```yaml
execution:
  - skill: "analyse"
    prompt: "analyse-prompt"
    step_type: "generation"
  - skill: "approve-plan"          # gate step
    prompt: "approve-plan-prompt"
    step_type: "validation"
  - skill: "generate"
    prompt: "generate-prompt"
    step_type: "generation"
```

### Batch Processing (for_each)

Process multiple items with the same pipeline. Each iteration is independent.

```
Input (list) → [Process → Validate] (for each item) → Summarise → Output
```

**When to use:** Translating to multiple languages, processing multiple documents, analysing multiple data sources.

**Typical step count:** 1-3 in loop + 1 post-loop = 2-4 total.

**Skeleton:**
```yaml
loops:
  - id: "batch"
    mode: "for_each"
    steps: ["process-item"]
    maxIterations: 20
execution:
  - skill: "process-item"
    prompt: "process-prompt"
    step_type: "generation"
  - skill: "summarise"
    prompt: "summarise-prompt"
    step_type: "synthesis"
```

Prompts inside the loop use `{{loop.item}}` to access the current item. Post-loop steps use `{{loop.batch.results}}` for aggregated output.

### Fan-Out Review

Multiple independent review agents run in parallel, then a synthesis step combines their findings.

```
Input → Draft → [Review A | Review B | Review C] → Synthesise → Output
```

**When to use:** Multi-perspective analysis, compliance checks, code review with different focus areas.

**Typical step count:** 1 + parallel count + 1 = 4-6 total.

**Skeleton:**
```yaml
execution:
  - skill: "draft"
    prompt: "draft-prompt"
    step_type: "generation"
  - parallel:
    - skill: "review-style"
      prompt: "style-prompt"
      step_type: "review"
    - skill: "review-security"
      prompt: "security-prompt"
      step_type: "review"
    - skill: "review-logic"
      prompt: "logic-prompt"
      step_type: "review"
  - skill: "synthesise"
    prompt: "synthesise-prompt"
    step_type: "synthesis"
```

## Choosing a Pattern

| Goal | Pattern | Why |
|------|---------|-----|
| Transform input through stages | Linear | Each step builds on the last |
| Iterative quality improvement | Review loop | Verifier drives revision |
| User must approve before continuing | Gated | Gates prevent wasted work |
| Process a list of items | Batch (for_each) | Same pipeline per item |
| Multiple independent analyses | Fan-out | Parallel branches, one synthesis |
| Guided interactive experience | Gated + loop | Gates collect input, loop refines |

## Anti-Patterns

**Too many steps.** More than 8 steps adds complexity without proportional value. If you need that many, consider splitting into multiple skrpts or using a dependency.

**Missing derived_from.** Every prompt MUST have a `derived_from` connection to its skill. Without it, the app cannot resolve the skill→prompt mapping.

**Undeclared inputs.** If a prompt body uses `{{input.city}}`, the prompt frontmatter must declare `inputs: { city: { label: "...", description: "..." } }`. Missing declarations break the input form.

**Validation outside loops.** A validation step only makes sense inside a loop where it can trigger re-generation. A standalone validation step at the end of a pipeline produces a verdict but can't act on it.

**Gate steps with no guidance.** Gate prompts must tell the user what to check and what responses are valid (e.g. "Type 'approved' or provide feedback"). An empty gate confuses users.

**Prompts without context.** Later-stage prompts that don't reference `{{steps.previous.output}}` or `{{steps.Step Title.output}}` lose the pipeline's data flow. Every prompt after step 1 should reference upstream output.

**Overly complex first skrpt.** Start with a linear 3-step pipeline. Add loops, gates, and parallel branches only when the use case demands them.
