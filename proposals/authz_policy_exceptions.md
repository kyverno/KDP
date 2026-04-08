# Meta
[meta]: #meta
- Name: Policy Exceptions for Authorization (kyverno-authz)
- Start Date: 2026-04-08
- Author(s): @realshuting
- Supersedes: N/A

---

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

---

# Overview
[overview]: #overview

This KDP proposes adding exception support to kyverno-authz so operators can exclude selected requests from authorization policy evaluation. The design space has three valid approaches and this document evaluates all three. It does not assume that extending upstream `policies.kyverno.io` is required for MVP.

---

# Definitions
[definitions]: #definitions

- **PolicyException**: Upstream Kyverno resource in `policies.kyverno.io` used to exclude requests from policy enforcement.
- **Exception Matching**: Evaluating predicates to determine whether an exception applies to a request.
- **Evaluation Mode**: The runtime context used by kyverno-authz (`HTTP` or `Envoy`).
- **Binding CRD**: A separate authz-owned CRD that attaches authz-specific metadata (scope, expiry, approvals) to upstream exceptions.
- **CEL**: Common Expression Language used for declarative matching logic.

---

# Motivation
[motivation]: #motivation

1. Operators need temporary and permanent carve-outs for authz decisions.
2. Exceptions should be manageable from configurable sources, starting with in-cluster.
3. The solution should avoid unnecessary CRD churn while keeping room for governance features.
4. The solution should work consistently for both HTTP and Envoy modes.

Use cases:
1. Admin break-glass access.
2. Service migration windows.
3. Temporary allow for legacy clients.
4. Internal traffic exceptions with explicit scope.

Expected outcomes:
1. Requests matching an approved exception can bypass one or more target policies.
2. Exception behavior is observable and testable.
3. The API path is clear for both MVP and later governance enhancements.

---

# Proposal
[proposal]: #proposal

## Option Set

This KDP evaluates three approaches.

### Approach 1: Reuse upstream `PolicyException` as-is

Use `policies.kyverno.io/v1` or `v1beta1` `PolicyException` without adding fields. Authz-specific scope is encoded via labels/annotations and interpreted by kyverno-authz.

Example:
```yaml
apiVersion: policies.kyverno.io/v1beta1
kind: PolicyException
metadata:
  name: exempt-admin
  labels:
    authz/evaluation-modes: "HTTP,Envoy"
spec:
  policyRefs:
    - name: strict-authz-policy
      kind: ValidatingPolicy
  matchConditions:
    - name: is-admin
      expression: "object.user.username == 'admin'"
  reportResult: skip
```

### Approach 2: Upstream contribution to `PolicyException`

Propose generic fields (for example `scope`/`expiresAt`) in upstream `policies.kyverno.io/v1`.

Example direction:
```go
type PolicyExceptionSpec struct {
  // existing fields...
  Scope *ExceptionScope `json:"scope,omitempty"`
}
```

### Approach 3: Add authz-owned binding CRD

Keep upstream `PolicyException` unchanged and introduce `AuthzPolicyExceptionBinding` in an authz-owned API group.

Example:
```yaml
apiVersion: authzpolicies.kyverno.io/v1alpha1
kind: AuthzPolicyExceptionBinding
metadata:
  name: exempt-admin-binding
spec:
  exceptionRef:
    name: exempt-admin
  evaluationModes:
    - HTTP
    - Envoy
  expiresAt: "2026-12-31T00:00:00Z"
  reason: "Temporary migration window"
```

## Decision Criteria

1. Time to MVP.
2. Backward compatibility and operational risk.
3. Long-term API ownership and maintenance burden.
4. Governance features (expiry, approvals, ticketing).
5. User experience and cognitive load.

## Recommendation

Proposed phased recommendation:
1. **MVP**: Approach 1.
2. **If broad ecosystem value is demonstrated**: Approach 2 proposal upstream.
3. **If governance requirements appear**: Approach 3.

Rationale:
1. Delivers fastest path to user value.
2. Avoids premature custom CRD complexity.
3. Preserves future extensibility without forcing immediate upstream dependency.

---

# Implementation
[implementation]: #implementation

Implementation details are intentionally deferred until team design discussion completes.

This section will be updated after consensus on:
1. Selected approach for the first implementation phase.
2. Exception matching semantics and precedence.
3. Rollout plan and observability requirements.

## Link to the Implementation PR

TBD after design approval.

---

# Migration (OPTIONAL)
[migration]: #migration-optional

For Approach 1 MVP:
1. No breaking API changes in kyverno-authz policy resources.
2. Feature can be rolled out with `--enable-exceptions=false` by default.
3. Operators can pre-create exceptions before enabling enforcement.

If Approach 3 is adopted later:
1. Existing Approach 1 exceptions remain valid.
2. Bindings become additive metadata and control plane.

If Approach 2 is adopted later:
1. Migration depends on upstream field names and versioning guarantees.
2. Compatibility shims may be required for previously label-scoped behavior.

---

# Drawbacks
[drawbacks]: #drawbacks

---

# Alternatives
[alternatives]: #alternatives


---

# Prior Art
[prior-art]: #prior-art

---

# Unresolved Questions
[unresolved-questions]: #unresolved-questions
