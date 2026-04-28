# Part 4: Trust and provenance

> **Layer 4** of the spec. Defines the mechanisms that let consumers trust components they did not author.

A prompt library people copy-paste from is tactical; a prompt library people *depend on* requires trust. Trust here means: I can tell what a component does before running it, I know it's from who it claims to be from, and I know whether it will keep working when I upgrade.

This part defines four mechanisms:

1. **Versioning** — semver adapted for LLM artifacts (§4.2)
2. **Compatibility metadata** — model and runtime applicability (§4.3)
3. **Deprecation** — graceful evolution (§4.4)
4. **Provenance** — origin verification (§4.5)

Plus the error and trace formats that cut across all layers (§4.6, §4.7).

**Out of scope:** Behavioral testing of components — verifying that a component produces the outputs it claims — is not defined by this spec. Authors choose their own approach (Promptfoo, OpenAI Evals, Anthropic's tools, custom scripts, manual review). The spec defines what a component *is* and how it composes, not how to test it.

## 4.1 Lockfile and distribution

Reproducibility requires pinning exact resolved versions of every component in a dependency graph. Two patterns are supported.

### 4.1.1 APM-managed projects

When a project uses [APM](https://microsoft.github.io/apm/) for distribution, APM's `apm.lock.yaml` is authoritative. It pins every dependency to a 40-character git commit SHA. Pbib runtimes MUST defer to `apm.lock.yaml` for version pinning when it is present.

This is the recommended path. APM handles:

- Git-based dependency resolution
- Transitive dependency traversal
- Content hashing (via commit SHA)
- Supply-chain scanning (`apm audit`)
- Multi-tool deployment (`.claude/`, `.github/`, `.cursor/`, etc.)

Pbib's job when APM is present: read components from the locations APM deployed them, validate their structure, and render them. We do not re-pin what APM has already pinned.

### 4.1.2 Non-APM projects

When a project does not use APM, pbib runtimes MAY maintain a native lockfile:

```yaml
# pbib.lock.yaml
lockfileVersion: 1
components:
  "cod.review.security":
    version: 1.0.0
    integrity: "sha256-abc123..."
    source: "https://github.com/acme/prompts/blob/abc123/review/security.md"
    dependencies:
      "cod.review.base": "1.0.0"
      "met.p.reviewer-voice": "1.0.0"
```

The native lockfile provides the same guarantee APM does: a rendered prompt at time T is reproducible at time T+N as long as the lockfile is unchanged.

Runtimes MUST verify content hashes on every load when a lockfile is present. Mismatches MUST abort rather than warn — silent tampering defeats the purpose.

## 4.2 Versioning

Every Level 1+ component declares a `version` under `metadata.pbib.version` conforming to [Semantic Versioning 2.0.0](https://semver.org). The spec specializes semver for LLM artifacts.

### 4.2.1 What counts as a breaking change

A **major version bump** (X.0.0) is REQUIRED when:

- Removing or renaming an input, slot, or output field.
- Tightening an output schema in a way that rejects previously-valid outputs.
- Widening an input type in a way that breaks downstream consumers (new enum values, looser validation).
- Changing the body in a way that produces materially different model output.
- Removing a tag that was previously relied upon for discovery.

### 4.2.2 What counts as a minor change

A **minor version bump** (1.X.0) is REQUIRED when:

- Adding a new optional input or slot.
- Adding a new enum value to an input.
- Tightening an output schema in a backward-compatible way (adding optional fields).
- Body rewording that preserves output shape and meaning.
- Adding new tokens to a bundle.

### 4.2.3 What counts as a patch

A **patch bump** (1.0.X) is REQUIRED when:

- Fixing typos in body, description, or metadata.
- Updating `description`, `tags`, or other metadata without behavioral impact.

### 4.2.4 Enforcement

These rules are author-disciplined, the same way most package ecosystems work. The spec defines what counts as breaking; authors are responsible for honoring the rules when they publish.

Consumers protect themselves with three mechanisms:

- **Pin exact versions** when reliability matters more than upgrades.
- **Read changelogs** before upgrading.
- **Use lockfiles** (§4.1) so renderings stay reproducible across time.

If an author publishes a body change as a patch when it should have been major, consumers using `^1.0.0` ranges will get unexpected behavior. This is a real risk — npm has the same risk and the ecosystem manages it through community norms, signaling, and consumer caution.

### 4.2.5 Version ranges in dependencies

Components reference dependencies with semver ranges:

- `^1.0.0` — compatible with 1.x.x (most common)
- `~1.2.0` — compatible with 1.2.x
- `1.0.0` — exact version
- `>=1.0.0 <2.0.0` — explicit range

When APM is the distribution tool, these ranges are resolved to specific git SHAs at `apm install` time. In pure pbib resolution, ranges resolve to the highest matching published version.

## 4.3 Compatibility metadata

Components declare which models and runtimes they work with under `metadata.pbib.runtime_compatibility`. This is informational but critical — applying a Claude-tuned component to a small open model often produces degraded results.

Note: the Agent Skills spec has its own top-level `compatibility` field (a free-form string, max 500 chars). Our structured compatibility lives under `metadata.pbib.runtime_compatibility` to avoid collision.

```yaml
metadata:
  pbib:
    runtime_compatibility:
      tested_models:
        - claude-opus-4-7
        - claude-sonnet-4-6
      compatible_models:
        - "claude-*"
        - "gpt-4*"
      incompatible_models:
        - "gpt-3.5-*"
      min_context: 8000
      style: xml_structured
      profile: core
```

### 4.3.1 Fields

| Field                 | Description                                                                                        |
|-----------------------|----------------------------------------------------------------------------------------------------|
| `tested_models`       | Models the author has actually exercised the component against. Factual claim by the author.       |
| `compatible_models`   | Models the author believes the component works with. Glob patterns permitted.                      |
| `incompatible_models` | Models the author has determined the component does not work with.                                 |
| `min_context`         | Minimum model context window size, in tokens.                                                      |
| `style`               | Informational. Values: `xml_structured`, `markdown_sections`, `plain_prose`, or any custom string. |
| `profile`             | `core` or `agent`. See [Part 5](05-profiles.md).                                                   |

### 4.3.2 Resolution semantics

Runtimes SHOULD check compatibility at resolution time and warn when:

- The target model is in `incompatible_models`.
- The target model is not in `compatible_models` (with globs evaluated).
- The target model's context window is less than `min_context`.

Runtimes MUST NOT refuse to execute on compatibility mismatches — compatibility is advisory. The decision is the consumer's.

### 4.3.3 Compatibility under extension

When a component extends another, the effective compatibility is:

- `compatible_models`: intersection of parent and child.
- `tested_models`: child's set only.
- `min_context`: maximum of parent and child.

A child cannot widen compatibility beyond what its parent supports.

## 4.4 Deprecation

Components are deprecated before removal. A deprecated component continues to resolve but emits a warning at resolution time.

```yaml
metadata:
  pbib:
    deprecated:
      since: 1.5.0
      reason: "Superseded by cod.review.security-v2 with structured CWE output"
      replacement: cod.review.security-v2@^1.0.0
      removal_planned: 2.0.0
```

### 4.4.1 Deprecation rules

- Deprecation MUST NOT happen in a patch release.
- A component SHOULD remain resolvable for at least one major version after deprecation.
- Removal happens by publishing no new versions and eventually unpublishing old versions after a registry-defined deprecation period.

### 4.4.2 Replacement semantics

When `replacement` is specified, automated migration tools MAY suggest the replacement to consumers. The spec does not mandate automatic substitution.

## 4.5 Provenance

Provenance is the chain of evidence connecting a component to its author. Without provenance, a component's claims (author identity, tested models) are assertions that cannot be verified.

### 4.5.1 APM-based provenance

When distributed via APM, provenance is handled by git itself: the commit SHA is a cryptographic hash of the content, signed git tags provide author verification, and `apm audit` scans for content-level threats (hidden Unicode, prompt injection).

For most projects, APM's provenance model is sufficient and pbib needs to add nothing.

### 4.5.2 Direct-distribution provenance

For components distributed outside APM (downloaded directly, installed from custom registries), additional signing MAY be used:

- [Sigstore](https://www.sigstore.dev/) for public registries.
- SSH signatures for git-based registries.
- Minisign for minimal-infrastructure deployments.

The spec does not commit to a single mechanism. This is an **open question** for the community. See [README](../README.md).

### 4.5.3 Supply-chain constraints

Because components can include other components transitively, a malicious dependency can inject content into the final rendered prompt. Defenses:

- **Lockfile hashes** (APM's `apm.lock.yaml` or pbib's native lockfile) prevent silent mutation post-install.
- **Scoped handles** prevent typosquatting to a degree.
- **Explicit transitive visibility** — runtimes SHOULD surface the full set of components that will be rendered, not just direct dependencies.
- **APM audit integration** — when APM is present, pbib runtimes SHOULD surface `apm audit` findings alongside their own validation errors.

## 4.6 Error format

Errors emitted by any spec-conformant runtime MUST follow this structure:

```yaml
error:
  code: VALIDATION_FAILED
  message: "Input 'language' has invalid value 'java'"
  component: cod.review.security
  version: 1.0.0
  path: metadata.pbib.inputs.language
  details:
    received: java
    allowed: [python, typescript, go, rust]
  trace_id: ...
```

### 4.6.1 Error code prefixes

| Prefix          | Domain                                           |
|-----------------|--------------------------------------------------|
| `FORMAT_*`      | File format errors (Part 1)                      |
| `KIND_*`        | Component kind violations (Part 2)               |
| `COMPOSITION_*` | Composition rule violations (Part 3)             |
| `VALIDATION_*`  | Input/output/slot validation                     |
| `RESOLUTION_*`  | Dependency resolution                            |
| `INTEGRITY_*`   | Hash, signature, or provenance failures          |
| `RUNTIME_*`     | Execution-time failures (model errors, timeouts) |

Specific codes within each prefix are implementation-defined.

## 4.7 Trace format

Runtimes SHOULD emit a structured trace for every component invocation:

```yaml
trace:
  trace_id: 01HM3K8XQZR7...
  timestamp: 2026-04-24T14:33:12Z
  component: cod.review.security
  version: 1.0.0
  resolved_tree:
    - cod.review.security@1.0.0 (sha256-abc...)
    - cod.review.base@1.0.0 (sha256-def...)
    - met.p.reviewer-voice@1.0.0 (sha256-ghi...)
    - met.p.format-json-findings@1.0.0 (sha256-jkl...)
    - met.tone@1.0.0 (sha256-mno...)
  inputs_hash: sha256-pqr...
  model: claude-opus-4-7
  rendered_length_tokens: 2347
  latency_ms: 1823
  output_validation: passed
```

Traces enable drift tracking, debugging across versions, and post-hoc investigation. When something goes wrong, the trace shows exactly which components were resolved, at what versions, with what inputs.

## 4.8 Summary

Trust in this system is the product of several mechanisms working together:

- **Semver** means version numbers carry meaning, even if author-disciplined.
- **Lockfiles (APM or native) + content hashes** mean resolution is reproducible and tamper-evident.
- **Compatibility metadata** means consumers can predict whether a component will work for them.
- **Provenance** (via APM's git-based model or optional signing) means authorship claims are verifiable.
- **Traces** mean runtime behavior is auditable after the fact.

This is the trust model npm, pip, and most package ecosystems ship with. It's not perfect — it depends on author discipline and consumer caution — but it's what the field has accepted as a working baseline.

For components where stronger guarantees matter (security-critical prompts, regulated industries), authors are encouraged to add their own behavioral testing using whatever framework fits their needs. The spec doesn't prescribe how; it just gets out of the way.
