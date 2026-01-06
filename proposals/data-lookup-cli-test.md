# Meta
[meta]: #meta
- Name: Data Lookup Support for CLI Test Command
- Start Date: 2025-12-16
- Author(s): @realshuting

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

This feature enables CLI testing of API server calls for all Kyverno CEL libraries by introducing `clusterResources` and `httpResponses` attributes in `kyverno-test.yaml` files. These attributes allow users to specify test data files that populate a Kubernetes fake client and provide mock HTTP responses, enabling comprehensive testing of policies that use CEL libraries for dynamic lookups against Kubernetes APIs and HTTP endpoints.

# Definitions
[definitions]: #definitions

- **CLI Test Command**: The `kyverno test` command that validates policies against test cases without requiring a running cluster
- **CEL Libraries**: Common Expression Language (CEL) extensions that provide [additional functions](https://kyverno.io/docs/policy-types/cel-libraries/) for Kyverno policies, including resource, HTTP, user, image, imageData, and globalContext libraries
- **Fake Context Provider**: A testing implementation that provides static, predefined data instead of making real external calls
- **Real Lookup**: Dynamic data retrieval from actual external systems (Kubernetes API, HTTP endpoints, registries, etc.) during policy evaluation
- **ClusterResource CRD**: A new CRD type (`cli.kyverno.io/v1alpha1.ClusterResource`) that provides CRD definitions and resource manifests to populate the fake Kubernetes client for isolated testing
- **HTTPResponse CRD**: A new CRD type (`cli.kyverno.io/v1alpha1.HTTPResponse`) that provides mock HTTP response data for testing HTTP library functions without making real network calls

# Motivation
[motivation]: #motivation

- **Why should we do this?** The CLI test command currently uses fake context providers that return static, mocked data for all CEL library operations. This prevents comprehensive testing of policies that depend on real external systems, creating a significant gap between test and production environments where policies may behave differently with actual data sources.

- **What use cases does it support?**
  - Testing policies that query Kubernetes APIs via the [resource library](https://kyverno.io/docs/policy-types/cel-libraries/#resource-library)
  - Validating HTTP(S) endpoint responses using the [HTTP library](https://kyverno.io/docs/policy-types/cel-libraries/#http-library)
  - Testing user authentication logic with the [user library](https://kyverno.io/docs/policy-types/cel-libraries/#user-library)
  - Validating image metadata from real registries using the [imageData library](https://kyverno.io/docs/policy-types/cel-libraries/#imagedata-library)
  - Testing policies with real global context data via the [globalContext library](https://kyverno.io/docs/policy-types/cel-libraries/#globalcontext-library)
  - CI/CD pipelines that need to validate complete policy behavior against actual infrastructure

- **What is the expected outcome?** Users will be able to configure test data through `httpResponses` and `clusterResources` attributes in their `kyverno-test.yaml` files, enabling comprehensive testing of policies that use CEL libraries with controlled, predictable test data.

# Proposal

This feature enables comprehensive testing of CEL library functions by introducing test data configuration attributes in the `kyverno-test.yaml` file. Users can specify `clusterResources` and `httpResponses` to provide test data for resource and HTTP library testing.

## Test File Configuration

Add two new attributes to the `kyverno-test.yaml` file for configuring test data:

```yaml
apiVersion: cli.kyverno.io/v1alpha1
kind: Test
metadata:
  name: test-example

context: context.yaml

# List of ClusterResource CRD files to populate the fake Kubernetes client
clusterResources:
  - cluster-resources-1.yaml
  - cluster-resources-2.yaml

# List of HTTPResponse CRD files for HTTP library testing
httpResponses:
  - http-response-1.yaml
  - http-response-2.yaml

policies:
  - policy.yaml
resources:
  - resources/test-resource.yaml
results:
  - policy: test-policy
    rule: test-rule
    kind: Pod
    resources:
      - test-resource
    clusterResources:
      - resource-from-cluster-resources-1
    result: pass
```

## Behavior Changes

**Default behavior (when attributes are not specified):**
- All CEL library functions return mocked/static data from context files
- Resource lookups use existing context file data if available

**With `clusterResources` attribute:**
- **Resource Library**: Loads `ClusterResource` from specified files into fake Kubernetes client
- Enables testing with controlled, predictable resource data
- No cluster connection required
- Supports both CRD definitions and resource instances
- Resources from `clusterResources` files are available for `resource.Get()`, `resource.List()`, and `resource.Post()` calls

**With `httpResponses` attribute:**
- **HTTP Library**: Uses `HTTPResponse` CRDs from specified files for mock HTTP responses
- Allows testing HTTP-dependent policies without making real network calls
- `http.Get()`, `http.Post()` and other HTTP functions use the provided mock responses

## Test Data Provisioning

Test data is provided through separate CRD files referenced in the `kyverno-test.yaml` file:

- **`clusterResources` attribute**: List of file paths containing `ClusterResource` CRDs
  - These files define Kubernetes resources and CRD definitions to populate the fake client
  - Used for resource library testing with isolated, controlled data
  - Supports both CRD definitions and resource instances
- **`httpResponses` attribute**: List of file paths containing `HTTPResponse` CRDs
  - These files provide mock HTTP response data for HTTP library testing
  - Allows testing HTTP-dependent policies without network calls

Example `kyverno-test.yaml`:
```yaml
apiVersion: cli.kyverno.io/v1alpha1
kind: Test
metadata:
  name: test-example

context: context.yaml

clusterResources:
  - cluster-resources.yaml

httpResponses:
  - http-mocks.yaml

policies:
  - policy.yaml
resources:
  - resources/test-pod.yaml
results:
  - policy: test-policy
    rule: test-rule
    kind: Pod
    resources:
      - test-pod
    clusterResources:
      - resource-from-cluster-resources
    result: pass
```

Example `cluster-resources.yaml`:
```yaml
apiVersion: cli.kyverno.io/v1alpha1
kind: ClusterResource
metadata:
  name: test-cluster-resources
spec:
  crds:
    - crds/computeclass-crd.yaml
  resources:
    - apiVersion: cloud.google.com/v1
      kind: ComputeClass
      metadata:
        name: default
      spec:
        priorities:
          - spot: true
```

Example `http-mocks.yaml`:
```yaml
apiVersion: cli.kyverno.io/v1alpha1
kind: HTTPResponse
metadata:
  name: mock-api-response
spec:
  response:
    statusCode: 200
    body: '{"status": "success"}'
```

## Key Design Decisions

1. **Declarative Configuration**: Test data is configured declaratively in `kyverno-test.yaml` rather than through CLI flags, making test configurations explicit and version-controlled
2. **File-based Test Data**: Test data is provided through separate CRD files referenced in the test configuration, enabling reuse and organization
3. **Isolated Resource Testing**: Resource lookups use `ClusterResource` CRDs to populate fake client for predictable, isolated testing
4. **Structured Test Data**: New CRD types (`ClusterResource`, `HTTPResponse`) provide clear separation of concerns for different test data types
5. **Backward Compatible**: Existing test files without these attributes continue to work with default behavior
6. **Scoped Implementation**: Initially target resource and HTTP(S) libraries, with extensibility for future libraries

# Implementation

## Link to the Implementation PR


# Migration (OPTIONAL)

This feature is designed to be backward compatible with no breaking changes.

## Backward Compatibility

- **No Breaking Changes**: Existing test files without `clusterResources` and `httpResponses` attributes work unchanged
- **Optional Attributes**: Both `clusterResources` and `httpResponses` are optional attributes
- **Safe Defaults**: When attributes are not specified, existing behavior is maintained (mocked data from context files)

# Drawbacks

- **File Management**: Users need to maintain separate CRD files for test data, which adds some organizational overhead
- **Test Data Synchronization**: Test data in `clusterResources` and `httpResponses` files must be kept in sync with actual system behavior
- **Limited Real-world Testing**: This approach uses mocked data, so it doesn't test against real external systems (HTTP endpoints, registries, etc.)
- **File Path Management**: Relative file paths in `clusterResources` and `httpResponses` must be correctly specified relative to the test file location

# Alternatives

# Prior Art

# Unresolved Questions


# CRD Changes (OPTIONAL)

## New CRD Types

This proposal introduces two new CRD types in the `cli.kyverno.io/v1alpha1` API group for providing test data to the CLI:

### ClusterResource

The `ClusterResource` CRD allows users to specify Kubernetes resources and CRD definitions that will be loaded into the fake Kubernetes client for isolated testing.

```yaml
apiVersion: cli.kyverno.io/v1alpha1
kind: ClusterResource
metadata:
  name: test-resources
spec:
  crds:
    - path/to/crd-definition.yaml
  resources:
    - apiVersion: v1
      kind: ConfigMap
      metadata:
        name: test-config
        namespace: default
      data:
        key: value
```

**Fields:**
- `crds` ([]string): List of file paths or inline CRD definitions to register with the fake client
- `resources` ([]unstructured.Unstructured): List of Kubernetes resource manifests to load into the fake client

**Use Cases:**
- Populating the fake Kubernetes client with test data for resource library testing
- Providing CRD definitions required for custom resource testing
- Enabling isolated testing without cluster connectivity

### HTTPResponse

The `HTTPResponse` CRD allows users to provide mock HTTP response data for testing HTTP library functions without making real network calls.

```yaml
apiVersion: cli.kyverno.io/v1alpha1
kind: HTTPResponse
metadata:
  name: mock-http-response
spec:
  response:
    statusCode: 200
    body: '{"status": "ok"}'
    headers:
      Content-Type: application/json
```

**Fields:**
- `response` (*http.Response): HTTP response data including status code, headers, and body

**Use Cases:**
- Mocking HTTP endpoint responses for HTTP library testing
- Testing policies that depend on external HTTP services
- Enabling offline testing of HTTP-dependent policies

## Context CRD Updates

The existing `Context` CRD (`cli.kyverno.io/v1alpha1.Context`) is updated to support references to `ClusterResource` and `HTTPResponse` resources, while the deprecated `resources` field in `ContextSpec` is replaced by the more structured `ClusterResource` approach.