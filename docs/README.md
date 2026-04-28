# Portable Agent Component Format

**Status:** Draft
**Editors:** TBD

A file format and semantic model for LLM instruction artifacts — prompts, skills, personas, tools, workflows — that makes them reusable within teams, composable across projects, and verifiable between agents.

## Terminology

| Term | Meaning |
|------|---------|
| **Component** | A file conforming to this spec. Has front matter and optionally a body. |
| **Front matter** | The YAML metadata block at the top of a component file, delimited by `---` markers. Machine-readable. |
| **Body** | The free-form text below the front matter. Becomes part of the prompt sent to the model. Human-readable. |
| **Handle** | An optional structured identifier for cross-scope addressability (e.g., `cod.review.security`). |
| **Kind** | What type of component this is. One of `token_bundle`, `primitive`, `pattern`, `task`, `workflow`, `skin`, `skill`, `tool`. |
| **Runtime** | An implementation that parses, resolves, and renders components. |
| **Registry** | A distribution point for components — typically a git repository. |

## Goals

1. **Portability.** One format that any LLM-powered system (chat interface, coding agent, internal tool) can consume.
2. **Reusability.** Components can be composed, extended, and shared without copy-paste.
3. **Verifiability.** Every component is inspectable, typeable, and traceable. A consumer can determine what a component does — and what it depends on — before running it.
4. **Trust between agents.** When an agent installs a component from a third party — or hands work to another agent via a shared component — the contract is explicit.

## Non-goals

- **Prescribing prompt style.** The format is agnostic about whether component bodies use XML, markdown, or plain prose.
- **Defining how to test components.** Behavioral testing — verifying that a component produces the outputs it claims — is out of scope. Authors choose their own approach (Promptfoo, OpenAI Evals, Anthropic's tools, custom scripts, manual review). The spec defines what a component *is* and how it composes, not how to verify it works.
- **Replacing Agent Skills.** This spec extends the [Agent Skills](https://agentskills.io) format; it does not replace it. Every Level 0 component is a valid Agent Skills skill.
- **Competing with APM or other package managers.** Distribution is handled by existing tools ([APM](https://microsoft.github.io/apm/), git, Agent Skills registries). This spec defines the format those tools distribute.
- **Solving every prompt problem.** The spec is intentionally small. Orchestration, retrieval, and agent control flow are mostly out of scope.

## Relationship to existing standards

This spec is **additive** to what the community has already converged on:

- **[Agent Skills](https://agentskills.io)** defines the SKILL.md format: YAML front matter with `name`, `description`, and optional `license`, `metadata`, `allowed-tools`, `compatibility`. Our spec uses this format unchanged, placing our additions under `metadata.pbib.*` where the Agent Skills spec explicitly permits arbitrary extension.
- **[APM](https://microsoft.github.io/apm/)** provides dependency resolution and distribution via git. A component published in our format can be installed via APM like any other skill or instruction file. We do not compete with APM; we provide the format it distributes.
- **[MCP](https://modelcontextprotocol.io)** defines how agents connect to tools and data sources. Our `tool` kind references MCP tool schemas rather than redefining them.

The design principle throughout: **nest under `metadata.pbib.*` anything that is pbib-specific; keep flat anything that is a universal concept other tools might use.**

## Compatibility levels

A component is as much of a component as its metadata allows. Three progressive levels:

- **Level 0 — Local.** Just `name` + `description`. Valid SKILL.md. Loadable and renderable. Not composable.
- **Level 1 — Addressable.** Adds `metadata.pbib.version` and `metadata.pbib.kind`. Versionable, publishable, referenceable by other components.
- **Level 2 — Composable.** Adds composition features (`extends`, `inputs`, `slots`, etc.). Full participation in the system.

See [Part 1 §1.2](spec/01-format.md) for details.

## Architecture

The spec is organized into four layers:

| Layer | What it defines | Part |
|-------|----------------|------|
| 1. File format | How a single component is written on disk | [Part 1](spec/01-format.md) |
| 2. Semantic model | Component kinds and what each can do | [Part 2](spec/02-kinds.md) |
| 3. Composition | How components reference, extend, and combine | [Part 3](03-composition.md) |
| 4. Trust and provenance | Versioning, compatibility, deprecation, provenance | [Part 4](04-trust.md) |

[Part 5: Profiles](05-profiles.md) defines conformance levels (core, agent) for runtimes.

Worked examples are in [`examples/`](examples/).

## Reading order

- **If you're evaluating whether to adopt:** README → [`examples/`](examples/) → [Part 5](05-profiles.md).
- **If you're authoring components:** [Part 1](spec/01-format.md) → [Part 2](spec/02-kinds.md) → [`examples/`](examples/) → [Part 3](03-composition.md).
- **If you're implementing a runtime:** all five parts in order, plus [`examples/`](examples/) as test fixtures.
- **If you're reviewing the design:** [Part 2](spec/02-kinds.md) and [Part 3](03-composition.md) are where the load-bearing decisions live.

## One-page summary

Every component is a text file with YAML front matter. At minimum, it declares `name` and `description` (standard Agent Skills fields). When authors want more — versioning, composition, typed inputs — they add fields under `metadata.pbib.*`. Tools that don't know about pbib ignore those fields; our runtime reads them.

A minimal component:

```yaml
---
name: security-review
description: Review code for security vulnerabilities
license: MIT
---
# Body with instructions
```

A fully composable component:

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
    extends: cod.review.base@latest
    inputs:
      severity_threshold:
        type: enum[low, medium, high]
        default: medium
    slot_fills:
      persona: met.p.reviewer-voice
      format: met.p.format-json-findings
      focus: |
        Look for security vulnerabilities at **{{severity_threshold}}+** severity.
tags: [security, code, review]
---
```

Same file. Works in Claude Code, Codex CLI, OpenCode at Level 0. Works in pbib-aware runtimes at Level 2. No conversion, no export step.

## Conformance

A runtime MUST declare which profile it supports: `core` or `agent`. See [Part 5](05-profiles.md).

Where this spec uses the words **MUST**, **MUST NOT**, **SHOULD**, **SHOULD NOT**, and **MAY**, they are to be interpreted as described in [RFC 2119](https://www.rfc-editor.org/rfc/rfc2119).

## Open questions

- **Templating scope.** Whether a minimal template language (conditionals, loops) inside component bodies would pay for itself, or whether composition should remain purely structural.
- **Provenance signing.** Which signing format(s) to standardize on for direct-distribution provenance — Sigstore, SSH signatures, minisign, or another.

Feedback on these is especially welcome.

## Document map

```
README.md              -- this file
01-format.md           -- file format
02-kinds.md            -- component kinds
03-composition.md      -- composition rules
04-trust.md            -- versioning, compatibility, provenance
05-profiles.md         -- conformance profiles
examples/
  01-token.md          -- a token bundle
  02-primitive.md      -- an instruction primitive
  03-pattern.md        -- a reusable pattern with slots
  04-task.md           -- a task extending a pattern
  05-skill.md          -- a skill (agent-profile)
```
