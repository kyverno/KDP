# Kyverno Design Proposal - Adding Request Types to match and exclude

Author: [Prateek Nandle](https://github.com/Prateeknandle)

PR: [#3175](https://github.com/kyverno/kyverno/pull/3175)

## Contents

- [Kyverno Design Proposal - Adding Request Types to match and exclude](#kyverno-design-proposal---adding-request-types-to-match-and-exclude)
  * [Introduction](#introduction)
  * [Proposal](#proposal)
  * [Example](#example)
  * [Implementation](#implementation)

## Introduction

Until now we use request.operation under condition.key, to filter a request, now we'll use a tag requestTypes in match or exclude blocks to match specific requests comming from admission review requests, & then according to the defined conditions the policy would apply on to the resource.

## Proposal

After an request is made, the Kubernetes api will send the information about the requests to the registered webhook through admission review request. Then requestTypes value under match/exclude block will be matched with the incomming requests, if the requests matched then the policy would be applied to the resource & the behavior of the policy will depend on the conditions mentioned in the policy.

## Example

Policy:

```
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-labels
spec:
  validationFailureAction: enforce
  rules:
  - name: check-for-labels
    match:
      requestTypes: ["DELETE"]
      resources:
        kinds:
        - Pod
    validate:
      message: "label 'app.kubernetes.io/name' is required"
      deny:
        conditions:
          any:
          - key: "{{request.operation}}"
            operator: Equals
            value: DELETE
      pattern:
        metadata:
          labels:
            app.kubernetes.io/name: "?*"
```

When a pod with label `app.kubernetes.io/name: "?*"` is requested to be deleted, the request will match with the requestTypes, but the validate.deny condition will block the delete operation.

Policy:

```
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-labels
spec:
  validationFailureAction: enforce
  rules:
  - name: check-for-labels
    match:
      requestTypes: ["DELETE"]
      resources:
        kinds:
        - Pod
    validate:
      message: "label 'app.kubernetes.io/name' is required"
        conditions:
          any:
          - key: "{{request.operation}}"
            operator: Equals
            value: DELETE
      pattern:
        metadata:
          labels:
            app.kubernetes.io/name: "?*"
```
When a pod with label `app.kubernetes.io/name: "?*"` is requested to be deleted, the request will match with the requestTypes, and the condition will also allow the delete operation for the pod.

## Implementation

After api changes and defining requestTypes, request.operation will be used as a build-in variable to match the operation(values) mentioned in requestTypes. If the operation matches, then conditions/preconditions will be checked/evaluated, and then according to the conditions, policy will be applied on to the resource.