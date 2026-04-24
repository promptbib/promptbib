# Portable Agent Component Format

**Status:** Draft 0.1
**Editors:** TBD
**Format version:** 0.1.0

A portable file format and semantic model for LLM instruction artifacts — prompts, skills, personas, tools, workflows — that makes them reusable within teams, composable across projects, and verifiable between agents.

## Goals

1. **Portability.** One format that any LLM-powered system (chat interface, coding agent, internal tool) can consume.
2. **Reusability.** Components can be composed, extended, and shared without copy-paste.
3. **Verifiability.** Every component is inspectable, typeable, and optionally eval-backed. A consumer can determine what a component does and how well it does it before running it.
4. **Trust between agents.** When an agent installs a component from a third party — or hands work to another agent via a shared component — the contract is explicit.

## Non-goals

- **Prescribing prompt style.** The format is agnostic about whether component bodies use XML, markdown, or plain prose.
- **Replacing runtimes.** The spec defines a file format and semantic model. How a given agent loads, caches, or renders components is implementation-defined.
- **Solving every prompt problem.** The spec is intentionally small. Orchestration, retrieval, and agent control flow are out of scope.

## Architecture

The spec is organized into four layers. Each layer builds on the one below; each can be implemented incrementally.

| Layer                     | What it defines                               | Part                                    |
|---------------------------|-----------------------------------------------|-----------------------------------------|
| 1. File format            | How a single component is written on disk     | [Part 1](./docs/spec/01-format.md)      |
| 2. Semantic model         | Component kinds and what each can do          | [Part 2](./docs/spec/02-kinds.md)       |
| 3. Composition            | How components reference, extend, and combine | [Part 3](./docs/spec/03-composition.md) |
| 4. Trust and verification | Evals, signing, provenance, compatibility     | [Part 4](./docs/spec/04-trust.md)       |

A fifth document — [Part 5: Profiles](05-profiles.md) — defines conformance levels (core, agent) and which parts of the spec each profile requires.

Worked examples are in [`examples/`](./docs/examples/).

## Reading order

- **If you're evaluating whether to adopt:** README → [Part 5](./docs/spec/05-profiles.md) → [`examples/`](./docs/examples/).
- **If you're authoring components:** [Part 1](01-format.md) → [Part 2](./docs/spec/02-kinds.md) → [`examples/`](./docs/examples/) → [Part 3](./docs/spec/03-composition.md).
- **If you're implementing a runtime:** all five parts in order, plus [`examples/`](./docs/examples/) as test fixtures.
- **If you're reviewing the design:** [Part 2](./docs/spec/02-kinds.md) and [Part 3](./docs/spec/03-composition.md) are where the load-bearing decisions live.

## Conformance

A document using this spec MUST specify a profile. Profiles are defined in [Part 5](./docs/spec/05-profiles.md):

- **Core profile** — static components, file-based registries, offline resolution. Suitable for prompt libraries and applications.
- **Agent profile** — strict superset of core, adds dynamic context, streaming inputs, tool components, and cache-aware resolution.

Where this spec uses the words **MUST**, **MUST NOT**, **SHOULD**, **SHOULD NOT**, and **MAY**, they are to be interpreted as described in [RFC 2119](https://www.rfc-editor.org/rfc/rfc2119).

## Open questions

The following are unresolved and flagged in the relevant parts:

- Naming of the canonical domain taxonomy (the "three-letter table"). See [Part 2](02-kinds.md).
- Whether to bless a templating language or define composition as pre-rendering only. See [Part 3](03-composition.md).
- Signing and provenance format — candidates are Sigstore, SSH signatures, or minisign. See [Part 4](04-trust.md).
- Whether the agent profile should be its own spec document once it stabilizes. See [Part 5](05-profiles.md).

Feedback on these is especially welcome.

## Versioning of this spec

The spec itself follows semantic versioning. Component files declare the spec version they target via the `spec` field. Runtimes MUST reject components that target a spec version they do not implement.
