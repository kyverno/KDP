# Kyverno Design Proposal - Mutate Existing Resources

Created: March 21st, 2022
Author: [Shuting Zhao](https://github.com/realshuting)


Table of contents:
  * [Overview](#overview)
  * [Proposal](#proposal)
  * [Examples](#examples)
  * [Implementation](#implementation)
    + [Watch Trigger Resources](#watch-trigger-resources)
      - [1. Admission Webhook](#1-admission-webhook)
      - [2. Event Watcher](#2-event-watcher)
    + [Apply Policies](#apply-policies)
    + [Mutate Existing Resources](#mutate-existing-resources)

<small><i><a href='http://ecotrust-canada.github.io/markdown-toc/'>Table of contents generated with markdown-toc</a></i></small>


## Overview

Kyverno webhook controller runs as an admission controller in Kubernetes, and it mutates resources upon admission review. Such mutation is performed based on resources operations (CREATE/UPDATE/DELETE) via webhook.

As of release 1.6.1, these are still the outstanding issues which Kyverno needs to support hence this proposal:

- Mutate target resource which is different from the trigger resource, [#2139](https://github.com/kyverno/kyverno/issues/2139), [#1722](https://github.com/kyverno/kyverno/issues/1722).
- Mutate trigger resources based on policy update, [#1607](https://github.com/kyverno/kyverno/issues/1607).

*The **trigger** resource is matched in the policy while the **target** resource is the resource to be mutated*.


## Proposal

To support mutate existing resources:

- `spec.rules.match` is required in such rules to identify the *trigger* resource. i.e., Example 1.
- A new attribute `spec.rules.mutate.targets` will be introduced to specify a list of target resources to be mutated.
- Kyverno will fetch and mutate the existing resources once the trigger resource changes, the user needs to grant Kyverno permissions to operate on target resources.
- The patches will be added as an annotation to existing resources.
- Generates events on existing resources that are mutated.
- CLI support with the apply command, similar to background generation, to allow users to understand the impact such a policy may have on existing resources.
- Publishes metric data corresponding to existing resources that are mutated.




## Examples

1. This example adds the label `foo=bar` to deployment `staging/test-deploy` on configmap `staging/dictionary` update.

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: "..."
spec:
  rules:
    - name: "..."
      match:
        any:
        - resources:
            kinds:
            - ConfigMap
            names:
            - dictionary
            namespaces:
            - staging
      preconditions:
        any:
        - key: "{{ request.operation }}"
          operator: Equals
          value: UPDATE
      mutate:
        # specify target resources to be mutated
        targets:
        - apiVersion: apps/v1
          kind: Deployment
          name: test-deploy
          namespace: staging
        patchStrategicMerge:
          metadata:
            labels:
              foo: bar
```

2. This example restarts the deployment `staging/test-deploy` by updating the timestamp annotation on secret update.
```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: "..."
spec:
  rules:
    - name: "..."
      match:
        any:
        - resources:
            kinds:
            - Secret
            names:
            - certificate-store
            namespaces:
            - staging
      mutate:
        targets:
        - apiVersion: apps/v1
          kind: Deployment
          name: test-deploy
          namespace: staging
        patchStrategicMerge:
          metadata:
            annotations:
              kyverno.io/since-last-update: "{{ time_since('', '{{ request.object.metadata.creationTimestamp }}', '') }}"
```

3. This example adds label `foo=bar` to both incoming and existing deployments on policy CREATE and UPDATE events with `mutateExistingOnPolicyUpdate=true`.

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: "..."
spec:
  mutateExistingOnPolicyUpdate: true
  rules:
    - name: "..."
      match:
        any: 
        - resources:
            kinds:
            - Deployment
      mutate:
        targets:
        - apiVersion: apps/v1
          kind: Deployment
          name: foo
          namespace: "{{ request.operation.metadata.namespace }}"
        patchStrategicMerge:
          metadata:
            annotations:
              foo: bar
```

## Implementation

### Watch Trigger Resources

There are two ways to "watch" the trigger resource's activities:

#### 1. Admission Webhook

This approach leverages existing logic to register trigger resources in the webhook to receive events. It requires minimum code change and is consistent with other components, i.e., the Generate Controller.

#### 2. Event Watcher

An alternative is to register event watchers for trigger resources dynamically and de-register when a policy is removed. The solution lacks information about user context contained in the admission requests.

Based on the above analysis, we will use the admission webhook to watch trigger resources.

### Apply Policies

Kyverno will mutate target(existing) resources in the background, not through the webhook. Since the trigger resources are watched via admission webhook (all instances will be active in HA), we need to create an intermediate resource to store the change request, and reconcile it in the background to apply such policies, similarly to how the generate controller reconciles generate requests. Once an event is received in the admission webhook, Kyverno will block the request until it creates the change request to the cluster. In this way, Kyverno is able to recover from failures in case it crashes in between.

### Mutate Existing Resources

The dynamic client will be used to fetch and update the target resources.

### Link to the Implementation PR
https://github.com/kyverno/kyverno/pull/3669