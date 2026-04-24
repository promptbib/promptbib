# Example 1: Token bundle

A token bundle defines named terminal values that other components reference. This is the simplest component kind — it has no body, no inputs, no slots.

## The component

```yaml
---
handle: met.tone
version: 1.0.0
kind: token_bundle
spec: 0.1.0
description: Named tone values for prompt authors
values:
  formal: "Use precise, professional language. Avoid contractions and colloquialisms."
  terse: "Be maximally concise. No preamble. No caveats. No closing remarks."
  conversational: "Write naturally, as if explaining to a colleague over coffee."
  authoritative: "Write with confidence. State conclusions directly."
tags: [meta, style, tone]
---
```

## Walkthrough

**`handle: met.tone`.** The domain is `met` (meta), which is where cross-cutting components live. The family is `tone` — there's no further segment needed because this is a single bundle.

**`kind: token_bundle`.** The kind tells the runtime: this has no body, no inputs, no slots. Just look for `values:`.

**`values:`.** Each entry is a leaf — a string terminal value. Tokens don't reference other tokens (see Part 2 §2.3).

## How it's used

Other components reference tokens in two ways.

**Direct reference** — the bundle and key are known at authoring time:

```markdown
{{token: met.tone.formal}}
```

**Indirect reference** — the key comes from an input:

```yaml
inputs:
  tone:
    type: token_ref
    bundle: met.tone@^1.0.0
    default: terse
---
{{token: met.tone.[tone]}}
```

The square brackets around `[tone]` tell the runtime to substitute the input's value before resolving the token.

## Versioning

Token bundles version as a unit. If you add `warm` to the bundle, that's a minor bump (1.1.0). If you rename `terse` to `concise`, that's a major bump (2.0.0) because existing references break.

## What a project override looks like

A project can override a token value in its manifest:

```yaml
# promptbib.yaml
tokens:
  "met.tone.formal": "Use precise language consistent with ACME's style guide, section 3.2."
```

The override matches the original type (string) and becomes the value used when rendering anywhere in the project. See Part 3 §3.4.

## What this demonstrates

- The minimum structure a valid component needs.
- The token bundle pattern: values, not logic.
- The resolution scope system at its simplest.
