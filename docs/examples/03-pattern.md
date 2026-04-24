# Example 3: Pattern

A pattern is a structural template with slots that specializations fill. Patterns encode the *shape* of a class of prompts without committing to specific content.

## The component

```yaml
---
handle: cod.review.base
version: 1.0.0
kind: pattern
spec: 0.1.0
description: Base structure for code review tasks. Specializations fill the focus slot.
inputs:
  code:
    type: string
    required: true
    description: The source code to review
  language:
    type: enum[python, typescript, go, rust, java]
    required: true
slots:
  persona:
    type: component_ref
    allowed_kinds: [primitive]
    required: true
    description: Which reviewer persona to adopt
  focus:
    type: markdown
    required: true
    description: What to look for in this review
  format:
    type: component_ref
    allowed_kinds: [primitive]
    required: true
    description: How output should be structured
  guards:
    type: list[component_ref]
    allowed_kinds: [primitive]
    default: []
    description: Optional constraint primitives to include
compatibility:
  tested_models: [claude-opus-4-7, claude-sonnet-4-6]
  compatible_models: ["claude-*", "gpt-4*"]
  style: xml_structured
tags: [code, review, pattern]
---
{{slot: persona}}

<review_focus>
{{slot: focus}}
</review_focus>

{{slot: guards}}

<code language="{{language}}">
{{code}}
</code>

{{slot: format}}
```

## Walkthrough

**Four slots with different types.**

- `persona` — must be a primitive (not a pattern, not markdown text). Required.
- `focus` — markdown text. Required. This is where specializations express the *what* of the review.
- `format` — must be a primitive. Required.
- `guards` — a list of primitives. Defaults to empty. Specializations can add guard primitives to constrain the model.

**`allowed_kinds: [primitive]`.** This is the spec preventing authors from installing arbitrary content into structural slots. You cannot fill `persona` with a 500-word inline blob because the slot type rejects it at validation time.

**`compatibility.style: xml_structured`.** Informational. The pattern uses XML tags (`<review_focus>`, `<code>`) because the author tested it on Claude and decided XML structure helps. A registry or linter can use this to warn consumers whose target models prefer a different style.

**Body.** Static structure with four slot references. The body is the pattern — what varies is what fills the slots.

## The `persona` and `focus` slots illustrate the design

Note how `persona` is a `component_ref` while `focus` is `markdown`. Why?

- **Persona** is a cross-cutting concern. Teams will build a library of reusable personas (security reviewer, performance reviewer, accessibility reviewer). Forcing `persona` to be a component reference ensures that library gets reused. If it were `markdown`, every task would reinvent the persona inline.
- **Focus** is task-specific. The exact prose describing "what to look for in a security review" vs "what to look for in a performance review" is the whole point of the specialization. Forcing it through a component reference would create trivial one-use primitives.

The rule of thumb: **slot type follows expected reuse.** If the slot fill will be reused across tasks, make it `component_ref`. If it's inherently task-specific, make it `markdown`.

## What the pattern cannot do

Patterns don't get called directly. A runtime invocation like `registry.get("cod.review.base").run(code="...", language="python")` is a validation error — the pattern's required slots are unfilled.

Patterns exist to be extended. The only way to execute one is to extend it into a task (next example).

## Versioning this pattern

What counts as a breaking change?

- **Major (2.0.0):** Removing a slot. Renaming a slot. Adding a required slot with no default.
- **Minor (1.1.0):** Adding an optional slot with a default. Adding a new enum value to `language`. Adding a new compatible model.
- **Patch (1.0.1):** Fixing a typo in the description. Updating tags.

A body change (e.g., restructuring the XML tags) is major if it changes output shape, minor if the eval suite still passes. See Part 4 §4.2 for the eval backstop rule.

## What this demonstrates

- Typed slots with `component_ref` and `allowed_kinds`.
- List slots with defaults.
- The distinction between structural reuse (`component_ref` slots) and task-specific content (`markdown` slots).
- Informational style metadata that helps discoverability without prescribing.
- Why patterns exist: to make variation structural rather than accidental.
