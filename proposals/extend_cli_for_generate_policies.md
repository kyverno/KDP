# Kyverno Design Proposal - Extend CLI test command for generate policies

Author: [Shubham Nazare](https://github.com/shubham4443)

PR: [#3456](https://github.com/kyverno/kyverno/pull/3456)

## Contents

- [Kyverno Design Proposal - Extend CLI test command for generate policies](#kyverno-design-proposal---extend-cli-test-command-for-generate-policies)
  * [Introduction](#introduction)
  * [Proposal](#proposal)
    + [API Design Changes](#api-design-changes)
    + [Method to test data object](#method-to-test-data-object)
      - [Example: Add Network Policy](#example--add-network-policy)
    + [Method to test clone object](#method-to-test-clone-object)
      - [Example: Sync Secrets](#example--sync-secrets)
    + [Namespaced Policies](#namespaced-policies)
      - [Example: Policy Disruption Budget](#example--policy-disruption-budget)
  * [Implementation details](#implementation-details)
    + [Mock Generate-Request Controller](#mock-generate-request-controller)
    + [Building the result](#building-the-result)

<small><i><a href='http://ecotrust-canada.github.io/markdown-toc/'>Table of contents generated with markdown-toc</a></i></small>


## Introduction

Prior to this proposal, only validate and mutate policies were covered by the kyverno test CLI command. Generate policies were skipped by the CLI. This document proposes an approach for the test command to cover generate policies as well.

## Proposal

When using a generate rule, the target resource can be either an existing resource defined within Kubernetes, or a new resource defined in the rule itself. When the target resource is a pre-existing resource the clone object is used. When the target resource is a new resource defined within the manifest of the rule, the data object is used. Both rules are mutually exclusive, and only one will be specified in a rule. Both objects require a different approach for testing through CLI command.

### API Design Changes

The test file will take 2 new fields - generatedResource and cloneSourceResource.

```go

type TestResults struct {
	Policy              string              `json:"policy"`
	Rule                string              `json:"rule"`
	Result              report.PolicyResult `json:"result"`
	Status              report.PolicyResult `json:"status"`
	Resource            string              `json:"resource"`
	Kind                string              `json:"kind"`
	Namespace           string              `json:"namespace"`
	PatchedResource     string              `json:"patchedResource"`
	AutoGeneratedRule   string              `json:"auto_generated_rule"`
	GeneratedResource   string              `json:"generatedResource"`
	CloneSourceResource string              `json:"cloneSourceResource"`
}
```

### Method to test data object

The method is similar to the testing for mutate policies. User will provide the desired resource manifest file as generatedResource in the results array of the test file. This manifest file will be compared with the Kyverno generated resource and accordingly the test will display the result as pass or fail.

Additionally, like in the case of testing mutate policies, we provide the resource on which we want to perform mutation. For generate policies, the triggering resource should be provided on creation/update of which our generation rule is applied.

#### Example: Add Network Policy

A sample test file for [this policy](https://kyverno.io/policies/best-practices/add_network_policy/) will look like this -

```yaml
name: deny-all-traffic
policies:
  - policy.yaml
resources:
  - resource.yaml
results:
  - policy: add-networkpolicy
    rule: default-deny
    resource: hello-world-namespace
    generatedResource: generatedResource.yaml
    kind: Namespace
    result: pass
```
resource.yaml is the manifest for creating a new Namespace resource. The result will pass if the generatedResource.yaml looks like this -

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny
  namespace: hello-world-namespace
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
```

### Method to test clone object

Generate rules with clone object requires a pre-existing resource in the cluster. The manifest of the rule will not have the definition of this resource. Instead, the rule will only specify the name of the resource and the namespace where it resides. To have the resource exist on the cluster before the rule is applied, user should provide the definition of the resource. The rest of the approach remains same as that of data object.

#### Example: Sync Secrets

Consider [this policy](https://kyverno.io/policies/other/sync_secrets/). The test file should look like this -

```yaml
name: sync-secrets
policies:
  - policy.yaml
resources:
  - resource.yaml
results:
  - policy: sync-secrets
    rule: sync-image-pull-secret
    resource: hello-world-namespace
    generatedResource: generatedResource.yaml
    cloneSourceResource: cloneSourceResource.yaml # Pre-existing resource definition
    kind: Namespace
    result: pass
```

User should provide the pre-existing resource in the `cloneSourceResource` field of the results array. The cloneSourceResource.yaml for the above test file will look like this -

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: regcred
  namespace: default
type: Opaque
data:
  password: MWYyZDFlMmU2N2Rm
```

The test will pass if the generatedResource.yaml looks like this -

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: regcred
  namespace: hello-world-namespace
type: Opaque
data:
  password: MWYyZDFlMmU2N2Rm
```

### Namespaced Policies

In order to test namespaced policies, user have to explicitly mention the namespace of the policy in the test file. Additionally, the triggering resource should belong to the namespace for which the policy is configured.

#### Example: Policy Disruption Budget

Consider this namespaced policy -
```yaml
apiVersion: kyverno.io/v1
kind: Policy
metadata:
  name: create-default-pdb
  namespace: hello-world
spec:
  rules:
  - name: create-default-pdb
    match:
      resources:
        kinds:
        - Deployment
    generate:
      apiVersion: policy/v1
      kind: PodDisruptionBudget
      name: "{{request.object.metadata.name}}-default-pdb"
      namespace: "{{request.object.metadata.namespace}}"
      data:
        spec:
          minAvailable: 1
          selector:
            matchLabels:
              "{{request.object.metadata.labels}}"
```

The test file for this policy should explicitly mention the policy namespace.

```yaml
name: pdb-test
policies:
  - policy.yaml
resources:
  - resource.yaml
results:
  - policy: create-default-pdb
    rule: create-default-pdb
    resource: nginx-deployment
    generatedResource: generatedResource.yaml
    kind: Deployment
    result: pass
    namespace: hello-world
```

## Implementation details

Most of the implementation is similar to testing mutate policies. The key difference is that generate rules require a Generate-Request Controller.

### Mock Generate-Request Controller

To process generate rules, we need a Generate-Request Controller. We can make a simple instance of the Controller with only the Kubernetes' dynamic client. The dynamic client must be a fake one so as to not process the rule on an actual cluster. Additionally, the triggering resource and pre-existing resource for the clone object should exist in the cluster. The dynamic client will be initialized with these resources.

### Building the result

We can use the same function used for testing mutate policies to compare user defined resource and Kyverno generated resource and give it some generic name.
