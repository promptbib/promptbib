# Part 1: File format

> **Layer 1** of the spec. Defines how a single component is written on disk.

A component is a UTF-8 encoded text file with YAML front matter and an optional body. The format extends the [Agent Skills](https://agentskills.io) SKILL.md format: every component is a valid Agent Skills file, and every valid Agent Skills file is a Level 0 component.

## 1.1 Compatibility with Agent Skills

The Agent Skills spec defines the file format this spec builds on. Specifically:

- Required top-level fields: `name`, `description`.
- Permitted optional top-level fields: `license`, `metadata` (an arbitrary key-value mapping), `allowed-tools`, `compatibility` (a string, max 500 chars).

Our spec adds no new top-level fields. All pbib-specific extensions live under **`metadata.pbib.*`**, which the Agent Skills spec explicitly permits.

**Design rule:** A field goes under `metadata.pbib.*` if its semantics are defined and enforced by the pbib spec. A field stays flat if it represents a universal concept other tools might use.

| Field | Where | Why |
|-------|-------|-----|
| `name`, `description`, `license` | Top-level | Agent Skills standard |
| `allowed-tools`, `compatibility` | Top-level | Agent Skills standard |
| `tags` | Top-level | Universal convention |
| `metadata.author` | Flat under metadata | Universal, community convention |
| `metadata.pbib.version` | Under pbib | Our semver rules govern it |
| `metadata.pbib.kind` | Under pbib | Our kind vocabulary |
| `metadata.pbib.handle` | Under pbib | Our domain/composition system |
| `metadata.pbib.extends` | Under pbib | Pbib composition only |
| `metadata.pbib.inputs`, `slots`, `outputs` | Under pbib | Pbib type system |
| `metadata.pbib.runtime_compatibility` | Under pbib | Distinct from Agent Skills' string `compatibility` |

This nesting is what makes files work unchanged in Claude Code, Codex CLI, Cursor, OpenCode, and other Agent Skills consumers — they read only the standard fields and ignore our extensions.

## 1.2 Compatibility levels

A component is **as much of a component as its metadata allows**. Three progressive levels:

### Level 0 — Local

A file with minimal metadata. Valid SKILL.md. Works today in every Agent Skills consumer.

Required: `name`.

```yaml
---
name: security-review
description: Review code for security vulnerabilities
license: MIT
---
# Body content
```

**Enables:** Loading, rendering, use in existing tools.
**Does not enable:** Referencing by other components, versioning, lockfile entries.

### Level 1 — Addressable

Level 0 plus metadata that makes the component uniquely identified and versionable.

Additionally required: `metadata.pbib.version`, `metadata.pbib.kind`.

```yaml
---
name: security-review
description: Review code for security vulnerabilities
license: MIT
metadata:
  pbib:
    version: 1.0.0
    kind: skill
---
# Body unchanged
```

**Enables:** `extends` targeting, `{{include: ...}}` references, lockfile pinning, publishing.

A Level 1 component remains fully compatible with Level 0 consumers — they read `name` and `description` and ignore everything under `metadata.pbib.*`.

### Level 2 — Composable

Level 1 plus composition features: typed inputs, slot fills, output schemas.

Additional fields depend on which features are used. See [Part 3](03-composition.md).

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

**Enables:** Runtime input validation, composition, output schema enforcement.

### Capability matrix

| Capability                                    | L0 | L1 | L2 |
|-----------------------------------------------|----|----|----|
| Load and render                               | ✓  | ✓  | ✓  |
| Works in Claude Code, Codex, Cursor, OpenCode | ✓  | ✓  | ✓  |
| Versionable                                   | —  | ✓  | ✓  |
| Referenceable by other components             | —  | ✓  | ✓  |
| Validated inputs                              | —  | —  | ✓  |
| Composition (extends, slots, includes)        | —  | —  | ✓  |
| Output schema enforcement                     | —  | —  | ✓  |

Authors SHOULD start at whichever level fits their current need and move up only when features become useful.

## 1.3 File extension

A component file MAY use any of:

- **`SKILL.md`** — canonical filename for skill components. Default kind: `skill`.
- **`.prompt.md`** — canonical extension for other component kinds. Kind MUST be explicit in front matter.
- **`.prompt.yaml`** — manifest-only form. Body, if present, is a YAML string field.

Runtimes MUST support `SKILL.md` and `.prompt.md`. Support for `.prompt.yaml` is OPTIONAL.

## 1.4 File structure

```
---
<YAML front matter>
---
<body, optional>
```

### 1.4.1 Front matter

Front matter MUST be valid YAML. The top-level value MUST be a mapping.

### 1.4.2 Body

The body is everything after the closing `---` up to end of file. The body is not parsed by the spec; interpretation is defined by the component's kind and processed at render time (see [Part 3](03-composition.md)).

A component MAY have an empty body. This is common for:

- Components using `extends` and only filling slots.
- Token bundles (values live in front matter).
- Workflows (structural, no prose).

### 1.4.3 External body via `body_source`

A component MAY declare its body is sourced from another file:

```yaml
---
name: docx
description: Create and edit Word documents
license: MIT
metadata:
  pbib:
    version: 1.0.0
    kind: skill
    body_source: ./SKILL-full.md
---
```

When `body_source` is present under `metadata.pbib`, the file's own body MUST be empty. The runtime reads the referenced file's body content as this component's body at render time.

Use cases:

- Wrapping an existing SKILL.md in component metadata without duplicating content.
- Sharing the same body across multiple components with different metadata.
- Adopting third-party skills into your own component graph without forking them.

### 1.4.4 Encoding

Files MUST be UTF-8. No BOM. Line endings MAY be LF or CRLF; runtimes MUST accept both.

## 1.5 Front matter fields

### 1.5.1 Top-level (Agent Skills standard)

| Field           | Level           | Description                                                                                                                   |
|-----------------|-----------------|-------------------------------------------------------------------------------------------------------------------------------|
| `name`          | L0+ required    | Short identifier, hyphen-case, lowercase letters/digits/hyphens, max 64 chars.                                                |
| `description`   | L0+ recommended | Short human-readable summary. Max 1024 characters per Agent Skills spec.                                                      |
| `license`       | Optional        | License name (e.g., `MIT`) or path to license file.                                                                           |
| `allowed-tools` | Optional        | Space-delimited list of pre-approved tools (experimental, Agent Skills field).                                                |
| `compatibility` | Optional        | Environment requirements, free-form string, max 500 chars (Agent Skills field). Distinct from pbib's `runtime_compatibility`. |
| `tags`          | Optional        | List of strings for categorization.                                                                                           |

### 1.5.2 Under `metadata` (flat)

| Field | Level | Description |
|-------|-------|-------------|
| `metadata.author` | Optional | Author name or organization. Universal convention. |
| `metadata.pbib` | L1+ | Pbib-specific extension block. See §1.5.3. |

### 1.5.3 Under `metadata.pbib.*`

| Field                   | Level                   | Description                                                                    |
|-------------------------|-------------------------|--------------------------------------------------------------------------------|
| `version`               | L1+ required            | Semantic version of this component. See [Part 4 §4.2](04-trust.md).            |
| `kind`                  | L1+ required            | Component kind. See [Part 2](02-kinds.md).                                     |
| `handle`                | L1 optional             | Structured identifier for domain placement and composition. See §1.6.          |
| `body_source`           | Optional                | Path or handle of a file whose body this component uses (§1.4.3).              |
| `deprecated`            | Optional                | Deprecation notice. See [Part 4 §4.4](04-trust.md).                            |
| `runtime_compatibility` | Optional                | Pbib-specific model and runtime compatibility. See [Part 4 §4.3](04-trust.md). |
| `inputs`                | L2 when used            | Runtime input declarations. See [Part 3 §3.2](03-composition.md).              |
| `slots`                 | L2 when used            | Authoring-time extension points. See [Part 3 §3.3](03-composition.md).         |
| `outputs`               | L2 when used            | Output schema. See [Part 3 §3.5](03-composition.md).                           |
| `extends`               | L2 when used            | Handle or name of parent component. See [Part 3 §3.6](03-composition.md).      |
| `slot_fills`            | L2 when used            | Values for parent's slots.                                                     |
| `input_fills`           | L2 when used            | Values for nested components' inputs.                                          |
| `triggers`              | Optional (skill)        | Trigger declarations for the skill kind.                                       |
| `tools`                 | Optional (skill)        | Tool components the skill requires.                                            |
| `steps`                 | Required (workflow)     | Workflow step declarations. See [Part 2 §2.7](02-kinds.md).                    |
| `values`                | Required (token_bundle) | Token value dictionary. See [Part 2 §2.3](02-kinds.md).                        |
| `schema`                | Required (tool)         | Tool input/output schema. See [Part 2 §2.10](02-kinds.md).                     |
| `overrides`             | Required (skin)         | Skin override declarations. See [Part 2 §2.8](02-kinds.md).                    |

## 1.6 Name and handle

### 1.6.1 `name` is the primary identifier

Every component has a `name` — it's the Agent Skills required field. `name` uniquely identifies the component within the scope that contains it (the git repo, directory, or registry).

`name` grammar (inherited from Agent Skills):

- Lowercase letters, digits, hyphens only
- Max 64 characters
- Matches the containing directory name when present

### 1.6.2 `handle` is optional structured metadata

A component MAY declare a `handle` under `metadata.pbib.handle` for components participating in the domain taxonomy:

```yaml
metadata:
  pbib:
    handle: cod.review.security
```

The handle is what enables the "periodic table" organization — domain code (`cod`), family (`review`), variant (`security`). It's useful for:

- Cross-registry discovery ("find all security-review components").
- Registry indexing by domain.
- Disambiguating when multiple registries contain similarly-named components.

For small teams and internal libraries, `name` alone is sufficient. For public registries and cross-organization sharing, handles provide structure.

### 1.6.3 Handle grammar

```
handle         = domain "." family *("." segment)
domain         = 3*4 ALPHA                       ; 3-4 lowercase letters
family         = segment
segment        = ALPHA *(ALPHA / DIGIT / "-")    ; lowercase, kebab-case
```

When both `name` and `handle` are present, `name` SHOULD be the last segment of `handle`. Runtimes MAY warn on divergence but MUST NOT reject.

Three-letter domains are reserved for a canonical table (governance TBD). Four-letter prefixes are available for project-local or vendor-specific namespacing (e.g., `acme`).

### 1.6.4 Scoped handles

A handle MAY be scoped with a leading `@org/` prefix:

```
@anthropic/cod.review.security
@acme/biz.proposal.draft
```

Scopes disambiguate authors publishing in overlapping domains.

## 1.7 Variable interpolation

The body MAY contain variable references. Four forms:

| Form                          | Meaning                         | Resolved at |
|-------------------------------|---------------------------------|-------------|
| `{{input_name}}`              | A runtime input                 | render time |
| `{{slot: slot_name}}`         | A slot fill                     | load time   |
| `{{token: bundle.key}}`       | A token value                   | load time   |
| `{{include: name-or-handle}}` | Inline another component's body | load time   |

See [Part 3 §3.8](03-composition.md) for resolution semantics.

## 1.8 Referencing components

When a component references another (in `extends`, `slot_fills`, `input_fills`, `{{include: ...}}`, or workflow `steps`), the reference may take three forms:

| Form                    | Example                             | Meaning                                |
|-------------------------|-------------------------------------|----------------------------------------|
| Plain name              | `reviewer-voice`                    | Within the same scope (repo/registry). |
| Handle                  | `met.p.reviewer-voice`              | Explicit domain placement.             |
| Handle with version pin | `met.p.reviewer-voice@1.0.0`        | Exact version.                         |
| Handle with range       | `met.p.reviewer-voice@^1.0.0`       | Semver range.                          |
| Scoped handle           | `@anthropic/met.p.reviewer-voice`   | Cross-scope reference.                 |
| Git path                | `github/org/repo/path/to/component` | APM-style git reference.               |

Runtimes MUST resolve plain names against the current scope first. Handles are resolved through the registry. Git paths are resolved via the distribution tool (APM or git clone).

Version pinning defaults: if omitted, runtimes SHOULD resolve to the latest compatible version at lockfile generation time and pin it exactly. Lockfiles pin specific versions or git SHAs; the reference in the component file expresses intent.

## 1.9 Comments

YAML comments (`#`) in front matter are permitted and ignored.

The body has no comment syntax defined by this spec. Authors needing annotations SHOULD use their body format's conventions (e.g., HTML comments for markdown).

## 1.10 Examples

### Level 0 — plain SKILL.md

```markdown
---
name: docx
description: Create and edit Microsoft Word documents
license: MIT
---
# DOCX skill

Handle requests involving .docx files as follows...
```

### Level 1 — addressable

```markdown
---
name: docx
description: Create and edit Microsoft Word documents
license: MIT
metadata:
  pbib:
    version: 1.0.0
    kind: skill
tags: [docx, documents, office]
---
# Body unchanged from Level 0
```

### Level 2 — composable

```markdown
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
tags: [security, code, review]
---
```

See [`examples/`](examples/) for fully-worked examples of every component kind.

## 1.11 Validation

A runtime MUST validate:

1. File is valid UTF-8.
2. Front matter is present, delimited, and valid YAML with a top-level mapping.
3. `name` is present and conforms to the name grammar (§1.6.1).
4. If `description` exceeds 1024 characters, warn.
5. If `metadata.pbib.handle` is present, it conforms to the handle grammar (§1.6.3).
6. If `metadata.pbib.version` is present, it conforms to semantic versioning ([Part 4 §4.2](04-trust.md)).
7. If `metadata.pbib.kind` is present, it is a value defined in [Part 2](02-kinds.md).
8. If `metadata.pbib.body_source` is present, the file's own body is empty and the referenced file is resolvable.
9. For Level 1: both `metadata.pbib.version` and `metadata.pbib.kind` are present.
10. For Level 2: additional required fields for whichever composition features are used (validated per [Part 3](03-composition.md)).

Validation failures MUST produce structured errors naming the file, the field, and the violation. See [Part 4 §4.6](04-trust.md) for the error format.
