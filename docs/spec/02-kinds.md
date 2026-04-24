# Part 2: Component kinds

> **Layer 2** of the spec. Defines the taxonomy of component kinds and what each can do.

Every component at Level 1 and above declares a `kind` under `metadata.pbib.kind`. The kind determines what fields are meaningful, how the component may be composed, and what the runtime expects from it.

At **Level 0**, `kind` MAY be omitted. Runtimes infer from context:

- A file named `SKILL.md` → `kind: skill`
- Otherwise, the file is treated as a kind-agnostic text component that can be rendered but not composed.

## 2.1 Kind overview

| Kind | Purpose | Profile | Available at |
|------|---------|---------|--------------|
| `token_bundle` | Named terminal values | core | L1+ |
| `primitive` | Reusable instruction fragment | core | L1+ |
| `pattern` | Structural template with slots | core | L1+ |
| `task` | Complete invokable prompt | core | L1+ |
| `workflow` | Declared sequence of tasks | core | L2 |
| `skin` | Metadata and token overlay | core | L2 |
| `skill` | Agent capability bundle | agent | **L0+** |
| `tool` | Tool schema and invocation guidance | agent | L1+ |

`skill` is the only kind available at Level 0, because SKILL.md is an established Agent Skills format with existing consumers. Every other kind requires Level 1 metadata to be meaningful.

The five **core kinds** (`token_bundle`, `primitive`, `pattern`, `task`, `workflow`) are sufficient for static prompt libraries and applications. The three **agent-profile kinds** (`skin`, `skill`, `tool`) add capability required by coding agents. `skin` stays in the core column because skinning is useful everywhere.

## 2.2 Role axis vs domain axis

Kind is the **role axis** — what this component *does*. The `handle` (when present) places the component on the **domain axis** — what it's *about* (code, marketing, compliance, etc.).

A valid component table is two-dimensional:

```
                 cod     qas     arc     mkt     met
token_bundle     —       —       —       —       ✓
primitive        ✓       ✓       ✓       ✓       ✓
pattern          ✓       ✓       ✓       ✓       ✓
task             ✓       ✓       ✓       ✓       —
workflow         ✓       ✓       ✓       ✓       —
skin             —       —       —       —       ✓
```

Dashes mark combinations that don't make sense. The spec does not enforce this table — that's a registry governance concern. But runtimes SHOULD warn when a component occupies a dashed cell.

## 2.3 The four reuse mechanisms

Before walking through each kind, a summary of how components reuse each other. Composition is the whole point of the system; these are the mechanisms that enable it.

| Mechanism | Syntax | Relationship | Use when |
|-----------|--------|--------------|----------|
| **Extension** | `extends:` in front matter | Child IS-A parent | Creating variants of the same thing |
| **Inclusion** | `{{include: ...}}` in body | Component USES content | Shared text fragments |
| **Slot installation** | `slot_fills:` in front matter | Pattern + choice of filler | Structural composition |
| **Workflow orchestration** | `steps:` (workflow kind) | Task sequencing | Multi-step execution |

Plus **tokens** — named terminal values referenced by `{{token: bundle.key}}` — which are simpler than the other four but round out the picture.

Each mechanism is defined in [Part 3](03-composition.md). The kind documentation below references them.

## 2.4 `token_bundle`

A terminal value dictionary. Token bundles define named strings or structured values that other components reference.

**Level:** L1+ (tokens need identifiers to be referenced).
**Required:** Level 1 fields plus `metadata.pbib.values`.
**Body:** absent.

```yaml
---
name: tone
description: Named tone values for prompt authors
license: MIT
metadata:
  pbib:
    version: 1.0.0
    kind: token_bundle
    handle: met.tone
    values:
      formal: "Use precise, professional language. Avoid contractions."
      terse: "Be maximally concise. No preamble, no caveats."
      conversational: "Write naturally, as if explaining to a colleague."
---
```

**Rules:**

- Token bundles MUST NOT contain `inputs`, `slots`, `outputs`, or `extends`.
- Values MUST be strings, numbers, booleans, or lists of strings. Tokens are leaves — no nested mappings.
- Components reference tokens via `{{token: bundle.key}}` or via a `token_ref`-typed input.
- Adding a key is a minor version bump. Removing or renaming is major. Changing a value is minor unless the change alters meaning materially.

## 2.5 `primitive`

A reusable instruction fragment. The smallest composable unit that carries prompt text.

**Level:** L1+ (primitives need identifiers to be referenced).
**Required:** Level 1 fields plus body.
**Optional:** `inputs`, `outputs`, `runtime_compatibility`, `evals`.

```yaml
---
name: reviewer-voice
description: Establishes a code-reviewer persona
license: MIT
metadata:
  pbib:
    version: 1.0.0
    kind: primitive
    handle: met.p.reviewer-voice
    inputs:
      specialty: { type: string, required: true }
      tone:
        type: token_ref
        bundle: met.tone@^1.0.0
        default: terse
---
You are an experienced code reviewer specializing in {{specialty}}.

{{token: met.tone.[tone]}}

Focus on concrete, actionable findings.
```

**Rules:**

- Primitives MUST NOT declare `slots`. For variability, use `inputs`.
- Primitives MUST NOT `extends` another component. They are leaves in the composition graph.
- Primitives MAY be included in other components via `{{include: ...}}` or installed into slots of patterns and tasks.

## 2.6 `pattern`

A structural template with declared slots that specializations fill.

**Level:** L1+ addressable; inherently compositional at L2.
**Required:** Level 1 fields, `slots`, and body.
**Optional:** `inputs`, `outputs`, `runtime_compatibility`, `evals`.

```yaml
---
name: review-base
description: Base structure for code review tasks. Specializations fill slots.
license: MIT
metadata:
  pbib:
    version: 1.0.0
    kind: pattern
    handle: cod.review.base
    inputs:
      code: { type: string, required: true }
      language: { type: enum[python, typescript, go, rust, java], required: true }
    slots:
      persona:
        type: component_ref
        allowed_kinds: [primitive]
        required: true
      focus:
        type: markdown
        required: true
      format:
        type: component_ref
        allowed_kinds: [primitive]
        required: true
---
{{slot: persona}}

<review_focus>
{{slot: focus}}
</review_focus>

<code language="{{language}}">
{{code}}
</code>

{{slot: format}}
```

**Rules:**

- Patterns MUST declare at least one slot. A pattern with no slots SHOULD be a primitive.
- Patterns MAY be extended by tasks or by more specific patterns.
- Patterns SHOULD declare `outputs` when they commit to an output shape. If slot fills determine output shape, `outputs` MAY be omitted and inherited from the slot-filled format primitive.

## 2.7 `task`

A complete, invokable component. What application code calls.

**Level:** L1+ for standalone tasks; L2 for tasks using `extends`, `slot_fills`, or typed `inputs`.
**Required:** Level 1 fields.
**Typically has:** `extends`, `slot_fills`, `inputs`, `outputs`.
**Body:** usually absent (inherited from extended pattern) but MAY be present.

```yaml
---
name: security-review
description: Security-focused code review with CWE references
license: MIT
metadata:
  author: acme-security-team
  pbib:
    version: 1.0.0
    kind: task
    handle: cod.review.security
    extends: cod.review.base@^1.0.0
    inputs:
      severity_threshold:
        type: enum[low, medium, high]
        default: medium
    slot_fills:
      persona: met.p.reviewer-voice
      format: met.p.format-json-findings
      focus: |
        Look for security vulnerabilities at **{{severity_threshold}}+** severity.
        Reference CWE IDs where applicable.
    input_fills:
      persona:
        specialty: "application security"
        tone: formal
tags: [security, code, review]
---
```

**Rules:**

- A task is the typical runtime invocation target. Callers invoke tasks, not patterns or primitives directly.
- A task MUST satisfy all required slots of any pattern it extends. Unfilled required slots are a load-time validation error.
- A task MAY itself be extended, but this is uncommon — chains of task-extending-task usually indicate the base should be refactored into a pattern.
- A task without `extends` MUST have a body.

## 2.8 `workflow`

A declared sequence of task invocations with data flow between them.

**Level:** L2 (workflows are inherently compositional).
**Required:** Level 1 fields plus `steps`.
**Body:** absent.

```yaml
---
name: pr-review
description: Lint, security-review, and summarize a pull request
license: MIT
metadata:
  pbib:
    version: 1.0.0
    kind: workflow
    handle: cod.flow.pr-review
    inputs:
      diff: { type: string, required: true }
      language: { type: enum[python, typescript, go], required: true }
    steps:
      - id: lint
        task: cod.review.lint
        inputs:
          code: "{{inputs.diff}}"
          language: "{{inputs.language}}"
      - id: security
        task: cod.review.security
        inputs:
          code: "{{inputs.diff}}"
          language: "{{inputs.language}}"
          severity_threshold: high
      - id: summary
        task: cod.review.summarize
        inputs:
          findings:
            - "{{steps.lint.output.findings}}"
            - "{{steps.security.output.findings}}"
    outputs:
      format: "{{steps.summary.output}}"
---
```

**Rules:**

- Workflows MUST NOT have a body. They are pure coordination.
- Workflow steps reference tasks (not primitives or patterns).
- Step inputs MAY reference workflow `inputs` or prior step outputs via `{{steps.id.output...}}`.
- Execution semantics (parallelism, error handling, retries) are implementation-defined.

## 2.9 `skin`

A metadata and token overlay applied at invocation. Skins adjust tone, format, or constraints without changing component logic.

**Level:** L2.
**Required:** Level 1 fields plus `overrides`.
**Body:** absent.

```yaml
---
name: formal
description: Apply formal tone and strict output formatting globally
license: MIT
metadata:
  pbib:
    version: 1.0.0
    kind: skin
    handle: met.skin.formal
    overrides:
      tokens:
        met.tone.terse: met.tone.formal
        met.tone.conversational: met.tone.formal
      inputs:
        tone: formal
---
```

**Rules:**

- Skins MUST only contain overrides. They cannot introduce new prompt content.
- Skins are applied at invocation: `task.run(..., skin="met.skin.formal")`.
- Multiple skins MAY be applied in order; later skins override earlier ones.

## 2.10 `skill` (agent profile)

A capability bundle for agents. A skill declares triggers that cause an agent to load the component and instructions that guide behavior when the skill is active.

**Level:** L0+. This is the only kind that supports Level 0 — SKILL.md files with just `name` and `description` are valid skill components.

### 2.10.1 Level 0 skill — plain SKILL.md

```yaml
---
name: docx
description: Create and edit Microsoft Word documents. Use when the user asks to create, read, or modify .docx files.
license: MIT
---
# DOCX skill

When the user asks about .docx files, follow these steps:

1. For reading content, use extract-text.
2. For creating new documents, use the docx npm package.
3. For editing, unpack, edit XML, repack.
```

This is a working SKILL.md as used by Claude Code today. Our runtime loads it at Level 0.

At Level 0:

- Triggers are inferred from `description`.
- Tools are discovered from the body and from sibling `scripts/` folders.
- The component is not referenceable by other components.

### 2.10.2 Level 1 skill

Add two fields to make the skill addressable:

```yaml
---
name: docx
description: Create and edit Microsoft Word documents
license: MIT
metadata:
  pbib:
    version: 1.0.0
    kind: skill
---
# Body unchanged
```

Now the skill is version-pinned in lockfiles and referenceable by other components.

### 2.10.3 Level 2 skill

Add composition features:

```yaml
---
name: docx
description: Create and edit Microsoft Word documents
license: MIT
metadata:
  pbib:
    version: 1.0.0
    kind: skill
    handle: cod.skill.docx
    triggers:
      - keywords: [word, docx, document]
      - file_types: [.docx]
    tools:
      - met.tool.bash@^1.0.0
      - met.tool.file-io@^1.0.0
    inputs:
      target_file:
        type: context_window
        priority: medium
        max_tokens: 8000
        required: false
    runtime_compatibility:
      profile: agent
      tested_models: [claude-opus-4-7]
---
# Body can now use {{slot: ...}}, {{include: ...}}, etc.
```

**Rules:**

- Skills are an agent-profile kind (at Level 1+). Core-profile runtimes MAY treat them as opaque components.
- At Level 0/L1, triggers are implicit from `description` — matching how existing Agent Skills consumers work.
- At Level 2, `triggers` is an explicit set of match criteria. Runtime-specific trigger types are permitted as extensions.
- `tools` lists tool components the skill requires. The runtime MUST make these available when the skill is active.
- A skill body MAY use any composition feature at Level 2.

See [Part 5 §5.2](05-profiles.md) for full agent-profile rules.

### 2.10.4 Relationship to Agent Skills `allowed-tools`

The Agent Skills spec has an experimental top-level `allowed-tools` field. When present, pbib runtimes SHOULD:

- Read `allowed-tools` as a list of tool names the skill is permitted to use.
- Treat the pbib `metadata.pbib.tools` list as declaring tool components to install.
- Intersect the two: a tool must appear in both `allowed-tools` (permission) and `tools` (availability) to be usable by the agent.

This preserves compatibility with Agent Skills permission models while adding pbib's typed tool-component system.

## 2.11 `tool` (agent profile)

A tool schema and invocation guidance.

**Level:** L1+.
**Required:** Level 1 fields plus `schema`.
**Optional:** body, `runtime_compatibility`, `evals`.

```yaml
---
name: web-search
description: Search the web for current information
license: MIT
metadata:
  pbib:
    version: 1.0.0
    kind: tool
    handle: met.tool.web-search
    schema:
      name: web_search
      input:
        type: object
        required: [query]
        properties:
          query: { type: string, description: "Search query, 1-6 words" }
          max_results: { type: integer, default: 5 }
      output:
        type: object
        properties:
          results:
            type: array
            items:
              type: object
              properties:
                title: { type: string }
                url: { type: string }
                snippet: { type: string }
---
Use web search for current information that changes frequently
(prices, news, weather). Do not use for stable facts. Keep queries
short; prefer 1-6 words.
```

**Rules:**

- `schema` MUST follow a JSON Schema-compatible form.
- The body, if present, is invocation guidance the agent receives alongside the schema.
- A tool MAY be referenced by skills (via `tools:`) or by tasks (via `input_fills` when a task uses a primitive accepting a tool reference).
- Tools MAY wrap MCP server tools — in which case `schema.mcp_server` points at the MCP server and the schema is derived from MCP discovery.

## 2.12 Authoring guidance

Three design rules that cut across kinds:

**Prefer fewer, larger primitives over many small ones.** Heavy fragmentation of instructions can degrade model performance relative to a single coherent block. Primitives should be reusable *units of meaning*, not minimum-size *units of text*.

**Prefer patterns with few slots over patterns with many.** Each slot is an extension point; each extension point is a source of variation. A pattern with eight slots is usually three patterns pretending to be one.

**Start at the lowest level that meets your need.** If a plain SKILL.md works, don't add Level 1 metadata. If your skill doesn't need composition, don't add Level 2 features. Complexity creep is the most common failure mode of component systems.

## 2.13 Open question: canonical domain table

The three-letter domains used throughout the spec (`cod`, `mkt`, `met`) are illustrative. The actual canonical table — which codes exist, who owns each, how new ones are added — is a registry governance concern, not a spec concern.

This is an open question for the spec community to resolve before 1.0. See [README](../README.md).
