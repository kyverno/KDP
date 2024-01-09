# Meta
[meta]: #meta
- Name: Policy-based Webhook Configurations
- Start Date: 2023-12-12
- Update data (optional):
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

Configure Kyverno resource webhook configurations based on policy settings. With Kyverno 1.11, the webhooks are registered for selected resources (by policies) with `apiGroups`, `apiVersions` and `resources`. This KDP proposes to register these resources' webhooks with the fine-grained filters via `matchConditions`.

# Definitions
[definitions]: #definitions

Kyverno has two webhook configurations that are auto-created and managed for resources selected by policies, a MutatingWebhookConfiguration and a ValidatingWebhookConfiguration respectively. For brevity, this proposal simply uses "webhooks" to refer to the two mentioned webhook configurations.

# Motivation
[motivation]: #motivation

From [#9111](https://github.com/kyverno/kyverno/issues/9111) and [#8063](https://github.com/kyverno/kyverno/issues/8063), the request is to register webhooks for the policy selected resources which:

1. allows users to customize webhooks flexibly 
2. eliminates unnecessary admission requests being forwarded to Kyverno

Benefits of policy based webhooks:
1. offers the flexibility for webhook configurations
2. prevents unexpected webhook failures
3. reduces the admission review processing duration

# Proposal

There are two dimensions for this feature:

1. offer users the option to configure webooks' attributes via `matchConditions` based on the policy settings, can be later extended to `namespaceSelector`
2. register webhooks based on the policy type, specifically to configure the `scope` attribute of webhooks for namespaced policies

The following snippets list out how policy configurations are transformed to the webhook configurations.

**a. configure the webhooks' `matchConditions` via CEL expressions (requires Kubernetes 1.27+)**
The new attribute will be added to Kyverno policy `spec.rules.match.(any/all).resources.matchConditions`.
```yaml
spec:
  rules:
  - match:
      all:
      - resources:
          matchConditions:
            matchExpressions:
            # Match requests made by non-node users.
            - name: 'exclude-kubelet-requests'
              expression: '!("system:nodes" in request.userInfo.groups)' 
```
The above rule will be transformed to the following webhook configuration.
```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingWebhookConfiguration
webhooks:
  - name: my-webhook.example.com
    # note: Kubernetes allows up to 64 matchConditions per webhook
    matchConditions:
    - name: 'exclude-kubelet-requests'
      expression: '!("system:nodes" in request.userInfo.groups)' 
```
**b. configure webhook's scope based on the policy type**

By default the scope of the registered webhooks rules are set to "*" which means no scope restrictions. For a namespaced policy, the scope will be registered as "Namespaced".

```yaml
apiVersion: kyverno.io/v1
kind: Policy
metadata:
  name: inject-annotations
  namespace: default
  annotations:
    policies.kyverno.io/title: Inject Annotations
spec:
  background: false
  rules:
    - name: add-annotations
      match:
        any:
          - resources:
              kinds:
                - '*'
```
The above rule will be transformed to the following webhook configuration.
```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingWebhookConfiguration
webhooks:
- name: my-webhook.example.com
  rules:
  - apiGroups:
    - '*'
    apiVersions:
    - '*'
    resources:
    - '*'
    scope: Namespaced
```

**c. filter by the request operations**
The existing attribute `spec.rules.match.(any/all).resources.operations` can be transformed to the operations per webhook's rule.  
```yaml
spec:
  rules:
    - match:
        all:
        - resources:
            kinds:
              - apps/v1/Deployment
            operations:
              - CREATE
```
The above rule will be transformed to the following webhook configuration.
```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingWebhookConfiguration
webhooks:
- name: my-webhook.example.com
  rules:
  - operations: ["CREATE"]
    apiGroups: ["apps"]
    apiVersions: ["v1"]
    resources: ["deployments"]
```

# Implementation

TBD.

## Link to the Implementation PR

https://github.com/kyverno/kyverno/pull/8065 registers the webhook's scope based on the policy type.

https://github.com/kyverno/kyverno/pull/8437 registers the webhook operations based on the policy rule definitions.

## Other requirements

- All the mentioned attributes under the policy `match` block will be examined when creating policy reports. While this is not necessary needed when Kyverno processes the admission requests as they are already filtered by the webhook. The optimization can be made to accelerate the admission review process.

# Migration (OPTIONAL)

This section should document breaks to public API and breaks in compatibility due to this KDP's proposed changes. In addition, it should document the proposed steps that one would need to take to work through these changes.

# Drawbacks

Why should we **not** do this?

# Alternatives

- What other designs have been considered?
- Why is this proposal the best?
- What is the impact of not doing this?

# Prior Art

Discuss prior art, both the good and bad.

# Unresolved Questions

- The `exclude` block of a rule will not be considered when transforming to the webhooks
- This feature should be offered as an opt-in operation, all the mentioned attributes under the policy `match` block are optionally registered in webhooks.

# CRD Changes (OPTIONAL)

Does this KDP entail any proposed changes to the core CRD or any extensions? If so, please document changes here.