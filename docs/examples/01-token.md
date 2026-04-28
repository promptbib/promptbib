# Example 1: Token bundle

A token bundle defines named terminal values that other components reference. This is the simplest component kind — it has no body, no inputs, no slots.

## The component

```yaml
---
name: tone
description: Named tone values for prompt authors
license: MIT
metadata:
  pbib:
    version: 1.0.0
    kind: token_bundle
    handle: met.tone
    values:
      formal: "Use precise, professional language. Avoid contractions and colloquialisms."
      terse: "Be maximally concise. No preamble. No caveats. No closing remarks."
      conversational: "Write naturally, as if explaining to a colleague over coffee."
      authoritative: "Write with confidence. State conclusions directly."
tags: [meta, style, tone]
---
```

## Walkthrough

**`name: tone`.** The Agent Skills required field. Within this scope, just `tone` is enough.

**`metadata.pbib.handle: met.tone`.** The optional structured handle gives this component cross-scope addressability. `met.tone` is one author's organizational choice — your library might use any structure (`tone`, `style.tone`, `acme.shared.tone`, etc.). The spec defines the grammar but not the meaning.

**`metadata.pbib.kind: token_bundle`.** The kind tells the runtime: this has no body, no inputs, no slots. Just look for `values:`.

**`metadata.pbib.values:`.** Each entry is a leaf — a string terminal value. Tokens don't reference other tokens (see Part 2 §2.4).

## How it's used

Other components reference tokens in two ways.

**Direct reference** — the bundle and key are known at authoring time:

```markdown
{{token: met.tone.formal}}
```

**Indirect reference** — the key comes from an input:

```yaml
metadata:
  pbib:
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
# project manifest
tokens:
  "met.tone.formal": "Use precise language consistent with ACME's style guide, section 3.2."
```

The override matches the original type (string) and becomes the value used when rendering anywhere in the project. See Part 3 §3.4.

## What this demonstrates

- The minimum structure a Level 1 component needs.
- Where Agent Skills standard fields (`name`, `description`, `license`, `tags`) live vs. where pbib-specific fields live.
- The token bundle pattern: values, not logic.
- The resolution scope system at its simplest.
