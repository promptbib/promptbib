# Part 3: Composition

> **Layer 3** of the spec. Defines how components reference, extend, and combine.

Composition is the heart of the system. Get this wrong and the spec is just a file format; get it right and components become a genuine programming model for LLM instruction.

This part defines six mechanisms:

1. **Inputs** — runtime-provided values (§3.2)
2. **Slots** — authoring-time extension points (§3.3)
3. **Tokens** — named terminal values with scoped overrides (§3.4)
4. **Outputs** — declared output shape with validation (§3.5)
5. **Extension** — parent-child relationships via `extends` (§3.6)
6. **Inclusion** — inlining one component's body into another (§3.7)

Plus one global concept: the resolution algorithm (§3.8).

## 3.1 Inputs vs slots: the core distinction

This distinction is load-bearing. Authors consistently confuse these; the spec clarifies:

- **Inputs** are values the *caller* provides at runtime. Inputs vary per invocation. They are declared in `inputs:` and filled by calling code.
- **Slots** are extension points the *child component* fills at authoring time. Slots vary per variant. They are declared in `slots:` and filled via `slot_fills:` in an extending component.

A task and a pattern typically declare both: inputs for data (code to review, document to summarize) and slots for structural variants (which persona, which output format).

Runtime callers never see slots; authors never "pass slot fills at runtime." If that's what a caller needs, the right abstraction is an input whose value flows into a slot fill.

## 3.2 Inputs

### 3.2.1 Declaration

`inputs` is a mapping from input name to input specification:

```yaml
inputs:
  code:
    type: string
    required: true
    description: The source code to review
  language:
    type: enum[python, typescript, go, rust, java]
    required: true
  severity_threshold:
    type: enum[low, medium, high]
    default: medium
```

### 3.2.2 Input types

| Type | Description |
|------|-------------|
| `string` | A text value. |
| `integer` | A whole number. |
| `number` | A numeric value (integer or float). |
| `boolean` | True or false. |
| `enum[a, b, c]` | One of an explicit set of values. |
| `list[T]` | A sequence of values of type T. |
| `object` | A nested mapping. When used, `schema` MUST be provided. |
| `token_ref` | A reference to a value in a token bundle. Requires `bundle`. |
| `component_ref` | A reference to another component. May require `allowed_kinds`. |
| `context_window` | Large contextual input with truncation metadata. **Agent profile only.** |

### 3.2.3 Input specification fields

| Field | Type | Description |
|-------|------|-------------|
| `type` | type expression | Required. One of the types above. |
| `required` | boolean | Default `false`. |
| `default` | any | Default value if not required. Must match `type`. |
| `description` | string | Human-readable description. |
| `bundle` | handle | For `token_ref`: the token bundle. |
| `allowed_kinds` | list | For `component_ref`: permitted component kinds. |
| `schema` | JSON schema | For `object`: the nested shape. |
| `min`, `max` | number | For numeric types: inclusive bounds. |
| `pattern` | regex | For string: validation regex. |

### 3.2.4 Validation

Runtimes MUST validate inputs at invocation time:

1. All required inputs present.
2. Each input's value matches its declared type.
3. Enum values are in the permitted set.
4. Numeric bounds, regex patterns, and schemas are satisfied.
5. `token_ref` values are keys present in the named bundle.
6. `component_ref` values resolve to components of an allowed kind.

Validation failures MUST produce a structured error identifying the input name and the violation. Runtimes MUST NOT silently coerce except where the spec explicitly permits (see §3.2.5).

### 3.2.5 Coercion

Runtimes MAY coerce:

- Integer-shaped strings to `integer` or `number`.
- `true`/`false` strings to `boolean`.
- Single values to single-element lists when the declared type is `list[T]` and the value matches T.

Runtimes MUST NOT coerce across enum values, across incompatible types, or in ways that could change meaning.

## 3.3 Slots

### 3.3.1 Declaration

```yaml
slots:
  persona:
    type: component_ref
    allowed_kinds: [primitive]
    required: true
    description: Which reviewer persona to adopt
  focus:
    type: markdown
    required: true
  guards:
    type: list[component_ref]
    allowed_kinds: [primitive]
    default: [met.p.guard.no-hallucination@^1.0.0]
```

### 3.3.2 Slot types

| Type | Description |
|------|-------------|
| `string` | A text value, no rendering assumptions. |
| `markdown` | A text value, interpreted as markdown when rendered. |
| `list[string]` | A sequence of strings. |
| `list[markdown]` | A sequence of markdown fragments. |
| `component_ref` | A handle pointing at a primitive or pattern to install. |
| `list[component_ref]` | A sequence of component handles, rendered in order. |

Slots MUST NOT be typed as `object`, `integer`, or other non-textual primitive types. Slots exist to produce prompt text.

### 3.3.3 Filling slots

A slot is filled in one of two ways:

**Via `slot_fills` in an extending component:**

```yaml
extends: cod.review.base@^1.0.0
slot_fills:
  persona: met.p.reviewer-voice@^1.0.0
  focus: |
    Look for performance issues...
  guards:
    - met.p.guard.no-hallucination@^1.0.0
    - met.p.guard.performance-focus@^1.0.0
```

**Via a default in the slot declaration** (the slot is not required and no child fills it):

```yaml
slots:
  output_format:
    type: component_ref
    allowed_kinds: [primitive]
    default: met.p.format-bullets@^1.0.0
```

### 3.3.4 Rendering slots in the body

The body references slots via `{{slot: name}}`. For `component_ref` slots, the reference expands to the referenced component's body (with its own inputs resolved). For `list[component_ref]` slots, the reference expands to each component's body in order, separated by two newlines.

For text-type slots, the reference expands to the literal filled value.

### 3.3.5 List-slot append mode

An extending component MAY append to (rather than replace) a list slot by using the `+` prefix on the slot name:

```yaml
slot_fills:
  "+guards":
    - met.p.guard.security-focus@^1.0.0
```

This appends to the inherited list rather than replacing it. Append mode applies only to list-typed slots; its use on non-list slots is a validation error.

## 3.4 Tokens

### 3.4.1 Token resolution scopes

Token values resolve through four scopes, in precedence order:

1. **Runtime call** — value passed at invocation, if any.
2. **Component default** — the `default:` in the `inputs:` declaration.
3. **Project override** — values declared in `promptbib.yaml` for the current project.
4. **Registry value** — the value in the published token bundle.

Higher-precedence scopes override lower. If a token is not resolvable at any scope, the runtime MUST produce an error.

### 3.4.2 Project overrides

A project's root manifest MAY declare token overrides:

```yaml
# promptbib.yaml
name: acme-prompts
version: 0.3.0
dependencies:
  met.tone: "^1.0.0"
tokens:
  "met.tone.formal": "Use precise language. Match ACME's house style guide."
  "acme.voice":
    friendly: "Warm but professional, never saccharine."
```

Overrides MUST match the type of the original token. A runtime MUST validate this at project load time.

### 3.4.3 Referencing tokens

In component bodies: `{{token: bundle.key}}`, e.g. `{{token: met.tone.formal}}`.

Indirect references (via an input): `{{token: met.tone.[tone]}}`, where `tone` is an input of type `token_ref`. The runtime substitutes the input value before token resolution.

## 3.5 Outputs

### 3.5.1 Declaration

```yaml
outputs:
  format: json
  schema:
    type: object
    required: [findings]
    properties:
      findings:
        type: array
        items:
          type: object
          required: [severity, line, description]
          properties:
            severity: { enum: [low, medium, high, critical] }
            line: { type: integer }
            description: { type: string }
            cwe: { type: string, pattern: "^CWE-\\d+$" }
```

### 3.5.2 Output formats

| Format | Description |
|--------|-------------|
| `text` | Unstructured text. No schema. |
| `markdown` | Markdown-formatted text. No schema. |
| `json` | JSON output matching `schema`. |
| `structured` | Provider-specific structured output (e.g., function-calling response). |

### 3.5.3 Validation and retry

When a component declares `outputs` with a schema, the runtime MUST parse the model's output and validate it against the schema. On validation failure, runtimes SHOULD attempt one automatic retry with the validation error included in the retry prompt. Persistent failure after retry MUST be raised as a structured error.

## 3.6 Extension

### 3.6.1 Syntax

```yaml
extends: parent.handle@version-constraint
```

Version constraints follow semver range syntax (`^1.0.0`, `~2.1.0`, `>=1.0.0 <2.0.0`).

### 3.6.2 Extension semantics

When component C extends component P:

**Inputs merge.** C's inputs are added to P's inputs. C MAY:

- Add new inputs.
- Override the `default` of an inherited input.
- **Narrow** an enum's permitted values (subset).
- **Tighten** numeric bounds (raise `min`, lower `max`).

C MUST NOT:

- Remove an input declared by P.
- **Widen** an enum (add new permitted values) — this is a breaking change.
- Change an input's type to an incompatible type.

**Slots override.** C's `slot_fills` fill P's slots. After extension:

- Every required slot in P MUST be filled by C (or inherited up the chain).
- C MAY introduce new slots.
- C MAY NOT remove slots declared by P.

**Outputs narrow.** C's `outputs` schema MAY tighten (add required fields, restrict enums) but MAY NOT loosen (remove required fields, widen types). A loosening is a major-version break.

**Metadata merges.**

- `tags`: union of parent and child.
- `description`: child overrides parent.
- `compatibility.compatible_models`: intersection of parent and child (child must be compatible with a subset of parent's models).
- `compatibility.tested_models`: child's set (does not inherit).

**Body inheritance.** C inherits P's body. C MAY override P's body entirely by providing its own body (unusual). C MAY reference the parent's body via `{{super}}` if it wraps rather than replaces.

### 3.6.3 Extension depth

Runtimes SHOULD limit extension chain depth to 8 levels. Deeper chains typically indicate design problems and significantly complicate debugging.

### 3.6.4 Diamond resolution

A component with two ancestors that share a common ancestor (diamond) is permitted. Conflict resolution rules:

- Inputs: the most-derived declaration wins.
- Slots: the most-derived `slot_fill` wins.
- Body: the nearest ancestor's body is used.

Conflicting declarations at the same derivation depth are a validation error. Authors MUST resolve by explicit override.

## 3.7 Inclusion

### 3.7.1 Syntax

```
{{include: handle@version-constraint}}
```

### 3.7.2 Semantics

Inclusion inlines another component's rendered body into the current body at render time. The included component's inputs MUST either:

1. Have defaults, or
2. Be satisfied via an `input_fills` entry in the including component.

```yaml
input_fills:
  "met.p.reviewer-voice@^1.0.0":
    specialty: "application security"
```

### 3.7.3 Inclusion vs extension

- **Extension** is a whole-component inheritance relationship (the child IS-A specialization of the parent).
- **Inclusion** is a content relationship (this component USES that component's text).

Inclusion does not inherit inputs, slots, or outputs. It only inlines body content.

### 3.7.4 Cycles

Runtimes MUST detect inclusion cycles and produce an error. A → B → A is invalid. The spec does not define behavior for self-inclusion, which is always an error.

## 3.8 Resolution algorithm

Given a task invocation with runtime inputs:

1. **Load.** Parse the component file. Validate format (Part 1) and kind (Part 2).
2. **Resolve extends chain.** Recursively load parents. Validate extension rules (§3.6.2). Build merged declarations.
3. **Resolve component_refs.** For every `component_ref` input, slot fill, and included component, load and validate the referenced component.
4. **Validate slots.** Every required slot must have a fill or default. Validate fill types against slot declarations.
5. **Validate inputs.** Apply caller-provided inputs against merged input declarations (§3.2.4).
6. **Resolve tokens.** Look up every token reference through the four-scope hierarchy (§3.4.1).
7. **Render body.** Recursively render included components and component-ref slot fills. Substitute input values. Assemble final prompt text.
8. **Execute.** Send to model. (Out of scope for this spec.)
9. **Validate output.** If `outputs.schema` is declared, parse and validate the model response.
10. **Emit trace.** Record resolution details for observability ([Part 4](04-trust.md)).

Runtimes MUST perform steps 1–6 before step 7. Runtimes MAY cache the results of steps 1–6 across invocations when inputs don't require re-resolution.

## 3.9 Templating scope

The four variable forms in §1.5 (`{{input}}`, `{{slot: ...}}`, `{{token: ...}}`, `{{include: ...}}`) are the only interpolation forms this spec defines. There is intentionally no:

- Conditional rendering (`{{if}}` / `{{else}}`).
- Loops (`{{for}}`).
- Expression evaluation.
- Filters or transformations.

Rationale: prompt authors attempting logic in templates produce hard-to-debug artifacts. Runtime consumers that need conditional or iterative composition should express that logic at the workflow or application layer, not in component bodies.

This is an **open question** for the spec — whether a minimal, well-specified template language (analogous to Mustache's "logic-less templates") would pay for itself. See [README](README.md).

## 3.10 Example: full composition

```yaml
---
handle: cod.review.security
version: 1.0.0
kind: task
spec: 0.1.0
extends: cod.review.base@^1.0.0
inputs:
  severity_threshold:
    type: enum[low, medium, high]
    default: medium
slot_fills:
  persona: met.p.reviewer-voice@^1.0.0
  format: met.p.format-json-findings@^1.0.0
  focus: |
    Look for security vulnerabilities at **{{severity_threshold}}+** severity.
input_fills:
  persona:
    specialty: "application security"
    tone: formal
---
```

Resolution produces (conceptually):

```
<persona primitive body, with specialty="application security" and tone token resolved>

<review_focus>
Look for security vulnerabilities at **high+** severity.
</review_focus>

<code language="python">
def authenticate(user, password):
    ...
</code>

<format primitive body, JSON schema instructions>
```

Every piece of that rendered output is traceable back to a specific component and version, which the lockfile ([Part 4](04-trust.md)) records. No part of the rendered prompt is untraceable, which is what makes the system auditable.
