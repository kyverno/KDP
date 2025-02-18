# New Policy Type - MutatingPolicy

Authors: Shuting Zhao
Created: February 18th, 2025
Abstract: MutatingPolicy for Kyverno

## Overview

Kubernetes provides [MutatingAdmissionPolicies](https://kubernetes.io/docs/reference/access-authn-authz/mutating-admission-policy/) as a native mechanism for performing resource mutations, complementing traditional mutating admission webhooks. However, Kyverno takes this capability to the next level by introducing a new `MutatingPolicy` type as part of its [Kyverno Design Proposal - New Policy Types](https://github.com/kyverno/KDP/pull/66). This innovative approach not only aligns with Kubernetes' in-tree solutions but also enhances them by integrating Kyverno's robust, built-in features.

## Proposal

Listed [features](https://github.com/kyverno/KDP/blob/5f3c816181bd3eee631d4c4a60224b045f9e512a/proposals/new_policy_types.md#validating-policy) under `ValidatingPolicy` are applicable to `MutatingPolicy`. In addition to these, this proposal clarifies how mutate existing ability is supported in `MutatingPolicy`, and showcases complex examples using mutatingpolicies.

### Mutation patterns

These two examples showcase the two supported mutation patch patterns `ApplyConfiguration` and `JSONPatch`.

```yaml
apiVersion: policies.kyverno.io/v1alpha1
kind: MutatingPolicy
metadata:
  name: "sidecar-policy.example.com"
spec:
  paramKind:
    kind: Sidecar
    apiVersion: mutations.example.com/v1
  matchConstraints:
    resourceRules:
    - apiGroups:   ["apps"]
      apiVersions: ["v1"]
      operations:  ["CREATE"]
      resources:   ["pods"]
  matchConditions:
    - name: does-not-already-have-sidecar
      expression: "!object.spec.initContainers.exists(ic, ic.name == \"mesh-proxy\")"
  failurePolicy: Fail
  reinvocationPolicy: IfNeeded
  mutations:
    - patchType: "ApplyConfiguration"
      applyConfiguration:
        expression: >
          Object{
            spec: Object.spec{
              initContainers: [
                Object.spec.initContainers{
                  name: "mesh-proxy",
                  image: "mesh/proxy:v1.0.0",
                  args: ["proxy", "sidecar"],
                  restartPolicy: "Always"
                }
              ]
            }
          }          
```

```yaml
apiVersion: policies.kyverno.io/v1alpha1
kind: MutatingPolicy
metadata:
  name: "sidecar-policy.example.com"
spec:
  paramKind:
    kind: Sidecar
    apiVersion: mutations.example.com/v1
  matchConstraints:
    resourceRules:
    - apiGroups:   [""]
      apiVersions: ["v1"]
      operations:  ["CREATE"]
      resources:   ["pods"]
  matchConditions:
    - name: does-not-already-have-sidecar
      expression: "!object.spec.initContainers.exists(ic, ic.name == \"mesh-proxy\")"
  failurePolicy: Fail
  reinvocationPolicy: IfNeeded
  mutations:
    - patchType: "JSONPatch"
      jsonPatch:
        expression: >
          [
            JSONPatch{
              op: "add", path: "/spec/initContainers/-",
              value: Object.spec.initContainers{
                name: "mesh-proxy",
                image: "mesh-proxy/v1.0.0",
                restartPolicy: "Always"
              }
            }
          ]          
```

### Mutate Existing

In order to mutate existing resources, a `targetConstraints` is added to custom existing resources:

* `targetConstraints.resourceRules`: required, selects resources based on GVR.
* `resourceRules.context`: optional, additional context lookup to filter targets.
* `resourceRules.selector`: optional, filters targets via label selectors.
* `resourceRules.matchConditions`: optional, filters target via CEL-based conditions.

By default `spec.mutateExistingOnPolicyUpdate` is disabled, set it to `true` to mutate existing resources based on policy events.

```yaml
apiVersion: policies.kyverno.io/v1alpha1
kind: MutatingPolicy
metadata:
  name: add-label-to-existing-secret
spec:
  mutateExistingOnPolicyUpdate: true
  matchConstraints:
    resourceRules:
    - apiGroups:   [""]
      apiVersions: [v1]
      operations:  [CREATE]
      resources:   ["configmaps"]
  ## custom targets filters
  targetConstraints:
    resourceRules:
    - apiGroups:   [""]
      apiVersions: [v1]
      resources:   ["secrets"]
  mutations:
    - patchType: JSONPatch
      jsonPatch: 
        expression: >-
          [
            JSONPatch{
              op: "add", path: "/metadata/labels",
              value: {foo: bar}
            }
          ] 
```

The built-in variable `target` is supported to reference target object, for example:

```yaml
  mutations:
    - patchType: "JSONPatch"
      jsonPatch:
        expression: >
          [
            JSONPatch{
              op: "replace", path: "/spec/tls",
              value: target.spec.tls.map(
                t, 
                {
                  "secretName": t.secretName, 
                  "hosts": dyn(t.hosts.map(h, h.replace(".old.com", ".new.com")))
                }
              )
            }
          ]
```

### Complex use cases