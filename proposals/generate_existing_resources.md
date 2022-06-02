# Kyverno Design Proposal - Generate Existing Resources

Created: Feb 24th, 2022
Author: [Prateek Pandey](https://github.com/prateekpandey14)

  * [Overview](#overview)
  * [Proposal](#proposal)
  * [Implementation](#implementation)
  * [Example](#example)

## Overview

Generate-Policy Controller runs as an custom controller in Kubernetes watches
UpdateRequest resource and applies generate rules to the resource.
- UpdateRequest
- Namespace(re-evaluates policy on namespace resource updates)

Use of a generate rule is common when creating new resources from the point after
which the policy was created. For example, a Kyverno generate policy is created 
so that all future namespaces can receive a standard set of Kubernetes resources. 

However, there are use cases to generate resources into existing resources. This
can be extremely useful when deploying Kyverno to an existing cluster in use where
you wish policy to apply retroactively.

- Trigger generate rules on existing resources: https://github.com/kyverno/kyverno/issues/2570

Currently this use case can be achieved using selectors, for example


Below is the example which triggers the resource generation on update of label/annotation, we should add a selector for filtering the resources based on label/annotation.

```yaml 
    match:
      resources:
        kinds:
        - v1/ServiceAccount
        selector:
          matchLabels:
            security: standard
```
 
But the problem this creates is that new Service Accounts also need to include this arbitrary label otherwise they are excluded. 
To avoid having to add the label to all new service accounts, the match.any can be used as well, as described below.

```yaml
    match:
      any:
      # for existing service accounts
      - resources:
        kinds:
        - v1/ServiceAccount
        selector:
          matchLabels:
            security: standard
      # for new service accounts
      - resources:
        kinds:
        - v1/ServiceAccount
```

Creating a ClusterPolicy with a rule containing a match statement which matches on kind as well as the label you have set on the resource, but this feels very counter-intuitive and cumbersome, hence the new proposal to make this easy to use.


## Proposal

- Whenever trigger resource is created or updated, kyverno will  see the policy update, if it finds that policy has the `genExisting:true` field set , the new generate background scan handler will scan through all the namespaces with matching Kind and then trigger the current generate controller workflow to handle the UpdateRequest operations.
This approach seems to be more intuitive, helps and improves user experience as compared to earlier one.

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: sync-docker-creds
spec:
  background: true
  generateExistingOnPolicyUpdate: true   // trigger generate rule in existing resources
  rules:
  - name: sync-image-pull-secret
    match:
      resources:
        kinds:
        - Namespace
    generate:
      kind: Secret
      name: docker
      namespace: "{{request.object.metadata.name}}"
      synchronize: true
      clone:
        namespace: default
        name: tls-pair
```
- Kyverno will write a label `kyverno.io/background-gen-rule: <rule_name>` to each
newly-generated resource if `genExisting: true`.

- CLI support with the generate dry-run command, similar to background generation,
to allow users to understand the impact such a policy may have on existing resources.

## Implementation

- Add a new attribute to the policy generation rule definition, to handle the generate of existing trigger resources.

```go
// Spec contains a list of Rule instances and other policy controls.
type Spec struct {
	// Rules is a list of Rule instances. A Policy contains multiple rules and
	// each rule can validate, mutate, or generate resources.
	Rules []Rule `json:"rules,omitempty" yaml:"rules,omitempty"`

    ...
   // MutateExistingOnPolicyUpdate controls if a mutateExisting policy is applied on policy events.
	// Default value is "false".
	// +optional
	MutateExistingOnPolicyUpdate bool `json:"mutateExistingOnPolicyUpdate,omitempty" yaml:"mutateExistingOnPolicyUpdate,omitempty"`

	// GenerateExistingOnPolicyUpdate controls wether to trigger generate rule in existing resources
	// If is set to "true" generate rule will be triggered and applied to existing matched resources.
	// Defaults to "false" if not specified.
	// +optional
	GenerateExistingOnPolicyUpdate bool `json:"generateExistingOnPolicyUpdate,omitempty" yaml:"generateExistingOnPolicyUpdate,omitempty"`
}
```

- Expand the background scan workflow, which currently only applies on validation
and image verify rules with `background: true` to create or update policy reports,
to also apply generate policies on matching resources with Kind as well.


- Define a separate handler as a background scanner to handle the generate specific policies.

- Kyverno will write a label `kyverno.io/background-gen-rule: <rule_name>` to each 
newly-generated resource based upon this behavior, so in case the action be undesirable by the user,
it is easier to find and then (optionally) delete those resources to clean them up.

  For example, consider this scenario:
  <details>
  <summary>Click to expand!</summary>

  - A user writes a policy to generate a new NetworkPolicy resource based on existing Deployments.They forget to use a label 
  selector to filter down the list of Deployments.

  - They do not use the kyverno apply command to perform a dry run on their running cluster to see the effect creating this policy will have.

  - The new policy/rule is created in a running cluster. New policy/rule creates 100 NetworkPolicy objects (one for every Deployment) instead of just the intended 10 (the ones with the label selector they forgot to apply).

  - The user now has to clean up all (or at least 90) unwanted NetworkPolicy resources.
  If all Kyverno-created NetworkPolicy resources have a standard label written to them, for example, `kyverno.io/background-gen-rule: <rule_name>`, the user can perform these operations to clean up the undesirable `NetworkPolicies` in one fell swoop.

      - `kubectl get netpol -l kyverno.io/background-gen-rule: <rule_name>`

      - `kubectl delete netpol -l kyverno.io/background-gen-rule: <rule_name>` 
      
  </details>

## Example

1. This example generate data `NetworkPolicy` to both incoming and existing Namespace with `genExisting=true`

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: "..."   
spec:
  background: true
  validationFailureAction: audit
  generateExistingOnPolicyUpdate: true   // trigger generate rule in existing resources
  rules:
  - name: default-deny
    match:
      resources:
        kinds:
        - Namespace
    generate:
      kind: NetworkPolicy
      name: default-deny
      namespace: "{{request.object.metadata.name}}"
      synchronize: true
      data:
        spec:
          # select all pods in the namespace
          podSelector: {}
          # deny all traffic
          policyTypes:
          - Ingress
          - Egress
```

2. This example generate clone `Secret` to both incoming and existing Namespace with `genExisting=true`

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: "..."
spec:
  background: true
  validationFailureAction: audit
  generateExistingOnPolicyUpdate: true   // trigger generate rule in existing resources
  rules:
  - name: sync-image-pull-secret
    match:
      resources:
        kinds:
        - Namespace
    generate:
      apiVersion: v1
      kind: Secret
      name: secret-basic-auth
      namespace: "{{request.object.metadata.name}}"
      synchronize: true
      clone:
        namespace: default
        name: secret-basic-auth
```

