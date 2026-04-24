# Part 5: Profiles

> **Layer 5** of the spec. Defines conformance levels for runtimes and components.

The spec supports use cases ranging from static prompt libraries to dynamic coding agents. These have different performance, context, and capability requirements. To avoid a single bloated standard, the spec defines two conformance profiles:

- **Core profile** — static composition, offline resolution.
- **Agent profile** — strict superset of core, adds dynamic context, tools, and skills.

Components declare their target profile via `metadata.pbib.runtime_compatibility.profile`. Runtimes declare which profiles they support. A runtime MUST NOT execute a component targeting a profile it does not implement.

## 5.1 Core profile

### 5.1.1 Scope

Core profile covers:

- Applications with bounded prompt needs.
- Prompt libraries for teams and organizations.
- Chat interfaces with curated prompt catalogs.
- Batch-processing systems and pipelines.

### 5.1.2 Required kinds

A core-profile runtime MUST support:

- `token_bundle` ([Part 2 §2.4](02-kinds.md))
- `primitive` (§2.5)
- `pattern` (§2.6)
- `task` (§2.7)
- `workflow` (§2.8)
- `skin` (§2.9)

### 5.1.3 Required mechanisms

A core-profile runtime MUST implement:

- Full file format parsing ([Part 1](01-format.md)).
- All composition mechanisms ([Part 3 §§3.2–3.7](03-composition.md)).
- The resolution algorithm (§3.8).
- Input, slot, and output validation.
- Lockfile verification (either APM's `apm.lock.yaml` or pbib's native lockfile) ([Part 4 §4.1](04-trust.md)).
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

## 5.2 Agent profile

### 5.2.1 Scope

Agent profile covers:

- Coding agents (Claude Code, Cursor, continue.dev, Codex CLI, internal agent platforms).
- Research agents with tool access.
- Multi-turn systems with evolving context.
- Any consumer that composes prompts with runtime-gathered context.

### 5.2.2 Relationship to core

Agent profile is a **strict superset** of core. An agent-profile runtime MUST implement everything in §5.1, plus the additions below. A core-profile component MUST execute unchanged on an agent-profile runtime.

The reverse does not hold: an agent-profile component MAY fail on a core-profile runtime, which is why components declare their target profile.

### 5.2.3 Additional kinds

An agent-profile runtime MUST support:

- `skill` ([Part 2 §2.10](02-kinds.md))
- `tool` (§2.11)

### 5.2.4 Additional input types

An agent-profile runtime MUST support the `context_window` input type:

```yaml
metadata:
  pbib:
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
| `truncation` | Strategy: `head`, `tail`, or `smart` (implementation-defined). |
| `streaming` | Boolean. Whether the input arrives incrementally. Default `false`. |

When multiple `context_window` inputs compete for budget, the runtime MUST truncate in ascending priority order. Ties are broken by declaration order.

### 5.2.5 Tool attachment

An agent-profile runtime MUST resolve the `tools:` field of skills and tasks:

```yaml
metadata:
  pbib:
    tools:
      - met.tool.web-search@^1.0.0
      - met.tool.file-io@^1.0.0
```

Tool resolution makes the tool schemas available to the model alongside the component's prompt text. Implementation details are runtime-specific.

When a component also declares Agent Skills' top-level `allowed-tools`, the runtime MUST intersect the two lists: a tool must appear in both `allowed-tools` (permission) and `metadata.pbib.tools` (availability) to be usable.

### 5.2.6 Skill triggers

An agent-profile runtime MUST match skill triggers against runtime state (user message, file types, keywords) and load matching skills on demand:

```yaml
metadata:
  pbib:
    triggers:
      - keywords: [word, docx, document]
      - file_types: [.docx]
      - context_match: "user is asking to create a file"
```

Trigger matching is runtime-specific. The spec defines only the declaration format.

### 5.2.7 Caching requirements

An agent-profile runtime MUST implement caching of resolution algorithm steps 1–6 ([Part 3 §3.8](03-composition.md)). Agents invoke components at high frequency; full resolution per invocation is not viable.

The cache MUST be keyed by `(handle, version, input_fills_hash, slot_fills_hash)` at minimum. Cache invalidation MUST respect lockfile changes.

## 5.3 Declaring profile

### 5.3.1 In components

```yaml
metadata:
  pbib:
    runtime_compatibility:
      profile: core    # or: agent
```

If omitted, the runtime MUST infer from the component's `kind`:

- `skill` or `tool` → agent profile.
- Components using `context_window` inputs → agent profile.
- All others → core profile.

Explicit declaration is RECOMMENDED.

### 5.3.2 In runtimes

A runtime documents its conformance by declaring supported profiles. Format is convention-only; specific runtimes may use their own metadata shapes.

## 5.4 Profile compatibility summary

| Runtime supports | Can execute components targeting |
|------------------|----------------------------------|
| core only | core only |
| agent | core, agent |

Core profile is the stable floor. Agent profile is today's primary extension. Future profiles MUST NOT break core-profile semantics for components targeting core.

## 5.5 Open question

Whether agent profile should become its own specification document once the feature set stabilizes. The case for splitting: agent-profile concerns (truncation strategies, caching semantics, trigger matching) are substantial enough to warrant separate treatment and evolution. The case against: a single spec reduces fragmentation and keeps the profile distinction concrete.

The current plan is to keep both profiles in this spec through 1.0, then reconsider based on implementation experience.
