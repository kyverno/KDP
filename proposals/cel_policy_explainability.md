# Meta
[meta]: #meta
- Name: CEL Policy Evaluation Explainability
- Start Date: 2026-05-21
- Update data (optional): N/A
- Author(s): rx18-eng
- Supersedes: N/A

# Table of Contents
[table-of-contents]: #table-of-contents
- [Meta](#meta)
- [Table of Contents](#table-of-contents)
- [Overview](#overview)
- [Definitions](#definitions)
- [Motivation](#motivation)
- [Proposal](#proposal)
- [Implementation](#implementation)
- [Migration (OPTIONAL)](#migration-optional)
- [Drawbacks](#drawbacks)
- [Alternatives](#alternatives)
- [Prior Art](#prior-art)
- [Unresolved Questions](#unresolved-questions)
- [CRD Changes (OPTIONAL)](#crd-changes-optional)

# Overview
[overview]: #overview

The CEL-based policy types (ValidatingPolicy, MutatingPolicy, GeneratingPolicy, DeletingPolicy, ImageValidatingPolicy) evaluate CEL expressions to decide whether a policy applies to a resource and what action to take. Today, when a policy does not behave the way a user expects (a match condition silently evaluates to false, a variable resolves to an unexpected value, or an expression errors at runtime), Kyverno surfaces only the final outcome, not the reasoning behind it. This proposal adds an opt-in explainability capability that captures the per-expression CEL evaluation trace (which match conditions passed or failed, what each variable resolved to, which expression produced the verdict, and where an error originated) and exposes it through the CLI (a `kyverno explain` command and an `--explain` flag on `kyverno test` / `kyverno apply`) and, in a later phase, through opt-in runtime diagnostics on the live admission and background paths.

# Definitions
[definitions]: #definitions

- **CEL**: Common Expression Language, the expression language used by the `policies.kyverno.io` v1beta1 policy types.
- **EvalDetails**: the cel-go `*cel.EvalDetails` value returned alongside an evaluation result when a program is compiled with the state-tracking option. It records the computed value of each node in the expression's abstract syntax tree (AST).
- **Match condition**: a CEL expression on a policy that gates whether the policy applies to a given request or resource.
- **Variable**: a named CEL expression whose result is reusable within other expressions of the same policy.
- **Decision trace**: the structured record this proposal introduces, describing how a single policy evaluation reached its outcome for a single resource.

# Motivation
[motivation]: #motivation

The v1beta1 CEL policy types are the project's strategic direction; the traditional `kyverno.io/v1` ClusterPolicy and Policy types are deprecated with planned removal. As users migrate, a recurring friction point is that CEL policies are harder to debug than the policies they replace. When a CEL policy does not fire, or fires unexpectedly, the user has no visibility into the evaluation:

- A match condition that evaluates to false produces no signal about which sub-expression was responsible.
- A variable that resolves to an unexpected value is invisible.
- A CEL runtime error during background scan or admission surfaces as a generic failure rather than the offending sub-expression.

This appears repeatedly in user reports, including a MutatingPolicy that does not apply with no log explaining why, CLI test results that disagree with their own displayed reason, and offline CEL evaluation that produces opaque outcomes.

The expected outcome is that a user can ask Kyverno "why did this policy reach this decision for this resource?" and receive a precise, per-expression answer. This turns multi-hour debugging sessions and filed bug reports into a single self-service command.

Representative use cases:

- A platform engineer authoring a new ValidatingPolicy checks why it does not match a test resource.
- An operator debugging a MutatingPolicy that does not apply in-cluster sees which match condition gated it out.
- A user triaging a background-scan discrepancy sees that a variable resolved to an unexpected value.

# Proposal

A decision trace for CEL policy evaluation, surfaced through two interfaces.

**CLI (phase 1):** a `kyverno explain` command, plus an `--explain` flag on `kyverno test` and `kyverno apply`, accepts a policy and one or more resources and prints, per resource and per policy:

- each match condition, the boolean it produced, and the sub-expression values that led to it
- the resolved value of each variable
- the validation, mutation, generation, or deletion expression that produced the verdict
- for an error, the exact sub-expression that failed and the error message

Illustrative output:

```
$ kyverno explain --policy require-prod-labels.yaml --resource pod.yaml

ValidatingPolicy/require-prod-labels  ->  Pod/default/prod-app
  matchConditions:
    only-prod    object.metadata.namespace == "prod"
                 object.metadata.namespace = "default"   -> false
  result: SKIPPED (match conditions not satisfied)
```

**Runtime (phase 2):** a policy can opt into a debug mode so the same trace is emitted as a structured event or a high-verbosity log entry on the live admission and background paths, for cases that reproduce only in-cluster.

The capability is opt-in everywhere. State tracking is enabled only when an explanation is requested, so the default admission path is unaffected.

# Implementation

The core insight is that Kyverno already computes the data needed for an explanation and then discards it. Across the CEL policy compilers (for example `pkg/cel/policies/vpol/compiler/policy.go`, and the equivalent files for mpol, gpol, and dpol), evaluation calls take the form:

```go
out, _, err := matchCondition.ContextEval(ctx, data)
out, _, err := variable.ContextEval(ctx, dataNew)
out, _, err := validation.Program.ContextEval(ctx, dataNew)
```

The discarded second return value is the cel-go `*cel.EvalDetails`. When a CEL program is compiled with the `cel.OptTrackState` program option, `EvalDetails.State()` exposes the evaluated value of every node in the expression's AST, keyed by node id. The source information that cel-go retains at parse time maps those node ids back to source-text spans, which is how a node value becomes a readable per-expression result.

Phase 1 (CLI):

1. Add an internal explain mode to the CEL compiler and engine that compiles programs with `cel.OptTrackState` and retains the `EvalDetails` instead of discarding it. This mode is off by default and enabled only for explanation requests.
2. Add a trace-collection type that, given the `EvalDetails`, the program's AST, and the source info, produces a structured per-expression record (expression text, evaluated value, error if any).
3. Wire the trace into the CLI processor used by `kyverno test` and `kyverno apply`, behind `--explain`, and add a dedicated `kyverno explain` command for the single-policy, single-resource case.
4. Render the trace in a human-readable form, with an `-o json` option for tooling.

Phase 2 (runtime, separate follow-up): allow a policy to opt into emitting the trace from the live admission and background paths via a debug annotation or spec field, surfaced as a structured event or a `V(n)` log entry.

## Link to the Implementation PR

N/A (proposal stage).

# Migration (OPTIONAL)

N/A. The feature is additive and opt-in. It introduces no breaking changes to existing APIs, and the default evaluation path is unchanged when an explanation is not requested.

# Drawbacks

- State tracking via `cel.OptTrackState` has runtime overhead, so it cannot be always-on in the admission path. This constrains the runtime phase to opt-in usage.
- Mapping AST node state back to readable source spans adds implementation complexity and must track cel-go version changes.
- The trace output format becomes a user-facing surface that needs to remain stable once published.

# Alternatives

- **Add the missing per-policy result metrics for the CEL types** (for example `kyverno_policy_results` for ValidatingPolicy). This improves observability but tells a user only that a policy failed, not why. It does not address debugging at the expression level.
- **Improve documentation and examples.** This helps authoring but gives no visibility into a specific evaluation.
- **A standalone policy simulation engine.** Most of its debugging value is subsumed by explainability, and it overlaps existing testing tooling.

The impact of not doing this is continued migration friction from the traditional policy types to the CEL types, since CEL policies remain meaningfully harder to debug than the policies they replace.

# Prior Art

- The cel-go REPL and the CEL playground both use state tracking to display intermediate evaluation results, which demonstrates that mapping AST node state to source text is practical.
- OPA exposes decision logs, and Gatekeeper's gator tool offers detailed constraint traces. Kyverno's CEL path currently offers neither at the sub-expression level.
- Kyverno's own playground renders policy results and could consume the same trace in a later iteration.

# Unresolved Questions

- The exact output schema for the structured (`-o json`) trace.
- Whether the runtime opt-in (phase 2) is best expressed as a policy annotation, a spec field, or a per-request mechanism.
- How much of the trace to expose for ImageValidatingPolicy, whose evaluation involves external verification steps beyond CEL.
- The performance budget that defines an acceptable overhead ceiling for the opt-in runtime path.

# CRD Changes (OPTIONAL)

Phase 1 (CLI) requires no CRD changes. Phase 2 (runtime opt-in) may introduce a debug field or annotation convention on the CEL policy types. The exact mechanism is listed under Unresolved Questions and would be specified before phase 2 implementation.
