# Part 1: File format

> **Layer 1** of the spec. Defines how a single component is written on disk.

A component is a UTF-8 encoded text file with two sections: **front matter** (structured metadata, YAML) and **body** (free-form text).

## 1.1 File extension

A component file MUST use one of the following extensions:

- `.prompt.md` — canonical single-file format. Body is interpreted as prose/markdown.
- `.prompt.yaml` — manifest-only form. Body, if present, is a YAML string field.

Runtimes MUST support `.prompt.md`. Support for `.prompt.yaml` is OPTIONAL.

The double extension (`.prompt.md`) is deliberate: it gives editors correct syntax highlighting while making components findable with `*.prompt.*` globs regardless of body format.

## 1.2 File structure

A component file consists of:

1. Opening delimiter: `---` on its own line
2. Front matter: valid YAML
3. Closing delimiter: `---` on its own line
4. Body: any text content until end of file

```
---
handle: cod.review.security
version: 1.0.0
kind: task
spec: 0.1.0
---
Body content starts here.
```

### 1.2.1 Front matter

Front matter MUST be valid YAML. The top-level value MUST be a mapping (not a sequence or scalar).

### 1.2.2 Body

The body is everything after the closing `---` up to the end of file. The body is **not** parsed by the spec; its interpretation is defined by the component's `kind` and processed at render time (see [Part 3](03-composition.md)).

A component MAY have an empty body. This is common for components that use `extends` and only fill slots (see [Part 3](03-composition.md)).

### 1.2.3 Encoding

Files MUST be UTF-8. Byte-order marks (BOM) MUST NOT be present. Line endings MAY be `LF` or `CRLF`; runtimes MUST accept both but SHOULD normalize to `LF` when emitting files.

## 1.3 Reserved front matter fields

The following top-level fields are reserved by this spec. Authors MAY add additional fields for tool-specific metadata; runtimes MUST ignore unknown fields.

### 1.3.1 Required fields

| Field | Type | Description |
|-------|------|-------------|
| `handle` | string | Globally unique identifier. See §1.4. |
| `version` | string | Semantic version of this component. See [Part 4 §4.2](04-trust.md). |
| `kind` | string | Component kind. One of the values defined in [Part 2](02-kinds.md). |
| `spec` | string | Version of this specification the component targets. |

### 1.3.2 Optional fields

| Field | Type | Description |
|-------|------|-------------|
| `description` | string | Short human-readable summary. SHOULD be ≤ 200 characters. |
| `inputs` | mapping | Runtime input declarations. See [Part 3 §3.2](03-composition.md). |
| `slots` | mapping | Authoring-time extension points. See [Part 3 §3.3](03-composition.md). |
| `outputs` | mapping | Output schema. See [Part 3 §3.5](03-composition.md). |
| `extends` | string | Handle of parent component. See [Part 3 §3.6](03-composition.md). |
| `slot_fills` | mapping | Values for the parent's slots when using `extends`. |
| `input_fills` | mapping | Values for nested components' inputs. |
| `compatibility` | mapping | Model and runtime compatibility metadata. See [Part 4 §4.3](04-trust.md). |
| `evals` | sequence | References to eval suites. See [Part 4 §4.4](04-trust.md). |
| `tags` | sequence of strings | Free-form categorization labels. |
| `deprecated` | mapping | Deprecation notice. See [Part 4 §4.5](04-trust.md). |

Per-kind additional fields are defined in [Part 2](02-kinds.md).

## 1.4 Handle grammar

A handle uniquely identifies a component within a namespace.

### 1.4.1 Grammar (ABNF)

```
handle         = domain "." family *("." segment)
domain         = 3*4 ALPHA                       ; 3-4 lowercase letters
family         = segment
segment        = ALPHA *(ALPHA / DIGIT / "-")    ; lowercase, kebab-case
```

Examples of valid handles:

- `cod.review` (domain + family, two segments)
- `cod.review.security` (three segments)
- `met.tone` (meta domain, single-family token bundle)
- `acme.billing.invoice.reminder` (four segments, project-scoped)

### 1.4.2 Domain reservation

Three-letter domains are reserved for the canonical domain table. The canonical table is defined at the registry level, not by this spec. A runtime MAY validate domains against a table supplied at configuration time.

Four-letter domains are available for project-local or vendor-specific namespacing. By convention, organizations SHOULD use a four-letter prefix (e.g., `acme`) for internal components to avoid collisions with the canonical table.

### 1.4.3 Scoped handles

A handle MAY be scoped with a leading `@org/` prefix:

```
@anthropic/cod.review.security
@acme/biz.proposal.draft
```

Scopes operate exactly as in the npm package ecosystem: they disambiguate authors publishing in the same domain. An unscoped handle refers to the canonical public registry; a scoped handle refers to that scope's namespace.

### 1.4.4 Uniqueness

Within a given scope, the pair `(handle, version)` MUST be unique. A runtime that encounters two components with the same `(scope, handle, version)` MUST reject the registry as invalid.

## 1.5 Variable interpolation

The body MAY contain variable references. The spec defines four reference forms:

| Form | Meaning | Resolved at |
|------|---------|-------------|
| `{{input_name}}` | A runtime input | render time |
| `{{slot: slot_name}}` | A slot fill | load time |
| `{{token: bundle.key}}` | A token value | load time |
| `{{include: handle@version}}` | Inline another component's body | load time |

References outside these four forms are not defined by this spec. Runtimes MAY support additional forms (e.g., conditionals, loops) but MUST document them as extensions.

See [Part 3 §3.7](03-composition.md) for detailed resolution semantics.

## 1.6 Comments

YAML comments (lines starting with `#`) in front matter are permitted and ignored.

The body has no comment syntax defined by this spec. Authors who need annotations SHOULD keep them in the body text itself and rely on the body format's conventions (e.g., HTML comments `<!-- -->` for markdown bodies).

## 1.7 File size

The spec places no hard limit on component file size. Runtimes SHOULD warn when a component body exceeds 16 KB, as large bodies typically indicate the component should be decomposed.

Token bundles and tool definitions are exempt from this guidance.

## 1.8 Example

A minimal valid component:

```markdown
---
handle: met.tone.formal
version: 1.0.0
kind: token_bundle
spec: 0.1.0
description: Formal tone instructions
---
Use precise, professional language. Avoid contractions and colloquialisms.
```

See [`examples/`](examples/) for complete examples of every component kind.

## 1.9 Validation

A runtime MUST validate the following before accepting a component:

1. File is valid UTF-8.
2. Front matter is present, delimited, and valid YAML with a top-level mapping.
3. All required fields (§1.3.1) are present.
4. `handle` conforms to the grammar (§1.4.1).
5. `version` conforms to semantic versioning ([Part 4 §4.2](04-trust.md)).
6. `kind` is a value defined by the kinds vocabulary ([Part 2](02-kinds.md)).
7. `spec` is a version the runtime implements.

Additional validation defined per-kind and per-composition-rule happens later; see the relevant parts.

Validation failures MUST produce structured errors naming the file, the field, and the violation. See [Part 4 §4.6](04-trust.md) for the error format.
