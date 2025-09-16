# Meta
[meta]: #meta
- Name: Converting Kyverno Dynamic Lookups to Native Kubernetes Parameter Resources
- Start Date: 2025-09-15
- Author(s): [Mariam Fahmy](https://github.com/MariamFahmy98)

# Table of Contents
[table-of-contents]: #table-of-contents
- [Overview](#overview)
- [Definitions](#definitions)
- [Motivation](#motivation)
- [Proposal](#proposal)


# Overview

Kyverno and native Kubernetes policies (VAPs and MAPs) both enforce admission rules but handle external data differently.

- Kyverno allows policies to make live API calls using functions like `resource.Get()` and `resource.List()` to fetch this data on the fly.

- VAPs and MAPs use a declarative parameter resources (`paramKind` and `paramRef`) to inject external data into policy expressions. A policy declares what kind of data it needs (with `paramKind`), and a separate "binding" provides a reference to the specific resource (with `paramRef`).

This document compares these two methods and demonstrates how to translate Kyverno's dynamic lookups into the native parameter resources.

# Motivation

A common question we see in the community is whether Kyverno supports the same parameter resources as native Kubernetes policies. The answer is no; Kyverno achieves the same goal using its own dynamic lookup functions. This creates a challenge for anyone trying to automatically generate native VAPs or MAPs from the existing Kyverno policies, as there is no straightforward, one-to-one mapping for these dynamic functions.

Related issue: https://github.com/kyverno/kyverno/issues/13927

# Proposal

## Scenario 1: Converting resource.Get() (Fetching a Single, Known Resource)

This scenario covers policies that need to look up a single, specific resource. The following ValidatingPolicy checks that a Pod's name matches a value in a ConfigMap located in the Pod's own namespace. The lookup is dynamic and executed at runtime.

```yaml
apiVersion: policies.kyverno.io/v1alpha1
kind: ValidatingPolicy
metadata:
  name: disallow-host-path
spec:
  matchConstraints:
    resourceRules:
    - apiGroups:   [""]
      apiVersions: ["v1"]
      operations:  ["CREATE", "UPDATE"]
      resources:   ["pods"]
  variables:
    - name: cm
      expression: resource.Get("v1", "configmaps", object.metadata.namespace, "policy-cm")
  validations:
    - expression: object.metadata.name == variables.cm.data.name
```

The equivalent VAP using parameter resources would look like this:

```yaml
# Policy
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingAdmissionPolicy
metadata:
  name: "disallow-host-path"
spec:
  matchConstraints:
    resourceRules:
    - apiGroups:   [""]
      apiVersions: ["v1"]
      operations:  ["CREATE", "UPDATE"]
      resources:   ["pods"]
  paramKind:
    apiVersion: v1
    kind: ConfigMap
  validations:
    - expression: "object.metadata.name == params.data.name"
---
# VAP Binding
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingAdmissionPolicyBinding
metadata:
  name: "disallow-host-path-binding"
spec:
  policyName: "disallow-host-path"
  paramRef:
    name: "policy-cm"
    # Omitting 'namespace' makes the lookup dynamic, just like in Kyverno`
    # namespace: "default"
```

## Scenario 2: Converting resource.List() (Fetching and Filtering Multiple Resources)

This scenario explores more complex policies that fetch a list of resources and filter them based on field values. The following expression finds a specific ConfigMap by name within a list and checks the `data` field.

```yaml
resource.List("v1", "configmaps", "default").items[?metadata.name=='clusterregistries'].data.registries == "enabled"
```

When trying to convert this expression to use parameter resources, we encounter two limitations:

1. Selection is label-based: `paramRef` can only select multiple resources using a label selector. 

2. No filteration: parameter resources can't filter on other fields like `metadata.name`.

3. `params` variable references one resource at a time, not the entire list so we can't loop through parameter resources like the following:

```yaml
params.filter(p, p.metadata.name == 'clusterregistries').all(p, p.data.registries == "enabled")
```

The equivalent VAP would look like this, but it doesn't fully replicate the original Kyverno logic:

```yaml
# Policy
spec:
  paramKind:
    apiVersion: v1
    kind: ConfigMap
# Binding
spec:
  paramRef:
    selector:
      matchLabels:
        app: myapp
    namespace: "default"
    parameterNotFoundAction: Deny
```

The validation expression would be something like this:

```yaml
params.data["registries"] == "enabled"
```

Another limitation is that parameter resource supports getting configmaps or custom resources but not secrets for example. So if the Kyverno policy uses `resource.Get()` or `resource.List()` to fetch a Secret or any resource type, we cannot convert that to a parameter resource.