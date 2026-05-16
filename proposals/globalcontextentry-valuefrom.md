# Meta

- Name: valueFrom Support for APICall Service Headers and CABundle
- Start Date: 2026-05-17
- Author(s): jakharmonika364
- Supersedes: N/A

# Table of Contents

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

This proposal adds `valueFrom` support to HTTP headers and `caBundle` in `ServiceCall` API calls, allowing sensitive values to be sourced from Kubernetes Secrets and ConfigMaps rather than being embedded directly in the resource spec. This applies to `GlobalContextEntry`, `ClusterPolicy`, and namespaced `Policy` context entries.

# Definitions

- **ServiceCall**: The `spec.apiCall.service` block in a `GlobalContextEntry` or policy context entry that configures an external HTTP API call.
- **GlobalContextEntry**: A cluster-scoped Kyverno resource that caches data fetched from external APIs for use across policies.
- **valueFrom**: A reference mechanism (following Kubernetes conventions) that allows a field value to be sourced from a Secret or ConfigMap key rather than specified inline.
- **secretKeyRef**: A reference to a specific key within a Kubernetes Secret.
- **configMapKeyRef**: A reference to a specific key within a Kubernetes ConfigMap.

# Motivation

- Currently, `ServiceCall` requires sensitive values like auth tokens and CA bundles to be embedded directly in the `GlobalContextEntry` or policy spec. This is a security concern as credentials are exposed in cluster resources.
- Users need a way to follow the principle of least exposure - secrets should live in Secrets, not in policy specs.
- The Kubernetes ecosystem has an established pattern for this (`secretKeyRef`/`configMapKeyRef` in pod env vars) that users already know and trust.
- Expected outcome: users can reference Secrets and ConfigMaps for header values and CA bundles, keeping credentials out of policy specs while keeping the API familiar.

# Proposal

Add a `valueFrom` field to `HTTPHeader` and a `caBundleFrom` field to `ServiceCall`, allowing values to be sourced from Secrets or ConfigMaps.

### HTTPHeader with valueFrom

Currently:
```yaml
headers:
  - key: Authorization
    value: "Basic dXNlcjpwYXNz"
```

Proposed:
```yaml
headers:
  - key: Authorization
    valueFrom:
      secretKeyRef:
        name: api-credentials
        namespace: kyverno
        key: basic-auth
```

### caBundle from ConfigMap

Currently:
```yaml
service:
  url: "https://my-api.example.com/data"
  caBundle: |-
    -----BEGIN CERTIFICATE-----
    MIICxxx...
    -----END CERTIFICATE-----
```

Proposed:
```yaml
service:
  url: "https://my-api.example.com/data"
  caBundleFrom:
    configMapKeyRef:
      name: corporate-ca
      namespace: kyverno
      key: ca-bundle.crt
```

### Full GlobalContextEntry example

```yaml
apiVersion: kyverno.io/v2alpha1
kind: GlobalContextEntry
metadata:
  name: api-data
spec:
  apiCall:
    method: GET
    service:
      url: "https://my-api.example.com/data"
      headers:
        - key: Authorization
          valueFrom:
            secretKeyRef:
              name: api-credentials
              namespace: kyverno
              key: basic-auth
        - key: X-Custom-Header
          valueFrom:
            configMapKeyRef:
              name: api-config
              namespace: kyverno
              key: custom-header-value
      caBundleFrom:
        configMapKeyRef:
          name: corporate-ca
          namespace: kyverno
          key: ca-bundle.crt
```

### Validation rules

- Exactly one of `value` or `valueFrom` must be set per header.
- Exactly one of `secretKeyRef` or `configMapKeyRef` must be set in a `valueFrom`/`caBundleFrom`.
- At most one of `caBundle` or `caBundleFrom` may be set.
- For **namespaced policies**: the `namespace` in the selector must match the policy namespace (or be omitted, defaulting to the policy namespace). Cross-namespace references are not allowed.
- For **cluster-scoped resources** (GlobalContextEntry, ClusterPolicy): `namespace` is required in the selector (no default exists).

# Implementation

## New API types (`api/kyverno/v1/common_types.go`)

Custom selector types are defined (not reusing `corev1.SecretKeySelector`) because Kyverno needs a `Namespace` field - essential for cluster-scoped resources like `GlobalContextEntry` that have no inherent namespace.

```go
type SecretKeySelector struct {
    Name      string `json:"name"`
    Namespace string `json:"namespace,omitempty"`
    Key       string `json:"key"`
}

type ConfigMapKeySelector struct {
    Name      string `json:"name"`
    Namespace string `json:"namespace,omitempty"`
    Key       string `json:"key"`
}

type HTTPHeaderSource struct {
    SecretKeyRef    *SecretKeySelector    `json:"secretKeyRef,omitempty"`
    ConfigMapKeyRef *ConfigMapKeySelector `json:"configMapKeyRef,omitempty"`
}
```

Updated `HTTPHeader`:

```go
type HTTPHeader struct {
    Key       string            `json:"key"`
    Value     string            `json:"value,omitempty"`
    ValueFrom *HTTPHeaderSource `json:"valueFrom,omitempty"`
}
```

Updated `ServiceCall`:

```go
type ServiceCall struct {
    URL          string            `json:"url"`
    Headers      []HTTPHeader      `json:"headers,omitempty"`
    CABundle     string            `json:"caBundle,omitempty"`
    CABundleFrom *HTTPHeaderSource `json:"caBundleFrom,omitempty"`
}
```

## Interface changes

**`pkg/engine/api/contextloader.go`** - widen `ContextLoader.Load` from `RawClient` to `Client`:

Every call site already passes a full `engineapi.Client`; the narrowing to `RawClient` was artificial. Widening it gives downstream code access to `GetResource` without workarounds.

**`pkg/engine/apicall/client.go`** - add `GetResource` to `ClientInterface`:

```go
type ClientInterface interface {
    RawAbsPath(ctx context.Context, path string, method string, dataReader io.Reader) ([]byte, error)
    GetResource(ctx context.Context, apiVersion, kind, namespace, name string, subresources ...string) (*unstructured.Unstructured, error)
}
```

**`pkg/engine/apicall/executor.go`** - add `policyNamespace` + `resolveValueFrom`:

- `NewExecutor` gains `policyNamespace string` parameter.
- `resolveValueFrom` resolves a `*HTTPHeaderSource` to a string:
  - **Secret**: fetches via `GetResource`, reads from `data` map, base64-decodes (Kubernetes always stores Secret data as base64 in the API).
  - **ConfigMap**: fetches via `GetResource`, reads from `data` map, returns plain string (no base64 decode).
- Namespace scoping logic:
  - Namespaced policy (`policyNamespace != ""`): omitted namespace defaults to policy namespace; explicit namespace must match.
  - Cluster-scoped (`policyNamespace == ""`): namespace is required; any namespace is allowed.

## Files changed

| File | Change |
|------|--------|
| `api/kyverno/v1/common_types.go` | Add new types; update `HTTPHeader`, `ServiceCall` |
| `api/kyverno/v1/zz_generated.deepcopy.go` | Regenerated via `make generate` |
| `pkg/engine/api/contextloader.go` | `RawClient` → `Client` in `ContextLoader.Load` |
| `pkg/engine/apicall/client.go` | Add `GetResource` to `ClientInterface` |
| `pkg/engine/apicall/executor.go` | Add `policyNamespace`, `resolveValueFrom`; update `addHTTPHeaders`, `buildHTTPClient` |
| `pkg/engine/apicall/apiCall.go` | Pass `policyNamespace` to `NewExecutor`; client type update |
| `pkg/engine/context/loaders/apicall.go` | `engineapi.RawClient` → `engineapi.Client`; pass to `apicall.New` |
| `pkg/engine/factories/contextloaderfactory.go` | `RawClient` → `Client` in `Load` and `newLoader` |
| `pkg/globalcontext/externalapi/entry.go` | Client type update; pass `policyNamespace: ""` to `NewExecutor` |
| Test mocks + new tests | Update mocks; add 11 new test cases |
| CRD YAMLs | Regenerated via `make generate` |

## Link to the Implementation PR

TBD

# Migration (OPTIONAL)

- The `HTTPHeader.Value` field changes from required to `omitempty`. Existing specs with `value: ""` will still parse correctly - this is backward compatible.
- The `ServiceCall.CABundle` field gains `omitempty`. Existing specs are unaffected.
- No changes to existing behavior when `valueFrom`/`caBundleFrom` are not used.

# Drawbacks

- Resolving secrets/configmaps at API call time adds Kubernetes API calls on every `ServiceCall` execution (including the periodic refresh for `GlobalContextEntry`). This is a minor performance overhead.
- RBAC must grant Kyverno read access to the referenced Secrets and ConfigMaps. This is an operational concern users must manage.

# Alternatives

## Reuse `corev1.SecretKeySelector` / `corev1.ConfigMapKeySelector`

Kubernetes core already defines these selector types. However, they embed `LocalObjectReference` which has only a `Name` field - no `Namespace`. For cluster-scoped resources like `GlobalContextEntry`, a `Namespace` field is required. Custom types are necessary.

## Separate `ResourceResolver` interface (instead of widening `ContextLoader.Load`)

A narrower `ResourceResolver` interface (only `GetResource`) could be injected into the executor separately, avoiding any change to `ContextLoader.Load`. However, this requires a type assertion (`RawClient` → `ResourceClient`) in the factory, which is fragile and creates a silent failure mode. Widening `ContextLoader.Load` is cleaner.

## Pre-resolve headers before executor

Resolve `valueFrom` references in `apiCall.Fetch()` and `externalapi/entry.go` before calling `executor.Execute`, keeping the executor unaware of `valueFrom`. This creates two resolution sites with duplicated logic and still requires `GetResource` access at both locations.

# Prior Art

- **Kubernetes pod env vars**: `secretKeyRef`/`configMapKeyRef` in `EnvVarSource` - the direct inspiration for this proposal's API shape.
- **Kyverno ConfigMap context entries**: Kyverno already supports loading data from ConfigMaps into policy context via `spec.context[].configMap`. This proposal extends a similar pattern to the HTTP layer.

# Unresolved Questions

- Should validation (mutual exclusion of `value`/`valueFrom`, `caBundle`/`caBundleFrom`) also be enforced at admission time via kubebuilder CEL markers, or is runtime validation sufficient? Runtime validation is proposed for now; CEL markers can be added later.
- Should caching of resolved secret/configmap values be considered to reduce API calls during high-frequency `GlobalContextEntry` refreshes? Out of scope for this proposal.

# CRD Changes (OPTIONAL)

New fields added to the `ServiceCall` schema (present wherever `APICall.service` appears — `GlobalContextEntry`, `ClusterPolicy`, `Policy`, `CleanupPolicy`):

```yaml
headers:
  items:
    properties:
      key:
        type: string
      value:
        type: string
      valueFrom:
        properties:
          configMapKeyRef:
            properties:
              key:
                type: string
              name:
                type: string
              namespace:
                type: string
            required: [name, key]
            type: object
          secretKeyRef:
            properties:
              key:
                type: string
              name:
                type: string
              namespace:
                type: string
            required: [name, key]
            type: object
        type: object
    required: [key]
    type: object
caBundleFrom:
  properties:
    configMapKeyRef:
      properties:
        key:
          type: string
        name:
          type: string
        namespace:
          type: string
      required: [name, key]
      type: object
    secretKeyRef:
      properties:
        key:
          type: string
        name:
          type: string
        namespace:
          type: string
      required: [name, key]
      type: object
  type: object
```

Full CRD YAML will be regenerated via `make generate`.
