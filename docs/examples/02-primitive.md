# Example 2: Primitive

A primitive is a reusable instruction fragment. Primitives have a body, typed inputs, and can reference tokens — but they cannot extend other components and cannot declare slots.

## The component

```yaml
---
name: reviewer-voice
description: Establishes a code-reviewer persona with configurable specialty and tone
license: MIT
metadata:
  pbib:
    version: 1.0.0
    kind: primitive
    handle: met.p.reviewer-voice
    inputs:
      specialty:
        type: string
        required: true
        description: What this reviewer specializes in (e.g., "application security")
      tone:
        type: token_ref
        bundle: met.tone@^1.0.0
        default: terse
    runtime_compatibility:
      compatible_models: ["claude-*", "gpt-4*"]
tags: [meta, persona, code]
---
You are an experienced code reviewer specializing in {{specialty}}.

{{token: met.tone.[tone]}}

Your job is to identify concrete, actionable issues. Do not restate what
the code does. If you are uncertain whether something is an issue, omit it.
```

## Walkthrough

**`name: reviewer-voice`.** Primary identifier within this scope.

**`metadata.pbib.handle: met.p.reviewer-voice`.** The `met.p.` prefix here is just one author's organizational choice — segments before the final one are how this author groups their components. The runtime determines kind from the `kind:` field, not from the handle.

**`metadata.pbib.inputs:` with two entries.** `specialty` is a required string. `tone` is a `token_ref` that must resolve to a key in the `met.tone@^1.0.0` bundle, with a default of `terse`.

**`metadata.pbib.runtime_compatibility.compatible_models`.** The author claims this works on any Claude model and any GPT-4 variant. Runtimes use this to warn if a consumer applies the component to, say, a small open-source model.

**Body.** Two variable references (`{{specialty}}` and `{{token: met.tone.[tone]}}`) plus static instruction text. The body doesn't use XML tags — for this primitive, prose is enough.

## How this primitive gets used

### Inside a slot

A pattern's body references this primitive via a slot:

```markdown
{{slot: persona}}
```

And a task fills the slot:

```yaml
metadata:
  pbib:
    slot_fills:
      persona: met.p.reviewer-voice
    input_fills:
      persona:
        specialty: "application security"
        tone: formal
```

The task passes values to the primitive's inputs via `input_fills`.

### As an inclusion

Another component can inline this primitive's body directly:

```yaml
---
name: some-other-task
metadata:
  pbib:
    version: 1.0.0
    kind: task
    inputs:
      code: { type: string, required: true }
    input_fills:
      "met.p.reviewer-voice@^1.0.0":
        specialty: "Python performance"
        tone: conversational
---
{{include: met.p.reviewer-voice@^1.0.0}}

Review this code:
<code>{{code}}</code>
```

Here the primitive's body is inlined where the `{{include: ...}}` tag appears, with its inputs resolved from the including component's `input_fills` section.

## Why primitives can't extend

Primitives are leaves in the composition graph. If `met.p.reviewer-voice` extended something, you'd have a chain: task → pattern → primitive → primitive → … and the semantics of "a primitive extending another primitive" are genuinely ambiguous (what does the child's body do — replace? wrap? append?).

The spec's answer: use extension only at the pattern/task layer, and use inclusion for primitive composition. One relationship, one meaning.

## What this demonstrates

- Inputs with types, defaults, and token references.
- The compatibility metadata pattern under `metadata.pbib.runtime_compatibility`.
- How tokens are dereferenced via indirect `[input_name]` substitution.
- The primitive's role: small, reusable, leaf-level.
