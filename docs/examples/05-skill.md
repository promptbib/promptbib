# Example 5: Skill (agent profile)

A skill is an agent capability bundle. It declares triggers, tools, and instructions the agent reads when the skill is active. This example requires the **agent profile**.

## The component

```yaml
---
handle: cod.skill.docx
version: 1.0.0
kind: skill
spec: 0.1.0
description: Create, read, or edit Word documents
triggers:
  - keywords: [word, docx, document, "word doc"]
  - file_types: [.docx]
  - context_match: "user is asking to create or edit a formatted document"
tools:
  - met.tool.bash@^1.0.0
  - met.tool.file-read@^1.0.0
  - met.tool.file-write@^1.0.0
inputs:
  target_file:
    type: context_window
    priority: medium
    max_tokens: 8000
    truncation: smart
    required: false
    description: Existing .docx content when editing
compatibility:
  runtime_profile: agent
  tested_models: [claude-opus-4-7, claude-sonnet-4-6]
  compatible_models: ["claude-*"]
tags: [skill, docx, documents]
---
# DOCX creation, editing, and analysis

## When this skill is active

You have been asked to create, read, or modify a Microsoft Word (.docx) document.
Follow the steps below in order.

## Reading content

For text extraction, use the `extract-text` tool via bash:

```bash
extract-text document.docx
```

For raw XML access when you need tracked changes or structural details:

```bash
python scripts/office/unpack.py document.docx unpacked/
```

## Creating new documents

Use the `docx` npm package. Key constraints:

- Always set page size explicitly (default is A4; use US Letter for US documents).
- Never use Unicode bullet characters — use `LevelFormat.BULLET` with numbering config.
- Tables need dual widths: `columnWidths` on the table AND `width` on each cell.

## Editing existing documents

1. Unpack the .docx into its XML components.
2. Edit the XML files directly using the Edit tool.
3. Repack into a new .docx.

## Validation

After creating or editing, validate:

```bash
python scripts/office/validate.py output.docx
```

If validation fails, unpack, fix the XML, and repack.
```

## Walkthrough

**`kind: skill`.** This kind is only available under the agent profile (Part 5 §5.2.3). A core-profile runtime encountering this component should either skip it or treat it as opaque.

**`triggers:`.** Three different match types:

- `keywords` — the user message contains any of these.
- `file_types` — a file with this extension is in context.
- `context_match` — a semantic match on the conversational state (implementation-defined).

The agent evaluates triggers continuously and activates the skill when any trigger matches. Multiple skills can be active simultaneously.

**`tools:`.** The skill declares three tool components it requires. When the skill is active, the runtime makes these tools' schemas available to the model. If a tool is unavailable, the runtime warns but still activates the skill (the skill body may handle missing tools gracefully).

**`inputs:` with a `context_window` type.** This is the agent-profile input type. The skill declares that when editing an existing document, it wants up to 8000 tokens of the file's content, with smart truncation if the file is larger. Priority is medium, so if multiple context inputs compete for budget, higher-priority ones win.

**Body is the actual skill instructions.** When the skill is active, this body is injected into the agent's effective prompt. The body can use all standard composition features — this example is simple, but a skill could extend a pattern, include primitives, or reference tokens.

## How this relates to existing skill systems

Claude Code's skill directory uses `SKILL.md` files with YAML front matter. The mapping is nearly direct:

| Claude Code `SKILL.md` | This spec |
|------------------------|-----------|
| `name:` | `handle:` (last segment) |
| `description:` | `description:` |
| Frontmatter triggers (implicit in description) | `triggers:` (explicit) |
| Script dependencies | `tools:` |
| Body | Body |

Migrating an existing skill to this spec is mostly additive: add `handle`, `version`, `kind`, `spec`, and structure the triggers explicitly. The body stays the same.

## What agents do with it

A Claude Code-like runtime with agent profile support would:

1. At startup or project load, scan all declared skills in its registries.
2. Build a trigger index (keyword → skills, file-type → skills, etc.).
3. On each turn, evaluate triggers against current state.
4. For each active skill, resolve its dependencies (tools, included components, tokens).
5. Inject the rendered skill bodies into the agent's effective prompt.
6. Make tool schemas available for tool-calling.

This is effectively what Claude Code already does with `SKILL.md` files — the spec just formalizes the structure so skills become portable across agent implementations.

## A skill that extends a pattern

Skills can use the full composition system:

```yaml
---
handle: cod.skill.security-review
version: 1.0.0
kind: skill
spec: 0.1.0
extends: cod.review.security@^1.0.0
triggers:
  - keywords: [security review, security audit, vulnerability check]
tools:
  - met.tool.file-read@^1.0.0
slot_fills:
  # Adjust the focus slot for agent use — the agent reads files itself
  focus: |
    Look for security vulnerabilities in code the user provides or
    that you read via file-read. Report findings with file path and line.
compatibility:
  runtime_profile: agent
---
# When to invoke

When the user asks for a security review of code:

1. If specific files are mentioned, read them with the `file-read` tool.
2. Otherwise, ask which files to review.
3. Apply the extended security review task to each file.
4. Aggregate findings grouped by severity.
```

This skill extends the `cod.review.security` task from Example 4, adjusting it for the agent context (which has file-reading capability and multi-step interaction).

## Cross-profile compatibility

A component marked `runtime_profile: agent` cannot run on a core-profile runtime. But the converse works: a core-profile task like `cod.review.security` can be invoked *from* an agent-profile skill, because the agent profile is a strict superset of core.

This is the spec's answer to "how do we share components between static apps and dynamic agents?" — write the underlying reasoning as core-profile components (tasks, patterns, primitives) and layer agent-specific wrappers on top as skills. The agent profile's extra features (triggers, tools, context windows) live in the wrapping layer, and the reusable reasoning stays portable.

## What this demonstrates

- How the agent profile adds `skill` and `tool` kinds without breaking core.
- The `context_window` input type for dynamic context.
- Trigger declarations that give agents a way to activate skills autonomously.
- Tool attachment as a first-class dependency.
- How existing skill systems (like Claude Code's SKILL.md) map onto this spec with minimal change.
- How agent components can extend core components for reuse across contexts.
