# Policy Exceptions

- **Authors**: Eileen Yu (@Eileen-Yu), Jim Bugwadia (jim@nirmata.com)
- **Created**: Sep 16th, 2022
- **Updated**: Sep 20th, 2022
- **Abstract**: Allow managing policy exceptions (exclude) independently of policies

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

Kyverno policy rules contain an `exclude` block which allows excluding admission requests by resource (name, API group, versions, and kinds), resource labels, namespaces, namespace labels, users, use groups, service accounts, user role and groups. This construct allows flexibility in selecting when a policy should be applied to a admission review request.

This proposal introduces the concept of a `PolicyException` which is similar to the `exclude` declaration but is decoupled from the policy lifecycle. 

## The Problem

Policies are often managed via GitOps, and a global policy set may apply across several clusters. In some cases a policy override may be required in specific clusters. Hence, managing exceptions in the policy declaration itself can become painful. 

In addition, when a policy exception declaration is updated, there is no clear record of why that exception was added.

These pain points are well ariculated in the following GitHub issue and discussion:

- [Overwriting Existing policies](https://github.com/kyverno/kyverno/discussions/4310)
- [Feature: New CRDs for PolicyViolationExcepotions](https://github.com/kyverno/kyverno/issues/2627)


## Proposed Solution

The proposed solution is to introduce two new Custom Resource Definitions (CRDs) that enable managing policy exceptions:

* **PolicyExceptionRequest**: a namespaced request to exclude a resource from a policy rule
* **PolicyExceptionApprovals**: a cluster-wide resource that permits one or more exceptions

#### High-Level Flow

1. A user that is impacted by a policy violaation can create a `PolicyExceptionRequest` for their workload:

`PolicyExceptionRequest` sample:

```yaml
...
spec:
  policyName:..
  ruleName:..
  reason:..
  exclude:..
  duration(optional):..
status:
  state: `Pending` / `Approved` / `Rejected`
```

- **policyName**: target policy name
- **ruleName**: target rule name
- **reason**: brief explanation for the exception
- **exclude**: all information available in the `exclude` declaration
- **duration(optional)**: how long the exception is required for

2. The admin can review the `PolicyExceptionRequest` and can either `approve` or `reject` it. This is done by updating the `PolicyExceptionApprovals` which serves as a log of approval decisions. 

`PolicyExceptionApprovals` sample:

```yaml
spec:
  exceptions:
  - name: permit-root-user
    namespace: nginx
  - name: permit-host-port
    namespace: nginx 
```


3. When Kyverno applies a policy rule, it will now use both the `exclude` declaration in the policy and a set of chached policy exceptions that match the policy rule.

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

**b. What scenario the requester want to exclude from the policy?**

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
