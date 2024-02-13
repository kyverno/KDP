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

Configure Kyverno resource webhook configurations based on policy settings. With Kyverno 1.11, the webhooks are registered for selected resources (by policies) with `apiGroups`, `apiVersions` and `resources`. This KDP proposes to register these resources' webhooks with the fine-grained filters.

# Definitions
[definitions]: #definitions

Kyverno has two webhook configurations that are auto-created and managed for resources selected by policies, a MutatingWebhookConfiguration `kyverno-resource-mutating-webhook-cfg` and a ValidatingWebhookConfiguration `kyverno-resource-validating-webhook-cfg` respectively. This proposal uses validatingwebhookconfiguration to demonstrate the proposed solution, the same applies to the resource mutatingwebhookconfiguration.

# Motivation
[motivation]: #motivation

From [#9111](https://github.com/kyverno/kyverno/issues/9111) and [#8063](https://github.com/kyverno/kyverno/issues/8063), the request is to register webhooks for the policy selected resources which:

1. allows users to customize webhooks flexibly 
2. eliminates unnecessary admission requests being forwarded to Kyverno

Benefits of policy based webhooks:
1. offers the flexibility for webhook configurations
2. prevents unexpected webhook failures

# Proposal

There are three dimensions for this feature:

- configure `matchConditions` in validatingwebhookconfiguration via `webhookMatchConditions` based on the policy definition
- register webhook's scope based on the policy type, specifically to configure the `scope` attribute of the validatingwebhookconfiguration for namespaced policies
- configure rule operations in validatingwebhookconfiguration based on `operations` defined in a Kyverno rule

**a. configure the webhooks' `matchConditions` via CEL expressions (requires Kubernetes 1.27+)**

A new attribute will be added to Kyverno policy `spec.webhookMatchConditions`:

- similar to `spec.failurePolicy`, `spec.webhookMatchConditions` is a per policy based configuration. It will be converted to `webhooks.matchConditions` directly in validatingwebhookconfiguration, the name for each webhook will follow the naming convention `validate.kyverno.svc.<failure policy>.<policy type>.<policy name>`. Each policy with `webhookMatchConditions` will add one entry in `validatingwebhookconfiguration.webhooks`.
- matching admission requests will be forwarded to and served at path `/validate/matchconditions/fail` by default. In case `spec.failurePolicy=Ignore`, requests are served at `/validate/matchconditions/ignore`.

Given this clusterpolicy with two rules, it will be transformed to validatingwebhookconfiguration listed below.

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: disallow-host-pid-ipc
spec:
  failurePolicy: Fail
  webhookMatchConditions:
  - name: 'exclude-kubelet-requests'
    expression: '!("system:nodes" in request.userInfo.groups)' 
  rules:
  - match:
      any:
      - resources:
          kinds:
          - Pod
    name: validate-hostPID-hostIPC
  - match:
      any:
      - resources:
          kinds:
          - Deployment
    name: auto-gen-validate-hostPID-hostIPC
```

- the service path is registered at `/validate/matchconditions/fail`
- the new webhook entry will be registered with name `validate.kyverno.svc.fail.clusterpolicy.disallow-host-pid-ipc`
- `clusterpolicy.spec.webhookMatchConditions` is transformed to `validatingwebhookconfiguration.webhooks.matchConditions`
- two rules matching `Pod` and `Deployment` are registered under `validatingwebhookconfiguration.webhooks.rules` respectively

```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingWebhookConfiguration
metadata:
  name: "kyverno-resource-validating-webhook-cfg"
webhooks:
- name: "validate.kyverno.svc.fail.clusterpolicy.disallow-host-pid-ipc"
  clientConfig:
    service:
      name: kyverno-svc
      namespace: kyverno
      path: /validate/matchconditions/fail
      port: 443
  matchConditions:
  - name: 'exclude-kubelet-requests'
    expression: '!("system:nodes" in request.userInfo.groups)' 
  rules:
  - apiGroups:   [""]
    apiVersions: ["v1"]
    resources:   ["pods"]
  - apiGroups: ["apps"]
    apiVersions: ["v1"]
    resources: ["deployments"]
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
- name: validate.kyverno.svc-fail
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

The existing attribute `spec.rules.match.(any/all).resources.operations` will be transformed to the operations per webhook's rule.  

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
- name: validate.kyverno.svc-fail
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

- Policies with `spec.webhookMatchConditions` will be examined when creating policy reports. For match conditions that contain userInfo or other RBAC related payloads, Kyverno will skip creating reports.

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