# Policy Exceptions

- **Authors**: Eileen Yu (@Eileen-Yu), Jim Bugwadia (jim@nirmata.com)
- **Created**: Sep 16th, 2022
- **Updated**: Sep 26th, 2022
- **Abstract**: Allow managing policy exceptions (exclude) independently of policies

## Contents

- [Introduction](#introduction)
- [The Problem](#the-problem)
- [Proposed Solution](#proposed-solution)
  - [High-Level Flow](#high-level-flow)
  - [Ideal Usage](#ideal-usage)
  - [Possible User Stories](#possible-user-stories)
- [Implementation](#implementation)
- [Alternative Solution](#alternative-solution)
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

- **PolicyExceptionRequest**: a namespaced request to exclude a resource from a policy rule
- **PolicyExceptionApprovals**: a cluster-wide resource that permits one or more exceptions

### High-Level Flow

There are 2 personnas that would be involved:

- **Policy User**: able to create, delete , and modify `PolicyExceptionRequest` resources
- **Policy Admin**: able to review `PolicyExceptionRequest`; and create, delete, modify `PolicyExceptionApprovals` resources

1. A user that is impacted by a policy violation can create a `PolicyExceptionRequest` for their workload:
   `PolicyExceptionRequest` sample:

```yaml
spec:
  exceptions:
  - policyName: "pod-security"
    ruleNames:
    - "restricted"
    reason: "..."
    exclude: {}      # same as a policy exclude declaration
    duration: "30d"  # optional time string 
status:
  state: "Pending" # Pending | Approved | Denied
```

`spec` is where the user fills in the request.

- exceptions: a list of exception objects:
  - policyName: target policy name (string)
  - ruleNames: an array of target rule names (string)
  - reason: brief explanation for the exception
  - exclude: specify the resource that need to be excluded from the target policy, it can be [MatchResources](https://github.com/kyverno/kyverno/blob/v1.8.0-rc2/api/kyverno/v1/match_resources_types.go#L10)
  - duration(optional): how long the exception would be valid

`status` is where the `PolicyExceptionController` would automatically update based on Policy Admin activity and other updates.

**PolicyExceptionRequest lifecycle**

* If the `state` is `Pending` (the default), user can delete or modify the PolicyExceptionRequest. 
* If the state is `Approved`, user will be blocked from modifying the PolicyExceptionRequest. 
* The user can always delete the PolicyExceptionRequest when it is no longer required. 
* If the state is `Denied`, user can modify the request. The controller would change the state back to `Pending`. We can use RBAC to restrict such accessibility.

2. The admin can review the `PolicyExceptionRequest` and can either `approve` or `deny` it. This is done by updating the `PolicyExceptionApprovals` which serves as a log of approval (or denial) decisions.

`PolicyExceptionApproval` sample:

```yaml
spec:
  approvalPolicy: Manual              # Manual (default) | Automatic
  exceptions:
    - name: permit-root-user
      namespace: nginx
      status: "Approved"
      reason: "..."
    - name: permit-host-port
      namespace: nginx
      status: "Denied"
      reason: ...
```

- approvalPolicy: policies that are approved manually / automatically
- exceptions: an array of exception elements
  - name: name of corresponding `PolicyExceptionRequest`
  - namespace: namespace of corresponding `PolicyExceptionRequest`
  - status: whether admin decide to approve / deny the `PolicyExceptionRequest`
  - reason: why admin approve / deny the `PolicyExceptionRequest`

The `status` and `reason` would also be updated to the `PolicyExceptionRequest` to let the user know by the `PolicyExceptionController`.

### Ideal Usage

1. User creates a CR `PolicyExceptionRequest` to request for exception in a certain namespace.
2. Admin reviews the request, and adds new element in `PolicyExceptionApprovals` to approve / deny the request.
3. A new `PolicyExceptionController` would update the `PolicyExceptionRequest` based on admin's feedback. If not approved, user can modify and resubmit `PolicyExceptionRequest`.
4. The `PolicyExceptionApprovals` would be cached by Kyverno for policy match and exclude.
5. User creates the corresponding resource. Kyverno matches the resource with corresponding policies and exceptions. If all broken rules can be matched with exceptions, let the request pass.

### Possible User Stories

**As a user, I want an exception from the latest tag rule for test.**

**a. What policy blocks what resources/user?**

The policy blocks users from deploying an image tag called `latest`.

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: disallow-latest-tag
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

**b. What scenario the user want to exclude from the policy?**

The user may not have access to create a new tag. The default one would be 'latest'. Then this would violate the policy.

**c. Why it is not necessary to modify the policy?**

The original rule is still valid for the other workloads. We just need an exception for the dev test.

**d. How does the user write the new CR?**

The user needs to specify the target policy / rule, and to what kind of resource the exception would take effect.

```yaml
apiVersion: kyverno.io/v1
kind: PolicyExceptionRequest
metadata:
  name: allow-tag-for-test
spec:
  exceptions:
    - policyName: disallow-latest-tag
      ruleNames:
        - validate-image-tag
  reason: allow-latest-tag-for-test
  exclude:
    all:
      - name: test
        kind:
          - Deployment
        namespace:
          - dev
```

The `state` would be automatically changed to `Pending`.

**e. How does the administrator review the CR?**

Once the CR is created, the admin can review the CR and update `PolicyExceptionApprovals`.

(1) The admin reject the request by adding one element, setting the `status` as `Denied`.

```yaml
spec:
  approvalPolicy: manual
  exceptions:
    - name: allow-tag-for-test
      namespace: dev
      status: Denied
      reason: ...
```

(2) The admin approve the request by adding one element, setting the `status` as `Approved`.

```yaml
spec:
  approvalPolicy: manual
  exceptions:
    - name: allow-tag-for-test
      namespace: dev
      status: Approved
      reason: ...
```

**f. What happened if the CR get rejected?**

The `status` in the `PolicyExceptionRequest` would be updated to `Denied`, and the `reason` in the `PolicyExceptionRequest` would also be updated to tell the user the rejection reason.

The user can modify `PolicyExceptionRequest` based on feedback and resubmit. The `status` in `PolicyExceptionRequest` would be then changed to `Pending` again and wait to be reviewed.

**g. What happened next if the CR get approved?**

Now the `PolicyExceptionRequest` is valid. Kyverno would cache the updated `PolicyExceptionApprovals`, and use in the logic for policy match and exclude.

**h. What does the user need to do after?**

Once the `PolicyExceptionRequest` gets approved, it automatically take effect. The user can now create the resource.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test
  labels:
    app.kubernetes.io/name: test
spec:
  containers:
    - name: test
      image: test:latest
```

**i. What if the user try to violate another policy rule that is not included by the exception request?**

If the breaking rule cannot be matched with the exceptions, the requested resources should be blocked still.

## Implementation Notes

We will introduce a new controller: `PolicyExceptionController` and two new CRDs: `PolicyExceptionRequest` and `PolicyExceptionApprovals`.

We can use `Kubebuilder` to scaffold the new operator. The CRDs `PolicyExceptionRequest` and `PolicyExceptionApprovals` are also defined in this scaffold project.

The `PolicyExceptionController` would act as a bridge between `PolicyExceptionRequest` and `PolicyExceptionApprovals`. 

More specifically, the controller will:
- Watch the `PolicyExceptionApprovals`, updates `status` & `reason` for `PolicyExceptionRequest`
- Block changes to `PolicyExceptionRequest` if the `state` is `Approved`
- Change `state` back to `PendingApproval` if user modified `PolicyExceptionRequest`
- Clean up stale entries / invalid references in `PolicyExceptionApprovals`
- Handle auto Approvals in the future
- Handle signed Approvals in the future

Kyverno would cache `PolicyExceotionApprovals`. We may introduce a new `ExceptionInformer` to get the latest `PolicyExceptionApprovals` in the cluster. We can optimize to only cache approved exceptions.

When the user applies the target resource, Kyverno would first go through the policy check. A new method will be introduced to fetch excpetions and add them to the policy rule `exclude` block.

![flow](https://user-images.githubusercontent.com/48944635/192390568-d03e2c98-5902-41a2-9082-07994582b33c.png)

## Alternative Solution

An alternative is to let admin directly give feedback in the `PolicyExceptionRequest`.

`PolicyExceptionRequest` Sample:

```yaml

---
spec:
  exceptions:
    - policyName: "..policyA"
      ruleNames:
        - ".."
        - ".."
    - policyName: "..policyB"
      ruleNames:
        - ".."
        - ".."
  reason: ..
  exclude: ..
  duration(optional): ..
status:
  state: PendingApproval / Approved / Denied
  feedback:
    result: deny/accept
    description: ...
```

After the `PolicyViolationException` CR is applied to the cluster, the admin would be able to review the content and modify `spec` if necessary.

`status` is the field which can only be filled by the admin.

- feedback:
  - result: whether to accept or deny the request
  - description: reason to change / deny the request
- state: automatically updated by `PolicyExceptionController` (not necessary to be manually updated)

The `PolicyExceptionController` would watch the `state` in `PolicyExceptionRequest`. If it's `Approved`， `PolicyExceptionController` would automatically create one `PolicyExceptionApproval` for each `PolicyExceptionRequest`.

`PolicyExceptionApproval` Sample:

```yaml

---
spec:
  exceptions:
    - policyName: "..policyA"
      ruleNames:
        - ".."
        - ".."
    - policyName: "..policyB"
      ruleNames:
        - ".."
        - ".."
  exclude: ..
  duration: ..
```

### Alternative Usage

1. User creates a CR `PolicyViolationException` to request for exception.
2. Admin reviews the request, approve or reject.
3. The `PolicyExceptionController` would automatically create a new CR `PolicyExceptionApproval` for each exception.
4. Kyverno gets these `PolicyExceptionApproval`s cached.
5. User creates the corresponding resource. Kyverno matches the resource with corresponding policies and exceptions. If all broken rules can be matched with exceptions, let the request pass.

This alternative may decrease complexity of state control. The `PolicyExceptionController` would not have to watch two CRs.
This may also help avoid some extreme conficts, such as user may modify the `PolicyExceptionRequest` when the admin is updating `PolicyExceptionApprovals`.

![flow](https://user-images.githubusercontent.com/48944635/192406280-4225202e-64e3-4a73-a9ab-31727646dc17.png)


### Questions and Notes

1. The PolicyExceptionRequest is namespaced. Is there a use-case for a cluster-wide exception?

2. Each new CR has a independent lifecycle and is managed by different roles. This seems OK, but needs more analysis.

3. The PolicyExceptionController will manage the `status` of the PolicyExceptionRequest and also watch the PolicyExceptionApprovals. Is this an appropriate controller design?


## Next Steps

This proposal just provides a solution for basic exception. There are many other things need to be taken into consideration.
1. More user stories / functions
2. Exceptions lifecycle management
3. Auto-approvals
4. Exception cleanup
5. Documentation


