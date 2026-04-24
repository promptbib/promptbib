# Part 5: Profiles

> **Layer 5** of the spec. Defines conformance levels for runtimes and components.

The spec supports use cases ranging from static prompt libraries to dynamic coding agents. These have different performance, context, and capability requirements. To avoid a single bloated standard, the spec defines two conformance profiles:

- **Core profile** — static composition, offline resolution, file-based registries.
- **Agent profile** — strict superset of core, adding dynamic context, streaming inputs, tools, and skills.

Every component declares which profile it targets via `compatibility.runtime_profile`. Every runtime declares which profiles it supports. A runtime MUST NOT execute a component targeting a profile it does not implement.

## 5.1 Core profile

### 5.1.1 Scope

Core profile covers:

- Applications with known, bounded prompt needs.
- Prompt libraries for teams and organizations.
- Chat interfaces with curated prompt catalogs.
- Batch-processing systems and pipelines.

### 5.1.2 Required kinds

A core-profile runtime MUST support:

- `token_bundle` (Part 2 §2.3)
- `primitive` (§2.4)
- `pattern` (§2.5)
- `task` (§2.6)
- `workflow` (§2.7)
- `skin` (§2.8)

### 5.1.3 Required mechanisms

A core-profile runtime MUST implement:

- Full file format parsing (Part 1).
- All composition mechanisms (Part 3 §§3.2–3.7).
- The resolution algorithm (§3.8).
- Input, slot, and output validation (§3.2.4, §3.3, §3.5.3).
- Lockfile production and verification (Part 4 §4.1).
- Semver resolution (§4.2.5).
- Error format (§4.7).

A core-profile runtime SHOULD implement:

- Compatibility checking (§4.3.2).
- Trace emission (§4.8).
- Eval execution when resolving (§4.4).

### 5.1.4 Permitted omissions

A core-profile runtime MAY omit:

- Dynamic context truncation (agent profile).
- Tool schema handling.
- Skill trigger matching.
- Streaming input support.
- Runtime cache invalidation policies beyond lockfile hash checks.

## 5.2 Agent profile

### 5.2.1 Scope

Agent profile covers:

- Coding agents (Claude Code, Cursor, continue.dev, internal agent platforms).
- Research agents with tool access.
- Multi-turn systems with evolving context.
- Any consumer that composes prompts with runtime-gathered context.

### 5.2.2 Relationship to core

Agent profile is a **strict superset** of core. An agent-profile runtime MUST implement everything in §5.1, plus the additions below. A core-profile component MUST execute unchanged on an agent-profile runtime.

The reverse does not hold: an agent-profile component MAY fail on a core-profile runtime, which is why components declare their target profile.

### 5.2.3 Additional kinds

An agent-profile runtime MUST support:

- `skill` (§2.9)
- `tool` (§2.10)

### 5.2.4 Additional input types

An agent-profile runtime MUST support the `context_window` input type:

```yaml
inputs:
  codebase:
    type: context_window
    priority: high
    max_tokens: 20000
    truncation: smart
    required: false
```

#### 5.2.4.1 `context_window` specification

| Field | Description |
|-------|-------------|
| `priority` | `low`, `medium`, `high`. Guides truncation order under context pressure. |
| `max_tokens` | Soft limit for this input. Runtime may truncate to fit. |
| `truncation` | Strategy: `head` (drop from beginning), `tail` (drop from end), `smart` (implementation-defined, e.g., drop middle). |
| `streaming` | Boolean. Whether the input arrives incrementally. Default `false`. |

When multiple `context_window` inputs compete for budget, the runtime MUST truncate in ascending priority order. Ties are broken by declaration order.

### 5.2.5 Tool attachment

An agent-profile runtime MUST resolve the `tools:` field of skills and tasks:

```yaml
tools:
  - met.tool.web-search@^1.0.0
  - met.tool.file-io@^1.0.0
```

Tool resolution makes the tool schemas available to the model alongside the component's prompt text. Implementation details (how the tool is advertised to the model, how invocations are handled) are runtime-specific.

### 5.2.6 Skill triggers

An agent-profile runtime MUST match skill triggers against runtime state (user message, file types present, keywords, etc.) and load matching skills on demand:

```yaml
triggers:
  - keywords: [word, docx, document]
  - file_types: [.docx]
  - context_match: "user is asking to create a file"
```

Trigger matching is runtime-specific. The spec defines only the declaration format; different runtimes MAY implement different matching strategies.

### 5.2.7 Caching requirements

An agent-profile runtime MUST implement caching of the resolution algorithm's steps 1–6 (Part 3 §3.8). Agents invoke components at high frequency; full resolution per invocation is not viable.

The cache MUST be keyed by `(handle, version, input_fills_hash, slot_fills_hash)` at minimum. Cache invalidation MUST respect lockfile changes.

## 5.3 Declaring profile

### 5.3.1 In components

```yaml
compatibility:
  runtime_profile: core
```

or

```yaml
compatibility:
  runtime_profile: agent
```

If omitted, the runtime MUST infer from the component's declared `kind`:

- `skill` or `tool` → agent profile.
- Components using `context_window` inputs → agent profile.
- All others → core profile.

Explicit declaration is RECOMMENDED.

### 5.3.2 In runtimes

A runtime documents its conformance by declaring supported profiles:

```yaml
# Runtime capability advertisement
runtime: promptbib-js
version: 0.3.0
supports_profiles: [core]
spec_versions: [0.1.0]
```

This is a convention for tooling (package managers, registries) to filter components by target runtime. The format itself is informational.

## 5.4 Profile evolution

As the spec matures, additional profiles may be defined. Anticipated profiles, not yet specified:

- **Minimal profile** — file format only, no composition. For the smallest possible embedding, like static site generators.
- **Distributed profile** — adds multi-agent coordination, shared state, inter-agent handoff contracts. For agent-to-agent systems.

These are speculative and are not part of this spec version.

## 5.5 Profile compatibility summary

| Runtime supports | Can execute components targeting |
|------------------|----------------------------------|
| core only | core only |
| agent | core, agent |
| core + minimal (future) | core, minimal |

The core profile is the stable floor. Agent profile is today's primary extension. Future profiles MUST NOT break core-profile semantics for components that target core.

## 5.6 Open question

Whether agent profile should become its own specification document once the feature set stabilizes. The case for splitting: agent-profile concerns (truncation strategies, caching semantics, trigger matching) are substantial enough to warrant separate treatment and evolution. The case against: a single spec reduces fragmentation and keeps the profile distinction concrete.

The current plan is to keep both profiles in this spec through 1.0, then reconsider based on implementation experience.
