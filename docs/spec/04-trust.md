# Part 4: Trust and verification

> **Layer 4** of the spec. Defines the mechanisms that let consumers trust components they did not author.

A prompt library people copy-paste from is tactical; a prompt library people *depend on* requires trust. Trust means: I can tell what a component does before running it, I can verify it performs as claimed, I know it is from who it claims to be from, and I know whether it will keep working when I upgrade.

This part defines five mechanisms:

1. **Versioning** — semver adapted for LLM artifacts (§4.2)
2. **Compatibility metadata** — model and runtime applicability (§4.3)
3. **Evals** — behavioral tests attached to components (§4.4)
4. **Deprecation** — graceful evolution (§4.5)
5. **Provenance** — signing and origin verification (§4.6)

Plus the error format and trace format that cut across all layers (§4.7, §4.8).

## 4.1 The lockfile

Before the mechanisms: a foundational concept. Any project that consumes components produces a **lockfile** that pins exact resolved versions and content hashes of every component in its dependency graph.

```yaml
# promptbib.lock
lockfileVersion: 1
components:
  "cod.review.security":
    version: 1.0.0
    integrity: "sha256-abc123..."
    resolved: "https://promptbib.org/cod.review.security/1.0.0"
    dependencies:
      "cod.review.base": "1.0.0"
      "met.p.reviewer-voice": "1.0.0"
      "met.p.format-json-findings": "1.0.0"
  "cod.review.base":
    version: 1.0.0
    integrity: "sha256-def456..."
    ...
```

The lockfile guarantees reproducibility: a rendered prompt at time T is reproducible at time T+N as long as the lockfile is unchanged. This is the same guarantee `package-lock.json` provides for npm.

Runtimes MUST produce a lockfile when resolving dependencies. Runtimes MUST verify content hashes on every load. Mismatches MUST abort rather than warn — silent tampering defeats the purpose.

## 4.2 Versioning

Every component declares a `version` conforming to [Semantic Versioning 2.0.0](https://semver.org). The spec specializes semver for LLM artifacts.

### 4.2.1 What counts as a breaking change

A **major version bump** (X.0.0) is REQUIRED when:

- Removing or renaming an input, slot, or output field.
- Narrowing an output schema (making it reject previously-valid outputs).
- Widening an input type in a way that breaks downstream consumers (new enum values, looser validation).
- Changing the body in a way that produces observably different model output on the component's eval suite.
- Removing a tag that was previously relied upon for discovery.

### 4.2.2 What counts as a minor change

A **minor version bump** (1.X.0) is REQUIRED when:

- Adding a new optional input or slot.
- Adding a new enum value to an input.
- Tightening an output schema in a backward-compatible way (adding optional fields).
- Body rewording that preserves output shape and passes the eval suite.
- Adding new tokens to a bundle.

### 4.2.3 What counts as a patch

A **patch bump** (1.0.X) is REQUIRED when:

- Fixing typos in body, description, or metadata.
- Updating `description`, `tags`, or other metadata without behavioral impact.
- Improving or adding evals without changing component behavior.

### 4.2.4 The eval backstop

Semver for prompts is only meaningful when tied to behavioral tests, because the text itself cannot reveal whether a change is breaking. The spec requires:

**Any minor or patch release MUST pass all evals the previous version passed.** If an eval fails, the change is major by definition.

This is the key rule that turns version numbers from aspiration into enforcement. Runtimes and CI systems MUST verify this before publishing a new version.

### 4.2.5 Version ranges in dependencies

Components reference dependencies with semver ranges:

- `^1.0.0` — compatible with 1.x.x (most common)
- `~1.2.0` — compatible with 1.2.x
- `1.0.0` — exact version
- `>=1.0.0 <2.0.0` — explicit range

Runtimes MUST resolve to the highest version satisfying the constraint. The lockfile pins the resolved version.

## 4.3 Compatibility metadata

Components declare which models and runtimes they work with. This is informational but critical — applying a Claude-tuned component to a small open model often produces degraded results, and consumers deserve to know.

```yaml
compatibility:
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
  runtime_profile: core
```

### 4.3.1 Fields

| Field | Description |
|-------|-------------|
| `tested_models` | Models the author has actually run evals against. Factual claim. |
| `compatible_models` | Models the author believes the component works with. Glob patterns permitted. |
| `incompatible_models` | Models the author has determined the component does not work with. |
| `min_context` | Minimum model context window size, in tokens. |
| `style` | Informational. Describes the prompt body style. Values: `xml_structured`, `markdown_sections`, `plain_prose`, or any custom string. |
| `runtime_profile` | `core` or `agent`. See [Part 5](05-profiles.md). |

### 4.3.2 Resolution semantics

Runtimes SHOULD check compatibility at resolution time and warn when:

- The target model is in `incompatible_models`.
- The target model is not in `compatible_models` (with globs evaluated).
- The target model's context window is less than `min_context`.

Runtimes MUST NOT refuse to execute on compatibility mismatches — compatibility is advisory. The decision is the consumer's.

### 4.3.3 Compatibility under extension

When a component extends another, the effective compatibility is the intersection of parent and child `compatible_models`, and the maximum of `min_context` values. A child cannot widen compatibility beyond what its parent supports.

## 4.4 Evals

An eval is a reproducible test of a component's behavior. Evals are what make versioning enforceable and what let consumers verify claims.

### 4.4.1 Eval attachment

A component MAY declare evals in its front matter:

```yaml
evals:
  - suite: cod.review.security.evals@^1.0.0
    profile: standard
  - file: ./evals/security-review.yaml
```

Evals MAY be:

- **Referenced by handle** — another component of kind `eval_suite` (a proposed future kind).
- **Referenced by file path** — a file co-located with the component.

### 4.4.2 Eval suite structure

An eval suite is a set of test cases with expected outcomes. The exact format is **not yet standardized** by this spec — this is deliberate, because evaluation practice in the LLM field is still maturing.

Minimum structure:

```yaml
version: 1
suite: cod.review.security.evals
cases:
  - id: sql-injection-detection
    inputs:
      code: |
        def query(user_input):
            return db.execute("SELECT * FROM users WHERE id=" + user_input)
      language: python
      severity_threshold: medium
    expected:
      output_contains_cwe: "CWE-89"
      output_min_severity: high
  - id: no-false-positive-safe-code
    inputs:
      code: |
        def query(user_input):
            return db.execute("SELECT * FROM users WHERE id=%s", [user_input])
      language: python
    expected:
      findings_max_count: 0
```

### 4.4.3 Eval results in registries

A registry that hosts components SHOULD also host eval results per `(component, version, model)` tuple:

```json
{
  "component": "cod.review.security",
  "version": "1.0.0",
  "model": "claude-opus-4-7",
  "suite": "cod.review.security.evals@1.0.0",
  "timestamp": "2026-01-15T10:30:00Z",
  "passed": 47,
  "failed": 2,
  "score": 0.96
}
```

This is what lets a consumer check, before adopting: "what does this component actually do, measured against what?"

### 4.4.4 Evals and semver

From §4.2.4: every minor/patch release MUST pass all evals the previous version passed. Publishing a version that fails this check MUST be blocked by the publishing toolchain.

## 4.5 Deprecation

Components are deprecated before removal. A deprecated component continues to resolve but emits a warning at resolution time.

```yaml
deprecated:
  since: 1.5.0
  reason: "Superseded by cod.review.security-v2 which supports structured CWE output"
  replacement: cod.review.security-v2@^1.0.0
  removal_planned: 2.0.0
```

### 4.5.1 Deprecation rules

- Deprecation MUST NOT happen in a patch release.
- A component SHOULD remain resolvable for at least one major version after deprecation.
- Removal happens by publishing no new versions and eventually un-publishing old versions after a deprecation period specified by the registry.

### 4.5.2 Replacement semantics

When `replacement` is specified, automated migration tools MAY suggest the replacement to consumers. The spec does not mandate automatic substitution — replacement is advisory.

## 4.6 Provenance

Provenance is the chain of evidence connecting a component to its author. Without provenance, a component's claims (author identity, eval results, tested models) are assertions that cannot be verified.

### 4.6.1 Signing

A published component MAY be accompanied by a signature over its file content.

**Recommended formats** (to be finalized before 1.0):

- [Sigstore](https://www.sigstore.dev/) for public registries.
- SSH signatures (git-native) for git-based registries.
- Minisign for minimal-infrastructure deployments.

This is an **open question** — the spec does not yet commit to a single mechanism. See [README](README.md).

### 4.6.2 Signature verification

Runtimes SHOULD verify signatures on components obtained from registries they do not fully control. Signature verification failures MUST be treated as integrity failures equivalent to lockfile hash mismatches.

### 4.6.3 Supply-chain constraints

Because components can include other components transitively (§3.7), a malicious dependency can inject content into the final rendered prompt. Defenses:

- **Lockfile hashes** (§4.1) prevent silent mutation post-install.
- **Scoped handles** (§1.4.3) prevent typosquatting to a degree.
- **Explicit transitive visibility** — runtimes SHOULD surface the full set of components that will be rendered, not just direct dependencies.
- **Optional `--no-transitive` install mode** — a consumer MAY require every transitively-included component to be explicitly listed in their project manifest. Slow but safer.

These are recommendations, not requirements — the spec cannot mandate security practice. But it structures the information in ways that makes such practice possible.

## 4.7 Error format

Errors emitted by any spec-conformant runtime MUST follow this structure:

```yaml
error:
  code: COMPONENT_VALIDATION_FAILED
  message: "Input 'language' has invalid value 'java'"
  component: cod.review.security
  version: 1.0.0
  path: inputs.language
  details:
    received: java
    allowed: [python, typescript, go, rust]
  trace_id: ...
```

### 4.7.1 Error codes

The spec reserves the following error code prefixes:

| Prefix | Domain |
|--------|--------|
| `FORMAT_*` | File format errors (Part 1) |
| `KIND_*` | Component kind violations (Part 2) |
| `COMPOSITION_*` | Composition rule violations (Part 3) |
| `VALIDATION_*` | Input/output/slot validation |
| `RESOLUTION_*` | Dependency resolution |
| `INTEGRITY_*` | Hash, signature, or provenance failures |
| `RUNTIME_*` | Execution-time failures (model errors, timeouts) |

Specific codes within each prefix are implementation-defined; a future version of the spec may standardize them.

## 4.8 Trace format

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

Traces are what enable drift tracking, debugging across versions, and the eval-tied validation claims this spec rests on. Without traces, the trust claims in this part are assertions without evidence.

## 4.9 Summary

Trust in this system is the product of several mechanisms working together:

- **Semver + evals** mean version numbers are behavioral claims, not decorative.
- **Lockfiles + content hashes** mean resolution is reproducible and tamper-evident.
- **Compatibility metadata** means consumers can predict whether a component will work for them.
- **Provenance** (when implemented) means authorship claims are verifiable.
- **Traces** mean runtime behavior is auditable after the fact.

Any of these alone is insufficient. Together, they let a consumer reasonably trust a component they did not author — which is what separates this spec from a shared folder of markdown files.
