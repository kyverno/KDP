# Meta
[meta]: #meta
- Name: The new CEL Generating Policies
- Start Date: 2025-05-09
- Author(s): [Mariam Fahmy](https://github.com/MariamFahmy98)

# Table of Contents
[table-of-contents]: #table-of-contents
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

This proposal introduces GeneratingPolicy, a new Kyverno Custom Resource Definition (CRD). This policy offers an alternative, CEL-driven approach to generating Kubernetes resources, designed to provide the same core functionalities as Kyverno's existing generate rules. By leveraging Common Expression Language (CEL) for its conditional logic and data manipulation, GeneratingPolicy aims for enhanced flexibility, expressiveness, and alignment with evolving Kubernetes CEL standards (e.g., ValidatingAdmissionPolicies, MutatingAdmissionPolicies).

# Definitions
[definitions]: #definitions

1. Trigger: It refers to the resource responsible for triggering the generate rule as defined in a combination of `match` and `exclude` blocks.

2. Downstream/target: It refers to the generated resource(s).

3. Source: It refers to the clone source when cloning new resources from existing ones.

# Motivation
[motivation]: #motivation

As the Kubernetes ecosystem and feature set has evolved, CEL has become the preferred method for defining build-in policies. As part of our initiative to align our policy APIs with the new Kubernetes build-in types, we want to provide new policy types for our existing feature set to align with these standards, improve usability and performance, and lower the barrier to entry for new users.

# Proposal
[proposal]: #proposal

This proposal introduces a new CRD, GeneratingPolicy, which enables users to declaratively generate Kubernetes resources using CEL expressions. The GeneratingPolicy will support both data sources (defining resources directly in the policy) and clone sources (cloning from existing resources). Also, it will support generate resources based on existing resources.

## Data Source

In case of defining the generated resource in the policy directly, the GeneratingPolicy CRD will look like this:

```yaml
apiVersion: policies.kyverno.io/v1alpha1
kind: GeneratingPolicy
metadata:
  name: zk-kafka-address
spec:
  evaluation:
    synchronize: true
    generateExisting: false
    orphanDownstreamOnPolicyDelete: false
  matchConstraints:
    resourceRules:
    - apiGroups:   [""]
      apiVersions: ["v1"]
      operations:  ["CREATE", "UPDATE"]
      resources:   ["namespaces"]
  matchConditions:
    - expression: "object.metadata.labels['color'] == 'red'"
      name: "red-label"
  variables:
    - name: nsName
      expression: "object.metadata.name"
    - name: configmap
      expression: >-
        {
          "kind": dyn("ConfigMap"),
          "apiVersion": dyn("v1"),
          "data": dyn({
            "zk-kafka-address": "192.168.10.10:2181,192.168.10.11:2181,192.168.10.12:2181"
          })
        }
  generate:
    - expression: generator.ApplyAll(variables.nsName, variables.configmap)
      name: generate-configmap
```

### Synchronization Behavior

The following table describes the behavior of deletion and modification events on the policy, trigger, and downstream resources.

| Action                                                               | **Sync Effect**      | **No Sync Effect**    |
| -------------------------------------------------------------------- | -------------------- | --------------------- |
| **Delete Downstream**                                                    | Downstream recreated | Downstream deleted    |
| **Delete Policy**<br>generate.orphanDownstreamOnPolicyDelete: true  | Downstream retained  | Downstream retained   |
| **Delete Policy**<br>generate.orphanDownstreamOnPolicyDelete: false | Downstream deleted   | Downstream retained   |
| **Delete Trigger**                                                       | Downstream deleted   | None                  |
| **Modify Downstream**                                                    | Downstream reverted  | Downstream modified   |
| **Modify Policy**                                                   | Downstream synced    | Downstream unmodified |
| **Modify Trigger**                                                       | Downstream deleted   | None                  |


## Clone Source

In case of cloning an existing resource, the GeneratingPolicy CRD will look like this:

```yaml
apiVersion: policies.kyverno.io/v1alpha1
kind: GeneratingPolicy
metadata:
  name: zk-kafka-address
spec:
  evaluation:
    synchronize: true
    generateExisting: false
    orphanDownstreamOnPolicyDelete: false
  matchConstraints:
    resourceRules:
    - apiGroups:   [""]
      apiVersions: ["v1"]
      operations:  ["CREATE", "UPDATE"]
      resources:   ["namespaces"]
  matchConditions:
    - expression: "object.metadata.labels['color'] == 'red'"
      name: "red-label"
  variables:
    - name: nsName
      expression: "object.metadata.name"
    - name: source
      expression: resource.Get("v1", "configmaps", "default", "allowed-registry")
  generate:
    - expression: generator.ApplyAll(variables.nsName, variables.source)
      name: generate-configmap
```

In case of cloning multiple resources, the GeneratingPolicy CRD will look like this:

```yaml
apiVersion: policies.kyverno.io/v1alpha1
kind: GeneratingPolicy
metadata:
  name: zk-kafka-address
spec:
  evaluation:
    synchronize: true
    generateExisting: false
    orphanDownstreamOnPolicyDelete: false
  matchConstraints:
    resourceRules:
    - apiGroups:   [""]
      apiVersions: ["v1"]
      operations:  ["CREATE", "UPDATE"]
      resources:   ["namespaces"]
  variables:
    - name: nsName
      expression: "object.metadata.name"
    - name: cmList
      expression: resource.List("v1", "configmaps", "default").items
    - name: sourceList
      expression: variables.cmList.filter(source, source.metadata.labels['allowedToBeCloned'] == 'true')
  generate:
    - expression: generator.ApplyAll(variables.nsName, variables.sourceList)
      name: generate-configmaps
```

The clone source is fetched using CEL expressions, either through `resource.Get()` or `resource.List()`.

### `forEach` with Data Source

This example demonstrates how to create a NetworkPolicy into a list of existing namespaces which is stored as a comma-separated string in a ConfigMap. It is equivalent to a generate rule that makes use of `foreach` with a data source.

Asuming that we have a ConfigMap with the following data:
```yaml
kind: ConfigMap
apiVersion: v1
metadata:
  name: default-deny
  namespace: default
data:
  namespaces: foreach-ns-1,foreach-ns-2,foreach-ns-3
```

```yaml
apiVersion: policies.kyverno.io/v1alpha1
kind: GeneratingPolicy
metadata:
  name: foreach-generate-data
spec:
  evaluation:
    synchronize: true
    generateExisting: false
    orphanDownstreamOnPolicyDelete: false
  matchConstraints:
    resourceRules:
    - apiGroups:   [""]
      apiVersions: ["v1"]
      operations:  ["CREATE", "UPDATE"]
      resources:   ["configmaps"]
  variables:
    - name: nsList
      expression: "object.data.namespaces.split(',')"
    - name: filteredNs
      expression: "variables.nsList.filter(ns, ['foreach-ns-1', 'foreach-ns-2'].exists(v, v == ns))"
    - name: targetResources
      expression: >
        variables.filteredNs.map(ns,
          {
            'apiVersion': dyn('networking.k8s.io/v1'),
            'kind': dyn('NetworkPolicy'),
            'metadata': {
              'name': 'my-networkpolicy-' + string(ns),
              'namespace': string(ns),
            },
            'spec': {
              'podSelector': {},
              'policyTypes': ['Ingress', 'Egress']
            }
          }
        )
  generate:
    - name: generate-network-policies
      expression: >
        variables.filteredNs.map(ns,
          generator.ApplyAll(ns, variables.targetResources)
        )
```

### `forEach` with Clone Source

This example demonstrates how to clone a Secret into each namespace listed in the triggering ConfigMap.

```yaml
apiVersion: policies.kyverno.io/v1alpha1
kind: GeneratingPolicy
metadata:
  name: foreach-clone
spec:
  evaluation:
    synchronize: true
    generateExisting: false
    orphanDownstreamOnPolicyDelete: false
  matchConstraints:
    resourceRules:
    - apiGroups:   [""]
      apiVersions: ["v1"]
      operations:  ["CREATE", "UPDATE"]
      resources:   ["configmaps"]
  variables:
    - name: nsList
      expression: "object.data.namespaces.split(',')"
    - name: source
      expression: resource.Get("v1", "secrets", "default", "source-secret")
    - name: targets
      expression: >
        variables.nsList.map(ns,
          {
            "apiVersion": variables.source.apiVersion,
            "kind": variables.source.kind,
            "metadata": {
              "name": variables.source.metadata.name,
              "namespace": ns,
              "labels": variables.source.metadata.labels,
              "annotations": variables.source.metadata.annotations
            },
            "data": variables.source.data,
            "type": variables.source.type
          }
        )
  generate:
    - name: foreach-clone-secrets
      expression: >
        variables.nsList.map(ns,
          generator.ApplyAll(ns, variables.targets)
        )
```

An alternative approach is to introduce a new function, `object.setField(key, value)`, which will return a new object with the specified field path overridden.

```yaml
apiVersion: policies.kyverno.io/v1alpha1
kind: GeneratingPolicy
metadata:
  name: foreach-clone
spec:
  evaluation:
    synchronize: true
    generateExisting: false
    orphanDownstreamOnPolicyDelete: false
  matchConstraints:
    resourceRules:
      - apiGroups:   [""]
        apiVersions: ["v1"]
        operations:  ["CREATE", "UPDATE"]
        resources:   ["configmaps"]
  variables:
    - name: nsList
      expression: "object.data.namespaces.split(',')"
    - name: source
      expression: resource.Get("v1", "secrets", "default", "source-secret")
    - name: targets
      expression: >
        variables.nsList.map(ns,
          variables.source.setField("metadata.namespace", ns)
        )
  generate:
    - name: foreach-clone-secrets
      expression: >
        variables.nsList.map(ns,
          generator.ApplyAll(ns, variables.targets)
        )
```

### `forEach` with CloneList Source
This example demonstrates how to clone multiple Secrets into each namespace listed in the triggering ConfigMap.

```yaml
apiVersion: policies.kyverno.io/v1alpha1
kind: GeneratingPolicy
metadata:
  name: foreach-clone-list
spec:
  evaluation:
    synchronize: true
    generateExisting: false
    orphanDownstreamOnPolicyDelete: false
  matchConstraints:
    resourceRules:
    - apiGroups:   [""]
      apiVersions: ["v1"]
      operations:  ["CREATE", "UPDATE"]
      resources:   ["configmaps"]
  variables:
    - name: nsList
      expression: "object.data.namespaces.split(',')"
    - name: secretList
      expression: resource.List("v1", "secrets", "default").items
    - name: sourceList
      expression: variables.secretList.filter(source, source.metadata.labels['allowedToBeCloned'] == 'true')
  generate:
    - name: foreach-clone-multiple-secrets
      expression: >
        variables.nsList.map(ns,
          variables.sourceList.map(secret,
            generator.ApplyAll(ns, secret)
          )
        )
```

An alternative approach is to make use of `object.setField(key, value)` to set the namespace for each secret and use `generator.ApplyAll()` to generate the secrets.

```yaml
apiVersion: policies.kyverno.io/v1alpha1
kind: GeneratingPolicy
metadata:
  name: foreach-clone-list
spec:
  evaluation:
    synchronize: true
    generateExisting: false
    orphanDownstreamOnPolicyDelete: false
  matchConstraints:
    resourceRules:
    - apiGroups:   [""]
      apiVersions: ["v1"]
      operations:  ["CREATE", "UPDATE"]
      resources:   ["configmaps"]
  variables:
    - name: nsList
      expression: "object.data.namespaces.split(',')"
    - name: secretList
      expression: resource.List("v1", "secrets", "default").items
    - name: sourceList
      expression: variables.secretList.filter(source, source.metadata.labels['allowedToBeCloned'] == 'true')
    - name: targets
      expression: >
        variables.nsList.map(ns,
          variables.sourceList.map(secret,
            secret.setField("metadata.namespace", ns)
          )
        )
  generate:
    - name: foreach-clone-multiple-secrets
      expression: >
        variables.nsList.map(ns,
          variables.sourceList.map(secret,
            generator.ApplyAll(ns, secret)
          )
        )
```

### Synchronization Behavior

The following table describes the behavior of deletion and modification events on the policy, trigger, and downstream resources.

| Action                 | **Sync Effect**       | **No Sync Effect**    |
| ---------------------- | --------------------- | --------------------- |
| **Delete Downstream**  | Downstream recreated  | Downstream deleted    |
| **Delete Policy**      | Downstream retained   | Downstream retained   |
| **Delete Source**      | Downstream deleted    | Downstream retained   |
| **Delete Trigger**     | Downstream deleted    | None                  |
| **Modify Downstream**  | Downstream reverted   | Downstream modified   |
| **Modify Policy**      | Downstream unmodified | Downstream unmodified |
| **Modify Source**      | Downstream synced     | Downstream unmodified |
| **Modify Trigger**     | Downstream deleted    | None                  |

