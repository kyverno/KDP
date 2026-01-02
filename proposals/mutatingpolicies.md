# New Policy Type - MutatingPolicy

Authors: Shuting Zhao
Created: February 18th, 2025
Updated: May 7th, 2025
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

The above examples demonstrate mutating resources via admission review process. The following attributes can be configured to enable mutation of existing resources.

* `spec.targetMatchConstraints`: optional, selects existing resources to mutate based on GVR, `namespaceSelector`, `objectSelector`, etc.
* `spec.evaluation.mutateExisting`: optional, enables mutation of existing resources. Default is `false`. When `spec.targetMatchConstraints` is not defined, Kyverno mutates existing resources matched in `spec.matchConstraints`.
* `spec.conditions`: optional, filters target resources via CEL-based conditions.

For example, the following policy mutates both existing configmaps and incoming configmaps with `CREATE` operation.

```yaml
apiVersion: policies.kyverno.io/v1alpha1
kind: MutatingPolicy
metadata:
  name: add-label-to-existing-secret
spec:
  # enables mutate existing feature
  evaluation:
    mutateExisting: true
  matchConstraints:
    resourceRules:
    - apiGroups:   [""]
      apiVersions: [v1]
      operations:  [CREATE]
      resources:   ["configmaps"]
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

This example defines a different target resource to be mutated, secrets. All existing secrets will be mutated when a new configmap is created.

```yaml
apiVersion: policies.kyverno.io/v1alpha1
kind: MutatingPolicy
metadata:
  name: add-label-to-existing-secret
spec:
  matchConstraints:
    resourceRules:
    - apiGroups:   [""]
      apiVersions: [v1]
      operations:  [CREATE]
      resources:   ["configmaps"]
  ## custom targets filters
  targetMatchConstraints:
  - # namespaceSelector:
    # objectSelector:
    # excludeResourceRules:
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

A `spec.conditions` field is supported for CEL-based conditional selections. A built-in variable `target` is supported to reference the target resource object. 

For example, the variable `target` is used below to filter secret's namespace via `conditions`, and to reference `.spec` object of the secret.

```yaml
  targetsMatchConstraints:
  - resourceRules:
    - apiGroups:   [""]
      apiVersions: [v1]
      resources:   ["secrets"]
  conditions:
  - name: "skip kube-system namespace"
    expression: >-
      target.metadata.namespace != "kube-system"
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