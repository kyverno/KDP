# Meta
[meta]: #meta
- Name: Namespaced CEL-Based Policies for Kyverno
- Start Date: 29th September 2025
- Update date: 29th September 2025
- Author(s): slashexx 
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

Kyverno currently supports cluster-scoped CEL-based policy types - `ValidatingPolicy`, `MutatingPolicy`, `ImageValidatingPolicy`, `GeneratingPolicy`, and `DeletingPolicy`. This proposal introduces namespaced counterparts for all five, enabling namespace owners to define and manage policies within their own namespaces without cluster-wide permissions. The design covers CRDs, admission integration, RBAC, controller behavior, multi-tenancy isolation, documentation, and tests.

# Definitions
[definitions]: #definitions

- **CEL**: Common Expression Language used by Kubernetes for schema and admission validations.
- **CEL-based policies**: Kyverno policies expressed via CEL rules and compiled into admission behavior.
- **Namespaced CEL policies**: Policy resources with `scope: Namespaced` that apply only to resources within their namespace.
- **Policy owner**: A user or team with write permissions in a namespace that can create/maintain namespaced policies for that namespace.

# Motivation
[motivation]: #motivation

- Enable decentralized policy management by namespace owners without requiring cluster-admin privileges.
- Improve security posture and reduce blast radius by limiting policy scope to namespaces.
- Provide better RBAC alignment and lifecycle management for teams operating in multi-tenant clusters.
- Maintain parity with existing cluster-scoped CEL policies while narrowing enforcement scope.

Expected outcomes:
- Implement namespaced versions of all five CEL-based policy types.
- Ensure proper RBAC integration for namespace-scoped policy management.
- Provide comprehensive documentation and examples.
- Add conformance and e2e tests validating namespaced behavior.

# Proposal

Introduce a set of namespaced CRDs—one per existing CEL-based policy type—whose enforcement is restricted to the containing namespace. The namespaced policies mirror the cluster-scoped counterparts in structure and behavior, except for:
- Scope-limited matching and enforcement: only resources in the same namespace are evaluated.
- RBAC aligned with namespace ownership roles (e.g., `edit` in that namespace).
- Controller reconciliation and admission registration reflect namespace scoping, avoiding cluster-wide impact.

New resource kinds (proposed):
- `NamespacedValidatingPolicy`
- `NamespacedMutatingPolicy`
- `NamespacedImageValidatingPolicy`
- `NamespacedGeneratingPolicy`
- `NamespacedDeletingPolicy`

These kinds are proposed to avoid conflict with existing cluster-scoped kinds and to keep intent explicit. The API group for CEL-based policies is `policies.kyverno.io` (as implemented today), currently at `v1alpha1`. Each namespaced policy kind supports the same rule schema and status shapes as its cluster-scoped equivalent, with explicit mention that cross-namespace targets are not evaluated.

Key properties:
- Namespaced scope; no cross-namespace policy effects.
- Admission binding only triggers for resources whose `metadata.namespace` equals the policy namespace.
- Owners in the namespace can CRUD their policies via standard RBAC roles.
- Coexistence with cluster-scoped policies; deterministic application order defined to avoid surprises.

# Implementation

This section outlines the design and technical details required to implement namespaced CEL-based policies.

1) CRDs (schema and validation)
- Define five CRDs which are namespace scoped: `NamespacedValidatingPolicy`, `NamespacedMutatingPolicy`, `NamespacedImageValidatingPolicy`, `NamespacedGeneratingPolicy`, and `NamespacedDeletingPolicy` under `policies.kyverno.io`.
- Reuse rule schemas from cluster-scoped types for parity to the extent possible.
- Add a common `Like` interface for both cluster-scoped and namespace-scoped policy versions which will be used throughout.

2) Admission integration
- Register admission handlers that include namespace scoping checks prior to rule evaluation.
- Admission requests where `metadata.namespace` is empty (cluster-scoped objects) are not evaluated by namespaced policies.
- Admission ordering: enforce that cluster-scoped and namespaced policies can both evaluate; ordering follows existing Kyverno semantics (documented). If needed, define a tie-break strategy preferring deny from either scope.

3) Controller and evaluation logic
- Use common shared controller for both versions which watch the namespced and cluster-scoped variants.
- Watch namespaced policy resources and maintain per-namespace policy caches.
- Ensure that `GeneratingPolicy` namespaced variant only creates resources within the same namespace and forbids cross-namespace or cluster-scoped generation.
- Ensure `DeletingPolicy` namespaced variant only targets resources within the same namespace (already true today).

4) RBAC integration
- Add `verbs: [get, list, watch, create, update, patch, delete]` for all namespaced CRDs to common namespace roles.
- Kyverno controller needs cluster-wide `list/watch` on the CRDs, but enforcement is scoped at evaluation time.
- Reflect actual shortNames for existing CRDs: `nvpol` (NamespacedValidatingPolicy) and `ndpol` (NamespacedDeletingPolicy).

5) Documentation and examples
- Author a dedicated guide and examples showing usage for each namespaced policy kind.
- Cover policy authoring, policy lifecycle (create/update/delete), and debugging patterns.
- Provide Helm/kustomize snippets for opt-in enablement.

6) Testing
- Unit tests for policy evaluation scoping and conflict resolution with cluster-scoped policies.
- E2E tests: create namespaced policies across multiple namespaces; verify isolation, allow/deny behavior, mutation, generation, and deletion logic are confined to namespaces.
- Conformance tests ensuring parity with cluster-scoped equivalents for in-namespace resources.

## Link to the Implementation PR

TBD (to be added once implementation PR is opened).

# CRD Changes

Proposed CRDs under `policies.kyverno.io` (all `scope: Namespaced`):
- `namespacedvalidatingpolicies.policies.kyverno.io` (shortName: `nvpol`)
- `namespacedmutatingpolicies.policies.kyverno.io` (shortName: `nmpol`)
- `namespacedimagevalidatingpolicies.policies.kyverno.io` (shortName: `nivpol`)
- `namespacedgeneratingpolicies.policies.kyverno.io` (shortName: `ngpol`)
- `namespaceddeletingpolicies.policies.kyverno.io` (shortName: `ndpol`)

Each CRD mirrors the corresponding cluster-scoped schema.

RBAC examples (illustrative):

```yaml
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: kyverno-namespaced-policy-editor
  namespace: example
rules:
- apiGroups: ["policies.kyverno.io"]
  resources:
  - namespacedvalidatingpolicies
  - namespacedmutatingpolicies
  - namespacedimagevalidatingpolicies
  - namespacedgeneratingpolicies
  - namespaceddeletingpolicies
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
```
