# Example 4: Task

A task is the complete, invokable component — what application code actually calls. This example extends the pattern from Example 3.

## The component

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
    extends: cod.review.base@^1.0.0
    inputs:
      severity_threshold:
        type: enum[low, medium, high]
        default: medium
        description: Minimum severity to report
    slot_fills:
      persona: met.p.reviewer-voice@^1.0.0
      format: met.p.format-json-findings@^1.0.0
      focus: |
        Look for security vulnerabilities at **{{severity_threshold}}+** severity:

        - Injection vulnerabilities (SQL, command, XSS, template)
        - Authentication and authorization flaws
        - Cryptographic misuse (weak algorithms, hardcoded keys, bad randomness)
        - Input validation and sanitization gaps
        - Information disclosure (logs, errors, timing)
        - Unsafe deserialization

        For each finding, reference the CWE ID where applicable.
      guards:
        - met.p.guard.no-hallucination@^1.0.0
        - met.p.guard.cite-lines@^1.0.0
    input_fills:
      persona:
        specialty: "application security and secure coding practices"
        tone: formal
    runtime_compatibility:
      tested_models: [claude-opus-4-7]
      compatible_models: ["claude-*", "gpt-4*"]
tags: [code, review, security, cwe]
---
```

Note there is no body. The task inherits the pattern's body and fills slots.

## Walkthrough

**`extends: cod.review.base@^1.0.0`.** The task inherits everything from the pattern — the `code` and `language` inputs, the body structure, the XML tags, the slot declarations.

**New input: `severity_threshold`.** The task adds a runtime input the pattern didn't have. Callers pass this at invocation.

**`slot_fills` fills all required pattern slots.**

- `persona` gets filled with a primitive reference.
- `format` gets filled with a primitive reference.
- `focus` gets filled with inline markdown — note that `{{severity_threshold}}` is interpolated from the new runtime input.
- `guards` gets filled with a list of two primitive references (overriding the pattern's default empty list).

**`input_fills` passes values to the nested primitives.** The `persona` slot references `met.p.reviewer-voice`, which itself has `specialty` and `tone` inputs. The task fills them here at authoring time — the caller never sees them.

**`runtime_compatibility.tested_models` is narrower than the pattern's.** The author has only exercised this specific task against `claude-opus-4-7`. That's a factual claim by the author; `compatible_models` is the broader claim of where it *should* work.

## How a caller invokes this

```python
from pbib import registry

task = registry.get("cod.review.security@^1.0.0")

result = task.run(
    code=open("auth.py").read(),
    language="python",
    severity_threshold="high",
)

for finding in result.findings:
    print(f"[{finding.severity}] Line {finding.line}: {finding.description}")
```

Four lines to call. The caller never touches the pattern, the persona primitive, the format primitive, or the guards. They just fill the task's three runtime inputs.

## The resolution that happens behind the scenes

When `task.run(...)` is called:

1. Runtime loads `cod.review.security@1.0.0`.
2. Follows `extends` to `cod.review.base@1.0.0`.
3. Merges inputs: `code`, `language` (from pattern) + `severity_threshold` (from task).
4. Validates caller inputs: `code` is a string, `language` is in the enum, `severity_threshold` is in the enum.
5. Loads each component referenced in slot_fills: `met.p.reviewer-voice`, `met.p.format-json-findings`, the two guard primitives.
6. Loads `met.tone@^1.0.0` (referenced by the persona primitive's `tone` input).
7. Resolves the `tone` token value: `formal` → "Use precise, professional language..."
8. Renders the persona primitive with `specialty="application security..."` and the resolved tone.
9. Renders the guard primitives.
10. Renders the format primitive.
11. Renders the task's `focus` slot fill with `severity_threshold=high` interpolated.
12. Assembles the final body by substituting all slot references in the pattern's body.
13. Sends to the model.
14. Parses output, validates against the JSON schema from the format primitive.
15. Returns a validated `result`.

Every one of those steps can fail with a structured error that names the exact component, field, and problem (Part 4 §4.6).

## The lockfile entry

After the first resolution, the project's lockfile records:

```yaml
components:
  "cod.review.security":
    version: 1.0.0
    integrity: "sha256-..."
    dependencies:
      "cod.review.base": "1.0.0"
      "met.p.reviewer-voice": "1.0.0"
      "met.p.format-json-findings": "1.0.0"
      "met.p.guard.no-hallucination": "1.0.0"
      "met.p.guard.cite-lines": "1.0.0"
      "met.tone": "1.0.0"
```

Every subsequent resolution of `cod.review.security` uses these exact versions until the lockfile is updated. This is what makes the behavior reproducible across time, CI runs, and production deployments.

When APM is the distribution tool, `apm.lock.yaml` plays this role instead — pinning by 40-char git commit SHA. Pbib defers to APM when present (Part 4 §4.1.1).

## Versioning this task

What would trigger what version bump?

- **Major (2.0.0):** Removing `severity_threshold`. Changing the output schema (via a new format primitive with a different schema) in an incompatible way.
- **Minor (1.1.0):** Adding a new severity value like `critical`. Adding a new optional input. Swapping to a newer-version persona primitive.
- **Patch (1.0.1):** Refining the focus prose without changing meaning. Fixing a typo.

These rules are author-disciplined (Part 4 §4.2.4). Consumers protect themselves by pinning exact versions or reading changelogs before upgrading.

## What this demonstrates

- Extension as the main mechanism for creating task variants.
- `slot_fills` + `input_fills` as two distinct authoring-time mechanisms.
- Runtime inputs flowing through authoring-time slot fills via interpolation.
- The full resolution picture: every piece of rendered prompt traces back to a specific component and version.
- The lockfile mechanism that makes resolution reproducible.
