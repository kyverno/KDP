# Kyverno Design Proposal - Adding Request Types to match and exclude

Author: [Prateek Nandle](https://github.com/Prateeknandle)

PR: [#3619](https://github.com/kyverno/kyverno/pull/3619)

## Contents

- [Kyverno Design Proposal - Adding Request Types to match and exclude](#kyverno-design-proposal---adding-request-types-to-match-and-exclude)
  * [Introduction](#introduction)
  * [Proposal](#proposal)
  * [Example](#example)
  * [Implementation](#implementation)

## Introduction

Until now we use `request.operation` under `condition.key`, to filter a request, after this proposal we'll use a tag `requestTypes` in `match` or `exclude` block to match/exclude specific requests coming from admission review requests.

## Proposal

After an request is made, the Kubernetes api will send the information about the requests to the registered webhook through admission review request. The value of `requestTypes` tag  under `match/exclude` block will be matched with the incomming requests and, if request is matched, the following rules under the policy will be applied else not.

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
      resources:
        kinds:
        - Pod
    exclude:
      requestTypes: ["CREATE"]
    validate:
      message: "label 'app.kubernetes.io/name' is required"
      pattern:
        metadata:
          labels:
            app.kubernetes.io/name: "?*"
```

When a Pod is requested to be created, the request will match with the `requestTypes` under the exclude block and the request from the user, for creating a Pod will be rejected.

## Implementation

After api changes and defining `requestTypes`, `request.operation` will be used as a build-in variable, kubernetes-api will send request-type information to it and we can use that information to match with the value of `requestTypes`. And if the request matches the rules will be appiled on to the resource else not.