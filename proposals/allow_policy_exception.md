# Allowing Policy Exception

- **Authors**: Jim Bugwadia (jim@nirmata.com), Eileen Yu (@Eileen-Yu)
- **Date**: Sep 16th, 2022
- **Update**:
- **Abstract**: Overwrite policy for certain resources

## Contents

- [Contents](#contents)
- [Introduction](#introduction)
- [The Problem](#the-problem)
- [Proposed Solution](#proposed-solution)
  - [Possible Solution](#possible-solution)
    - [Request](#request)
    - [Response](#response)
  - [Possible User Stories](#possible-user-stories)
- [Next Steps](#next-steps)

## Introduction

This document introduces a new CRD `PolicyViolationExceptions` . The proposed CRD aims to provide certain resource with privileges. The resource approved by such CRD would temporarily escape from specific policy rules.

## The Problem

When a policy is applied, all the matching resources / users have to follow the rules specified in it. But in certain cases, user may find necessary to get permission which will break current rules. Nonetheless, the original rule is still valid and should not be changed, especially for those cases that would break the rule for only once or in a period. This requires to introduce the exception mechanism.

Some related issues on this:

- [Overwriting Existing policies](https://github.com/kyverno/kyverno/discussions/4310)
- [Feature: New CRDs for PolicyViolationExcepotions](https://github.com/kyverno/kyverno/issues/2627)

One possible solution is to overwrite the existing policies, where the user adds certain cases in the `exclude` spec, or retune some other fields. However, this would cause some ambiguous permission boundary as well as some security risks. Also, additional spec would be needed to specify if a certain policy can be overwrote or not.

It is recommended to introduce a new CRD for those privileged resources while avoid affecting the existing policy.

## Proposed Solution

### Possible Solution

For a clearer authority management, a `Request-Response` mechanism can be considered.

#### Request

The user would create the `PolicyViolationExceptions` CR.

`PolicyViolationExceptions` Example:

```yaml
...
spec:
  policyName:..
  ruleName:..
  reason:..
  exclude:..
  duration(optional):..
status:
  state: `PendingApproval` / `Approved` / `Rejected`
```

- policyName: target policy name
- ruleName: target rule name
- reason: brief explanation for the exception
- exclude: specify the resource that need to be excluded from the target policy, may include resource information, e.g. kind, name, namespace, labels
- duration(optional): how long the exception would be valid

#### Response

Once the CR is created, the admin would be able to review the content and modify it if necessary.

1. If the exception is approved, a new CR `Approval` would be generated.

`Approval` Example:

```yaml
---
spec:
  exceptions: name:..
    nameSpace:..
    duration:..
```

- name: name for the approved exception
- nameSpace: valid namespace that would be affected by the exception
- duration: how long the exception would be valid

Then the relative resource would get mutated automatically based on the dual effect of the original policy and the exception.

2. If the exception is rejected, then nothing would go on.

The CRs would get recycled once the resources are reconciled OR it is outdated.

### Possible User Stories

**As a user, I want to be excepted from label tag rule for test.**

**a. What policy blocks what resources/user?**

The policy disables the user to use an image tag called `latest`.

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: disallow-latest-tag
  annotations:
    policies.kyverno.io/title: Disallow Latest Tag
    policies.kyverno.io/category: Best Practices
    policies.kyverno.io/severity: medium
    policies.kyverno.io/subject: Pod
    policies.kyverno.io/description: >-
      The ':latest' tag is mutable and can lead to unexpected errors if the
      image changes. A best practice is to use an immutable tag that maps to
      a specific version of an application Pod. This policy validates that the image
      specifies a tag and that it is not called `latest`.
spec:
  validationFailureAction: enforce
  background: true
  rules:
    - name: validate-image-tag
      match:
        resources:
          kinds:
            - Deployment
      validate:
        message: "Using a mutable image tag e.g. 'latest' is not allowed."
        pattern:
          spec:
            template:
              spec:
                containers:
                  - image: "!*:latest"
```

**b. What scenario the requester want to escape from the policy?**

The requester may not want to create a specific image tag which just used for test. The default one would be 'latest'. Then this would break the policy.

**c. Why it is not necessary to modify the policy?**

The original rule is still valid for the production. We just need an exception for the dev test.

**d. How does the requester write the new CR?**

The requester needs to specify the target policy / rule, and to what kind of resource the exception would take effect.

```yaml
apiVersion: kyverno.io/v1
kind: PolicyExceptionRequest
metadata:
  name: allow-tag-for-test
spec:
  policyName: disallow-latest-tag
  ruleName: validate-image-tag
  reason: allow-latest-tag-for-test
  exclude:
    all:
      - name: test
        kind:
          - Deployment
```

**e. How does the administrator review the CR?**

Once the CR is created, the admin can review the CR and modify the spec if necessary.

(1) The admin reject the request by modifying the `state`.

```yaml
spec:
---
status:
  state: Rejected
```

(2) The admin approve the request by modifying the `state`.

```yaml
spec:
---
status:
  state: Approved
```

**f. What happened if the CR get rejected?**

The `state` in the CR would be `PendingApproval` until the request get outdated and the `state` would be `Rejected`.

**g. What happened next if the CR get approved?**

A new CR `Approval` would be generated. Here the admin would specify the valid scope and period.

```yaml
apiVersion: kyverno.io/v1
kind: Approval
metadata:
  name: allow-tag-for-test
spec:
  exceptions:
    name: allow-tag-for-test
    nameSpcace: dev
    duration: 3
```

**h. What does the requester need to do after?**

Once the `Approval` CR gets generated, the exception would automatically take effect on the original policy. The requester won't need to do anything other than continue to create the resource.

## Next Steps

This proposal just provides a solution for basic exception. There are many other things need to be taken into consideration.

1. More user stories / functions
2. Recycling Mechanism
3. Documentation

## Open questions

1. should `status` be readOnly? And just let admin fill in `feedback`

example:

```yaml

---
spec:
  policyName: disallow-latest-tag
  ruleName: validate-image-tag
  reason: allow-latest-tag-for-test
  exclude:
    all:
      - name: test
        kind:
          - Deployment
  feedback:
    result: accept / deny
    description: ..
status:
  state: pending/...
```

2. api edit permission
3. Avoid contradiction (no 2 admins editing at the same time)

   - explore the mechanism

4. Requester can't change CR after apply
