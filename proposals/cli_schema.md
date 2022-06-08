# Kyverno Design Proposal - Schema for Kyverno CLI test command

- Name: Schema for Kyverno CLI test command
- Start Date: 2022-06-06
- Author(s): chipzoller

## Table of Contents

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

## Overview

[overview]: #overview

This proposal is to develop and implement a formal schema for Kyverno test manifests for use in the Kyverno CLI's `test` command. There are two parts:

1. The development of a formal schema
2. The "plumbing" of that schema back to existing and, potentially, new areas of the CLI to supply the necessary data

## Definitions

[definitions]: #definitions

None

## Motivation

[motivation]: #motivation

Test manifests today have no formalized structure and often require multiple files to declare expected test results (ex., `kyverno-test.yaml`, `values.yaml`, `user_info.yaml`, etc). Because there's no formalized structure, writing test cases is difficult because there's no linting or validation of the contents. This creates a poor user experience. Additionally, when adding new capabilities, it is currently not possible to version that structure which creates breaking changes for everyone. These changes will support anyone who wishes to write test cases for the `kyverno test` command. The outcome expected is a formalized, versioned schema with full validation similar to how Kubernetes and other systems work today.

## Proposal

This provides a high level overview of the feature.

A new schema is being proposed for the `test` command which will provide the necessary structure to account for all major current capabilities of Kyverno and close some issues related to problems. There are some high-level functional requirements that must be met.

1. The structure must be completely validated.
2. It must allow backwards compatibility with existing test manifests out in the wild.
   1. A test manifest which supplies the `apiVersion` and `kind` as specified in the proposed schema below is not subject to this backwards compatibility; only those in which these fields are absent should be subject to the existing logic.
3. All file names must be checked for existence and validity.
4. Objects in the `results[]` section should be validated against the rule type to determine if the necessary mandatory fields are present.
5. Policies which are of type `Policy` must specify Namespace.
6. There must not be duplicate names in resources 

```yaml
apiVersion: cli.kyverno.io/v1beta1
kind: KyvernoTest
metadata:
  name: test-check-nvidia-gpus
  labels:
    foolabel: foovalue
  annotations:
    fookey: foovalue
spec:
  policies:
    # support fully-qualified file paths (https://github.com/kyverno/kyverno/issues/2315)
    # wherever files are the value. Should be relative and absolute.
  - check-nvidia-gpu.yaml
  - relative/path/to/next/policy.yaml
  - /fully/qualified/path/to/next/policy.yaml
  resources:
  - resource.yaml
  - relative/path/to/next/resource.yaml
  - /fully/qualified/path/to/next/resource.yaml
  results:
  # results[] vary depending on the rule type being tested.
  # validate
  - policy: policyname01 # required
    rule: validaterulename # required
    # resources must support multiple resources, all of which
    # should be subject to the rest of the field values 
    # in this result. (https://github.com/kyverno/kyverno/issues/2857)
    resources: # required
    - resource01
    - resource02
    - resource03
    # namespace should set the namespace name for the resource(s), if not present,
    # but otherwise should override the Namespace name if present. If the policy
    # being tested is a `Policy` (Namespaced), and if the namespace is not defined
    # in the resource AND not defined here, it should assume the value of `default`
    # to be consistent with how Kubernetes treats resources (i.e., creating a resource
    # with no metadata.namespace field implies the default namespace) (https://github.com/kyverno/kyverno/issues/4059)
    # (https://github.com/kyverno/kyverno/issues/4087)
    namespace: foonamespace # optional
    kind: Kind # required
    result: myresult # required
  # mutate
  - policy: nextname
    rule: mutaterulename
    resources:
    - resource01
    patchedResource: mypatchedresource.yaml
    kind: Kind
    result: myresult
  # generate
  - policy: nextname
    rule: generaterulename
    resources:
    - resource01
    generatedResource: mypatchedresource.yaml
    kind: Kind
    result: myresult
  # verifyImages. Must support attestations (https://github.com/kyverno/kyverno/issues/2866)
  - policy: verify-attestations01
    rule: verifyimagesrulename
    resources:
    - resource01
    kind: Kind
    result: myresult
  variables:
    # values that are in the global object apply to all policies, rules, and resources
    # being tested. If a variable of the same name is found elsewhere (at a more granular level)
    # its value at that level should trump the global value. The same rules apply for values
    # here as elsewhere. See the variables.policies[].rules[].values for examples and comments.
    # The hierarchy should be global => rule => resource.
    global:
      request.operation: CREATE
    # policies[] is an array of objects. The name must be found in those specified in the files
    # named in spec.resources[].
    policies:
    - name: policyname01
      rules:
      # the rule name must correspond to a rule found in the parent policy.
      - name: rulename01
        # values themselves do not need to be validated other than for YAML syntax
        values:
          # values must support all types (string, int, float, bool, object, etc.)
          imageSize: "3Gi"
          # values may be in JMESPath expression format (dot-separated fields)
          namespacefilters.data.exclude: "[\"default\", \"test\"]"
          # a given value must also support an object like below
          imageData:
            labels:
              org.opencontainers.image.source: "https://github.com/kyverno/kyverno-examples"
              org.opencontainers.image.foo: "bar"
        # if a policy is written with a namespaceSelector, it falls under the rule.
        # See https://github.com/kyverno/kyverno/tree/main/test/cli/test/any-namespaceSelector
        namespaceSelector:
        - name: test1
          labels:
            foo.com/managed-state: managed
      # values which pertain to resources only, commonly request.operation
      # can be specified under this resources object. The name must match some
      # name that is found in spec.resources[].
      resources:
      - name: nginx-demo2
        values:
          request.operation: UPDATE
        # the userInfo field holds info on who/what is responsible for the resource. It is grouped
        # here, just like some values, because they accompany the resource itself and not
        # the rule. See output of `kubectl explain policy.spec.rules.match.subjects` for details
        userInfo:
          clusterRoles:
          - cluster-admin
      - name: nginx-demo3
        userInfo:
          roles:
          - nsoperator
      - name: nginx-demo4
        userInfo:
          subjects:
          - name: chip
            kind: User
          - name: devops
            kind: Group
      - name: nginx-demo5
        userInfo:
          subjects:
          - name: kyverno
            kind: ServiceAccount
            # Namespace is only applicable to ServiceAccount and should produce
            # a validation error if present on User or Group.
            namespace: kyverno
    - name: very-attestations01
      rules:
      - name: verifyimagesrulename
        # there will be an attestations object under rules[] which allows specifying variables
        # related to verifyImage rule types that need attestations. The below is a work in progress and
        # will need community input.
        attestations:
        - predicateType: https://example.com/CycloneDX/v1
          predicateResource: my-sbom.json
        - predicateType: https://example.com/CodeReview/v1
          predicateResource: my-codereview.json
        values:
          foo: bar
```

## Implementation

The implementation will be done by an LFX mentee supervised by one or more Kyverno maintainers with assistance provided, as necessary, by the broader contributor community.

The following issues should be either partially or completely addressed by the implemention.

https://github.com/kyverno/kyverno/issues/2323
https://github.com/kyverno/kyverno/issues/2315
https://github.com/kyverno/kyverno/issues/2302
https://github.com/kyverno/kyverno/issues/2857
https://github.com/kyverno/kyverno/issues/2945
https://github.com/kyverno/kyverno/issues/3271
https://github.com/kyverno/kyverno/issues/4059
https://github.com/kyverno/kyverno/issues/3915 (hopeful)

### Link to the Implementation PR

TBD

## Migration (OPTIONAL)

This section should document breaks to public API and breaks in compatibility due to this KDP's proposed changes. In addition, it should document the proposed steps that one would need to take to work through these changes.

## Drawbacks

There are no drawbacks to this proposal. It is a tablestakes feature and should have been implemented from day one.

Risks are that it fails to be implemented at all or with sufficient quality that it cannot be used. A mitigation for this risk is to simply rely on the current capabilities and delay the release of this functionality.

## Alternatives

N/A

## Prior Art

N/A

## Unresolved Questions

- The schema needs collaboration from the maintainers to account for other gaps not presently accounted for, specifically any `verifyImages` rule capabilities.

## CRD Changes (OPTIONAL)

This proposal should only affect the internal operation of the CLI. No CRD need be registered in a cluster to implement or execute this design.
