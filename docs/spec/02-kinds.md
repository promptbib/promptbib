# Part 2: Component kinds

> **Layer 2** of the spec. Defines the taxonomy of component kinds and what each can do.

Every component declares a `kind`. The kind determines what fields are meaningful, how the component may be composed, and what the runtime expects from it. This spec defines eight kinds; runtimes MUST support all eight (within their declared profile — see [Part 5](05-profiles.md)).

## 2.1 Kind overview

| Kind | Purpose | Profile | Composable as |
|------|---------|---------|---------------|
| `token_bundle` | Named terminal values | core | token reference |
| `primitive` | Reusable instruction fragment | core | extends target, include target, slot fill |
| `pattern` | Structural template with slots | core | extends target |
| `task` | Complete invokable prompt | core | extends target (rarely) |
| `workflow` | Declared sequence of tasks | core | runtime invocation only |
| `skin` | Metadata and token overlay | core | applied at invocation |
| `skill` | Agent capability bundle | agent | loaded by trigger |
| `tool` | Tool schema + invocation guidance | agent | referenced by skill or task |

The five **core kinds** (`token_bundle`, `primitive`, `pattern`, `task`, `workflow`) are sufficient for static prompt libraries and standard application use. The three **agent-profile kinds** (`skin`, `skill`, `tool`) add capability required by coding agents and other dynamic consumers.

`skin` is in the core column because it's useful everywhere. The agent-profile kinds are covered in more detail in [Part 5](05-profiles.md).

## 2.2 Role axis vs domain axis

Kind is the **role axis** — what this component *does*. The `handle` places the component on the **domain axis** — what it's *about* (code, marketing, compliance, etc.).

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

Dashes mark combinations that are meaningless: a `workflow` under the `met` (meta) domain doesn't make sense, and token bundles or skins are always meta-layer concerns.

The spec does not enforce this table — that's a registry governance concern. But runtimes SHOULD warn when a component occupies a dashed cell.

## 2.3 `token_bundle`

A terminal value dictionary. Token bundles define named strings (or structured values) that other components reference via `{{token: ...}}`.

**Required fields:** `handle`, `version`, `kind`, `spec`.
**Body:** absent or ignored. The YAML front matter contains the bundle entries under a `values` field.

```yaml
---
handle: met.tone
version: 1.0.0
kind: token_bundle
spec: 0.1.0
description: Named tone values for prompt authors
values:
  formal: "Use precise, professional language. Avoid contractions."
  terse: "Be maximally concise. No preamble, no caveats."
  conversational: "Write naturally, as if explaining to a colleague."
---
```

**Rules:**

- Token bundles MUST NOT contain `inputs`, `slots`, `outputs`, or `extends`.
- Token values MUST be strings, numbers, booleans, or lists of strings. Nested mappings are not permitted — a token is a leaf.
- A component references a token via `{{token: handle.key}}` or via a `token_ref`-typed input.
- Adding a new key is a minor version bump. Removing or renaming a key is a major bump. Changing a value is a minor bump unless the change alters meaning materially (author's judgment, documented in changelog).

## 2.4 `primitive`

A reusable instruction fragment. Primitives are the smallest composable unit that carries prompt text.

**Required fields:** `handle`, `version`, `kind`, `spec`, body.
**Optional fields:** `inputs`, `outputs`, `compatibility`, `evals`.

```yaml
---
handle: met.p.reviewer-voice
version: 1.0.0
kind: primitive
spec: 0.1.0
description: Establishes a code-reviewer persona
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

- Primitives MUST NOT declare `slots`. If a primitive needs variability, it uses `inputs`.
- Primitives MUST NOT `extends` another component. A primitive is a leaf in the composition graph.
- Primitives MAY declare `outputs`, but doing so is rare — primitives are usually fragments combined into larger components that declare the ultimate output.
- Primitives MAY be included in other components via `{{include: ...}}` or installed into slots of patterns and tasks.

## 2.5 `pattern`

A structural template with declared slots that specializations fill. Patterns encode the *shape* of a class of prompts without committing to specific content.

**Required fields:** `handle`, `version`, `kind`, `spec`, body.
**Required if non-trivial:** `slots`.
**Optional fields:** `inputs`, `outputs`, `compatibility`, `evals`.

```yaml
---
handle: cod.review.base
version: 1.0.0
kind: pattern
spec: 0.1.0
description: Base structure for code review. Specializations fill slots.
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

- Patterns MUST declare at least one slot. A pattern with no slots SHOULD be a primitive instead.
- Patterns MAY be extended by tasks or by more specific patterns.
- Patterns SHOULD declare `outputs` if they commit to an output shape. If slot fills determine output shape, `outputs` MAY be omitted and inherited from the slot-filled format primitive.

## 2.6 `task`

A complete, invokable component. A task is what application code calls at runtime.

**Required fields:** `handle`, `version`, `kind`, `spec`.
**Typically has:** `extends`, `slot_fills`, `inputs`, `outputs`.
**Body:** usually absent (inherited from extended pattern) but MAY be present.

```yaml
---
handle: cod.review.security
version: 1.0.0
kind: task
spec: 0.1.0
extends: cod.review.base@^1.0.0
description: Security-focused code review with CWE references
inputs:
  severity_threshold:
    type: enum[low, medium, high]
    default: medium
slot_fills:
  persona: met.p.reviewer-voice@^1.0.0
  format: met.p.format-json-findings@^1.0.0
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

- A task is the only kind a typical runtime invocation targets. Callers invoke tasks; they do not invoke patterns or primitives directly.
- A task MUST satisfy all required slots of any pattern it extends. Unfilled required slots produce a load-time validation error.
- A task MAY itself be extended, but this is uncommon — chains of task-extending-task usually indicate the base should be refactored into a pattern.
- A task without `extends` MUST have a body.

## 2.7 `workflow`

A declared sequence of task invocations with data flow between them. Workflows describe multi-step processes at a level a runtime can execute.

**Required fields:** `handle`, `version`, `kind`, `spec`, `steps`.
**Body:** absent. A workflow is structural, not prose.

```yaml
---
handle: cod.flow.pr-review
version: 1.0.0
kind: workflow
spec: 0.1.0
description: Lint, security-review, and summarize a pull request
inputs:
  diff: { type: string, required: true }
  language: { type: enum[python, typescript, go], required: true }
steps:
  - id: lint
    task: cod.review.lint@^1.0.0
    inputs:
      code: "{{inputs.diff}}"
      language: "{{inputs.language}}"
  - id: security
    task: cod.review.security@^1.0.0
    inputs:
      code: "{{inputs.diff}}"
      language: "{{inputs.language}}"
      severity_threshold: high
  - id: summary
    task: cod.review.summarize@^1.0.0
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
- Workflows reference tasks (not primitives or patterns) in their steps.
- Step inputs MAY reference workflow `inputs` or prior step outputs via `{{steps.id.output...}}` expressions.
- Workflow execution semantics (parallelism, error handling, retries) are implementation-defined. The spec describes only the declaration.

## 2.8 `skin`

A metadata and token overlay applied at invocation time. Skins adjust tone, format, or constraints without changing the component logic.

**Required fields:** `handle`, `version`, `kind`, `spec`, `overrides`.
**Body:** absent.

```yaml
---
handle: met.skin.formal
version: 1.0.0
kind: skin
spec: 0.1.0
description: Apply formal tone and strict output formatting globally
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
- Skins MAY override token values (redirecting one token to another) and default input values.
- Skins are applied at invocation: `task.run(..., skin="met.skin.formal")`. They are not baked into components.
- Multiple skins MAY be applied in order; later skins override earlier ones.

## 2.9 `skill` (agent profile)

A capability bundle for agents. A skill declares triggers that cause an agent to load the component and instructions that guide the agent's behavior when the skill is active.

**Required fields:** `handle`, `version`, `kind`, `spec`, body, `triggers`.
**Optional fields:** `inputs`, `tools`, `compatibility`, `evals`.

```yaml
---
handle: cod.skill.docx
version: 1.0.0
kind: skill
spec: 0.1.0
description: Create, read, or edit Word documents
triggers:
  - keywords: [word, docx, document]
  - file_types: [.docx]
tools:
  - met.tool.bash@^1.0.0
  - met.tool.file-io@^1.0.0
compatibility:
  tested_models: [claude-opus-4-7]
---
# Body is the actual skill instructions the agent reads when triggered.
```

**Rules:**

- Skills are an agent-profile kind. Core-profile runtimes MAY treat them as opaque components.
- `triggers` is an unordered set of match criteria. Runtime-specific trigger types are permitted as extensions.
- `tools` lists tool components the skill requires. The runtime MUST make these available to the agent when the skill is active.
- A skill body MAY use any composition feature ([Part 3](03-composition.md)) — extending a pattern, filling slots, including primitives.

See [Part 5 §5.2](05-profiles.md) for full agent-profile rules.

## 2.10 `tool` (agent profile)

A tool schema and invocation guidance. Tools describe capabilities an agent can invoke and, optionally, prompt text about when and how to use them.

**Required fields:** `handle`, `version`, `kind`, `spec`, `schema`.
**Optional fields:** body, `compatibility`, `evals`.

```yaml
---
handle: met.tool.web-search
version: 1.0.0
kind: tool
spec: 0.1.0
description: Search the web for current information
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
(prices, news, weather, current officeholders). Do not use for stable
facts. Keep queries short; prefer 1-6 words.
```

**Rules:**

- Tools are an agent-profile kind.
- `schema` MUST follow a JSON Schema-compatible form. The input and output shapes are the tool's contract.
- The body, if present, is invocation guidance the agent receives alongside the schema.
- A tool MAY be referenced by skills (via `tools:`) or by tasks (via `input_fills` when the task uses a primitive that accepts a tool reference).

## 2.11 Authoring guidance

Two design rules that cut across all kinds:

**Prefer fewer, larger primitives over many small ones.** Research on instruction-tuned models suggests that heavy fragmentation of instructions can degrade performance relative to a single coherent block. Primitives should be reusable *units of meaning*, not minimum-size *units of text*.

**Prefer patterns with few slots over patterns with many.** Each slot is an extension point; each extension point is a source of variation. A pattern with eight slots is usually three patterns pretending to be one.

## 2.12 Open question: canonical domain table

The three-letter domains used in examples throughout the spec (`cod`, `mkt`, `met`, etc.) are illustrative. The actual canonical table — which domains exist, who owns each, and how new ones are added — is a registry governance concern, not a spec concern.

This is an open question for the spec community to resolve before 1.0. See README.
