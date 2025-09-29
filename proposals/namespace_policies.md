# Meta
[meta]: #meta
- Name: Namespaced CEL-Based Policies for Kyverno
- Start Date: 29th September 2025
- Update data (optional): 29th September 2025
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

Kyverno currently supports cluster-scoped CEL-based policy types—`ValidatingPolicy`, `MutatingPolicy`, `ImageValidatingPolicy`, `GeneratingPolicy`, and `DeletingPolicy`. This proposal introduces namespaced counterparts for all five, enabling namespace owners to define and manage policies within their own namespaces without cluster-wide permissions. The design covers CRDs, admission integration, RBAC, controller behavior, multi-tenancy isolation, documentation, and tests.

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

These kinds are proposed to avoid conflict with existing cluster-scoped kinds and to keep intent explicit. The API group remains `kyverno.io` with versioning aligned to the corresponding cluster-scoped CRDs (e.g., `v2alpha1`/`v2beta1`/`v2` as applicable during rollout). Each namespaced policy kind supports the same rule schema and status shapes as its cluster-scoped equivalent, with explicit mention that cross-namespace targets are not evaluated.

Key properties:
- Namespaced scope; no cross-namespace policy effects.
- Admission binding only triggers for resources whose `metadata.namespace` equals the policy namespace.
- Owners in the namespace can CRUD their policies via standard RBAC roles.
- Coexistence with cluster-scoped policies; deterministic application order defined to avoid surprises.

# Implementation

This section outlines the design and technical details required to implement namespaced CEL-based policies.

1) CRDs (schema and validation)
- Define five new CRDs with `scope: Namespaced`.
- Reuse rule schemas from cluster-scoped types for parity to the extent possible.
- Include `x-kubernetes-validations` (CEL) to guard scope-incompatible fields (e.g., disallow selectors referencing other namespaces if present), and to validate rule fields consistent with cluster-scoped counterparts.
- Ensure status conditions record evaluation counts and last transition times per namespace context.

2) Admission integration
- Register admission handlers that include namespace scoping checks prior to rule evaluation.
- Admission requests where `metadata.namespace` is empty (cluster-scoped objects) are not evaluated by namespaced policies.
- Admission ordering: enforce that cluster-scoped and namespaced policies can both evaluate; ordering follows existing Kyverno semantics (documented). If needed, define a tie-break strategy preferring deny from either scope.

3) Controller and evaluation logic
- Extend/instantiate evaluators to index namespaced policies by namespace for O(1) lookup per request.
- Watch namespaced policy resources and maintain per-namespace policy caches.
- Ensure that `GeneratingPolicy` namespaced variant only creates resources within the same namespace and forbids cross-namespace generation unless explicitly allowed by design (default: disallow).
- Ensure `DeletingPolicy` namespaced variant only targets resources within the same namespace.

4) RBAC integration
- Add `verbs: [get, list, watch, create, update, patch, delete]` for the five namespaced CRDs to common namespace roles (e.g., a new `kyverno:policy-editor` Role or extend guidance for `edit` ClusterRole via RoleBinding).
- Kyverno controller needs cluster-wide `list/watch` on the CRDs, but enforcement is scoped at evaluation time.
- Provide example `Role`/`RoleBinding` snippets for policy authors in a namespace.

5) Status, events, and metrics
- Surface per-policy status and counters scoped to the namespace.
- Emit Kubernetes events in the namespace upon validation failures and policy errors.
- Expose Prometheus metrics labeled by namespace and policy name to support observability and SLOs.

6) Documentation and examples
- Author a dedicated guide and examples showing usage for each namespaced policy kind.
- Cover policy authoring, policy lifecycle (create/update/delete), and debugging patterns.
- Provide Helm/kustomize snippets for opt-in enablement.

7) Testing
- Unit tests for policy evaluation scoping and conflict resolution with cluster-scoped policies.
- E2E tests: create namespaced policies across multiple namespaces; verify isolation, allow/deny behavior, mutation, generation, and deletion logic are confined to namespaces.
- Conformance tests ensuring parity with cluster-scoped equivalents for in-namespace resources.

## Link to the Implementation PR

TBD (to be added once implementation PR is opened).

# Migration (OPTIONAL)

No mandatory migration is required. Workloads can adopt namespaced policies gradually:
- Start by defining `Namespaced*` policies alongside existing cluster-scoped policies.
- Optionally decompose broad cluster-scoped policies into smaller namespaced ones to delegate ownership.
- Ensure potential overlaps are understood (documented ordering and conflict rules).

# Drawbacks

- Additional CRDs increase surface area and operator complexity.
- Potential for overlapping policy effects between cluster and namespaced scopes leading to user confusion without clear documentation.
- Slight increase in memory and CPU due to per-namespace policy caches and watches.

# Alternatives

- Reuse the same Kind names but vary CRD scope: not feasible due to name conflicts and user confusion.
- Introduce a single multipurpose Policy with a `scope` field toggling namespace vs cluster: reduces explicitness, complicates RBAC, and risks breaking existing users.
- Leverage `Policy`/`ClusterPolicy` legacy model: does not align with CEL policy families and would fragment user experience.

# Prior Art

- Kyverno historically provided `Policy` (namespaced) and `ClusterPolicy` (cluster-scoped) for non-CEL policies.
- Kubernetes native admission resources include namespaced vs cluster-scoped counterparts in various APIs to support multi-tenancy.
- Other policy engines (e.g., Gatekeeper) allow namespace-level constraint templates and constraints, demonstrating demand for delegated policy control.

# Unresolved Questions

- Exact Kind naming and API versioning: whether to use `Namespaced*` prefix or separate API group to avoid collisions. Recommendation here is `Namespaced*` kinds under `kyverno.io`.
- Admission ordering semantics when both scopes apply: final deterministic rule set and documentation.
- Cross-namespace references in rules: whether to be strictly forbidden or supported with explicit opt-in.
- Feature gates or flags for a phased rollout vs default enablement.

# CRD Changes (OPTIONAL)

Proposed new CRDs (all `scope: Namespaced`):
- `namespacedvalidatingpolicies.kyverno.io`
- `namespacedmutatingpolicies.kyverno.io`
- `namespacedimagevalidatingpolicies.kyverno.io`
- `namespacedgeneratingpolicies.kyverno.io`
- `namespaceddeletingpolicies.kyverno.io`

Each CRD mirrors the corresponding cluster-scoped schema. Example validation (illustrative):

```yaml
x-kubernetes-validations:
- rule: "has(self.spec.namespaceSelector) ? false : true"
  message: "Cross-namespace selectors are not supported in namespaced policies"
- rule: "self.spec.targetNamespace == null || self.spec.targetNamespace == metadata.namespace"
  message: "Targets must reside in the same namespace as the policy"
```

RBAC examples (illustrative):

```yaml
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: kyverno-namespaced-policy-editor
  namespace: example
rules:
- apiGroups: ["kyverno.io"]
  resources:
  - namespacedvalidatingpolicies
  - namespacedmutatingpolicies
  - namespacedimagevalidatingpolicies
  - namespacedgeneratingpolicies
  - namespaceddeletingpolicies
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
```

Controller permissions (cluster-wide list/watch for reconciliation) remain unchanged in scope but are extended to include the new CRDs.
