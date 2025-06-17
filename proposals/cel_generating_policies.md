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
- [Data Source](#data-source)
- [Clone Source](#clone-source)
  - [`forEach` with Data Source](#foreach-with-data-source)
  - [`forEach` with Clone Source](#foreach-with-clone-source)
  - [`forEach` with CloneList Source](#foreach-with-clonelist-source)
- [Implementation](#implementation)
  - [Implementation Details of `generate` Rules](#implementation-details-of-generate-rules)
    - [When Are UpdateRequests Created?](#when-are-updaterequests-created)
    - [Policy Controller](#policy-controller)
  - [Implementation Details of the new `GeneratingPolicy` CRD](#implementation-details-of-the-new-generatingpolicy-crd)
    - [Trigger Changes](#trigger-changes)
    - [Policy Changes](#policy-changes)
    - [Cloned Source & Downstream Changes](#cloned-source--downstream-changes)
      - [Challenge: Discovering GVRs for Watchers](#challenge-discovering-gvrs-for-watchers)

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
        [
          {
            "kind": dyn("ConfigMap"),
            "apiVersion": dyn("v1"),
            "metadata": dyn({
              "name": "zk-kafka-address",
              "namespace": string(variables.nsName),
            }),
            "data": dyn({
              "KAFKA_ADDRESS": "192.168.10.13:9092,192.168.10.14:9092,192.168.10.15:9092",
              "ZK_ADDRESS": "192.168.10.10:2181,192.168.10.11:2181,192.168.10.12:2181"
            })
          }
        ]
  generate:
    - expression: generator.Apply(variables.nsName, variables.configmap)
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
  name: generate-secret
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
    - name: source
      expression: resource.Get("v1", "secrets", "default", "regcred")
  generate:
    - expression: generator.Apply(variables.nsName, [variables.source])
```

In case of cloning multiple resources, the GeneratingPolicy CRD will look like this:

```yaml
apiVersion: policies.kyverno.io/v1alpha1
kind: GeneratingPolicy
metadata:
  name: generate-secrets
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
    - name: sources
      expression: resource.List("v1", "secrets", "default")
  generate:
    - expression: generator.Apply(variables.nsName, [variables.sources])
```

```yaml
apiVersion: policies.kyverno.io/v1alpha1
kind: GeneratingPolicy
metadata:
  name: generate-secrets-and-networkpolicies
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
    - name: clonedSecrets
      expression: resource.List("v1", "secrets", "default")
    - name: clonedNetworkPolicies
      expression: resource.List("networking.k8s.io/v1", "networkpolicies", "default")
  generate:
    - expression: generator.Apply(variables.nsName, [variables.clonedSecrets, variables.clonedNetworkPolicies])
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

# Implementation

## Implementation Details of `generate` Rules

This sections explains how UpdateRequests (URs) are created and consumed by the background controller to manage downstream resources.

Both the trigger and downstream resources are added to the validating webhook rules. Then, the admission controller will create an UpdateRequest (UR). The UR will contain the following information:

```yaml
apiVersion: kyverno.io/v2
kind: UpdateRequest
metadata:
  generateName: ur-
  labels:
    generate.kyverno.io/policy-name: cpol-sync-clone
  ...
spec:
  context:
    admissionRequestInfo:
      admissionRequest:
        operation: CREATE
        kind:
          kind: Namespace
          version: v1
        object:
          metadata:
            name: ns-4
  policy: cpol-sync-clone
  requestType: generate
  deleteDownstream: false
  ruleContext:
  - rule: clone-secret
    trigger:
      kind: Namespace
      name: ns-4
status:
  state: Pending
```

These UpdateRequest resources are then consumed by the background controller, which is responsible for generating or deleting downstream resources accordingly.

### When Are UpdateRequests Created?

The admission controller generates UpdateRequest (UR) resources under several conditions:

1. **Trigger Creation or Update**
When a trigger resource is created and matches a generate rule, Kyverno creates a UR to generate the corresponding downstream resource.

2. **Trigger Deletion**
If a trigger is deleted, or no longer matches the rule, and the `synchronize` field is set to true, Kyverno creates a UR to delete the downstream resource.

3. **Generate on DELETE Operation**
If the generate rule is configured to act on the DELETE operation of the trigger resource, Kyverno creates a UR on trigger deletion.

4. **Cloned Source Changes or Downstream Changes**
When the cloned source resource changes, or if the downstream resource is modified, and the `synchronize` field is set to true Kyverno creates a UR to update the downstream resource.

    **In case the cloned source changes:**

    a. First, we fetch the downstream resources that have the following labels:

    ```
    generate.kyverno.io/source-group: ""
    generate.kyverno.io/source-kind: Secret
    generate.kyverno.io/source-namespace: default
    generate.kyverno.io/source-uid: b76c7fb3-7de5-4803-aecc-c1dffd3c4936
    generate.kyverno.io/source-version: v1
    ```

    b. Then, from the downstream resource, we can get information about the policy name, the rule name and the trigger from the following labels:

    ```
    generate.kyverno.io/policy-name: cpol-sync-clone
    generate.kyverno.io/policy-namespace: ""
    generate.kyverno.io/rule-name: clone-secret
    generate.kyverno.io/trigger-group: ""
    generate.kyverno.io/trigger-kind: Namespace
    generate.kyverno.io/trigger-namespace: ""
    generate.kyverno.io/trigger-uid: 130f5def-4c75-475d-8be5-5514f25ce508
    generate.kyverno.io/trigger-version: v1
    ```

    c. Finally, we can create a UR to update the downstream resource.

    **In case the downstream resource is modified:**

    a. We get the policy name, rule name and trigger from the labels:

    ```
    generate.kyverno.io/policy-name: cpol-sync-clone
    generate.kyverno.io/policy-namespace: ""
    generate.kyverno.io/rule-name: clone-secret
    generate.kyverno.io/trigger-group: ""
    generate.kyverno.io/trigger-kind: Namespace
    generate.kyverno.io/trigger-namespace: ""
    generate.kyverno.io/trigger-uid: 130f5def-4c75-475d-8be5-5514f25ce508
    generate.kyverno.io/trigger-version: v1
    ```

    b. Then, we can create a UR to revert the downstream resource.

In addition, URs are generated on policy events via the policy controller. The policy controller is responsible for creating URs when a policy is created, updated, or deleted. This ensures that the downstream resources are always in sync with the policy.

### Policy Controller

The policy controller, part of the background controller, generates URs during policy events (e.g., policy/rule updates, deletions) and also responsible for the `generateExisting` field. This field determines whether to generate downstream resources for existing trigger resources when the policy is created.

1. **When the Policy Definition Changes**

    If the generate rule embeds resource data (e.g., in a `data` field) and `synchronize` is enabled, Kyverno must sync downstreams by generating new URs.

    - First, we fetch the downstream resources that have the following labels:

    ```
    generate.kyverno.io/policy-name: zk-kafka-address
    generate.kyverno.io/policy-namespace: ""
    generate.kyverno.io/rule-name: k-kafka-address
    app.kubernetes.io/managed-by: kyverno
    ```

    - Then, we extract trigger metadata from downstream labels and creates URs accordingly.

2. **When a Policy or Rule is Deleted (with `synchronize: true`)**

    Kyverno must clean up downstreams when:

    - A policy is deleted.
    - A rule within the policy is removed.

    In such cases, Kyverno fetches downstreams using the same labels as above. It then extracts trigger information and creates a UR to delete the downstream resource.

3. **For `generateExisting` Feature**

    When a generate rule uses `generateExisting`, the policy controller:

    -  Lists all existing trigger resources.

    - Creates URs to generate downstreams for each applicable trigger.
  

## Implementation Details of the new `GeneratingPolicy` CRD

To enable synchronization, the `synchronize` field must be set to true. When enabled, any change to:

- the trigger resource (specified via `matchConstraints`),
- the GeneratingPolicy itself, or
- the cloned source resource,
- the downstream resource

will result in the regeneration or deletion of the corresponding downstream resources.

### Trigger Changes

For the trigger resource, it is added to the validating webhook rules. The admission controller will then create an UpdateRequest (UR) to generate the downstream resource(s).

### Policy Changes

For the GeneratingPolicy itself, the PolicyController will be responsible for syncing the downstream resources with the policy changes. It will watch for changes to the GeneratingPolicy CRD and create URs accordingly. The URs will contain the necessary information to update/delete the downstream resources based on the policy changes. For example, if a GeneratingPolicy is updated to change the data source, the PolicyController will create URs to update the downstream resources accordingly. Similarly, if a GeneratingPolicy is deleted, the PolicyController will create URs to delete the downstream resources.

### Cloned Source & Downstream Changes

1. Cloned Source Changes:

    If a policy clones from a source, modifications to that source must be detected.

    Dynamic watchers will be used to monitor the source GVR and trigger a regeneration if:

    - the source is modified → update the downstream resource.

    - the source is deleted → delete the downstream resource

2. Downstream Changes:

    If a downstream resource is externally modified, we need to revert it back to the policy-defined state.

    To handle this, we will also use dynamic watchers to monitor the downstream resource. If the downstream resource is modified, the watcher will trigger an update to revert it back to the state defined in the GeneratingPolicy.

#### Challenge: Discovering GVRs for Watchers

To watch for cloned or downstream resource changes, we must know the GroupVersionResource (GVR) of the involved resources. However, since both the source and the downstream resources are defined via CEL expressions, their GVRs are not known at the policy creation time.

There are three main approaches to solve this:

1. Evaluate CEL expressions when policy is created:

    CEL expressions will be evaluated immediately when a GeneratingPolicy is created or updated. However, this raises a challenge: how can we perform a dry-run evaluation of these expressions without actually creating any resources? In particular, expressions like `object.metadata.name` rely on the presence of an incoming resource (object), which does not exist at the time of policy creation. We need to determine how to safely evaluate such expressions in the absence of runtime context.

2. Explicit GVR Definition in Policy: 

    We can add a GVR field in the GeneratingPolicy definition, which will be set to the GVR of the cloned source resource and the downstream resource. It will help us bypass evaluating CEL just to know which resources to watch.

    ```yaml
    apiVersion: policies.kyverno.io/v1alpha1
    kind: GeneratingPolicy
    metadata:
      name: generate-secrets-and-networkpolicies
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
      generatedResources:
        - apiGroups: [""]
          apiVersions: ["v1"]
          kind: "Secret"
        - apiGroups: ["networking.k8s.io"]
          apiVersions: ["v1"]
          kind: "NetworkPolicy"
      variables:
        - name: nsName
          expression: "object.metadata.name"
        - name: clonedSecrets
          expression: resource.List("v1", "secrets", "default")
        - name: clonedNetworkPolicies
          expression: resource.List("networking.k8s.io/v1", "networkpolicies", "default")
      generate:
        - expression: generator.Apply(variables.nsName, [variables.clonedSecrets, variables.clonedNetworkPolicies])
    ```

3. Lazy Evaluation on First Execution: 

    a. CEL expressions will be evaluated lazily during the first invocation of `generator.Apply(...)`.

    b. On first successful execution:

      1. Extract the GVRs of cloned and generated resources from the result.

      2. Register watchers dynamically.

      3. Maintain a cache of registered GVRs to prevent duplication.

### Lazy Evaluation on First Execution

Workflow:

1. A user creates a GeneratingPolicy.

2. A trigger resource is created that matches the GeneratingPolicy.

3. An UpdateRequest (UR) is created to generate the downstream resource.

4. The background controller processes the UR and evaluates the CEL expressions.

5. During the first execution of `generator.Apply(...)`, we call `watchManager.SyncWatchers()`, passing the policy name and the list of the generated resources.

    ```go
    type Resource struct {
      Name      string
      Namespace string
      Hash      string
      Labels    map[string]string
      Data      *unstructured.Unstructured
    }

    type watcher struct {
      watcher       watch.Interface
      metadataCache map[types.UID]Resource
    }

    type WatchManager struct {
      // clients
      client dclient.Interface

      // mapper
      restMapper meta.RESTMapper

      // dynamicWatchers refrences the GVR of the generated resources to the corresponding watcher.
      dynamicWatchers map[schema.GroupVersionResource]*watcher
      // policyRefs maps the policy name to the set of generated resources.
      policyRefs map[string][]schema.GroupVersionResource
      // refCount tracks the number of policies that generates the same resource.
      refCount map[schema.GroupVersionResource]int

      log  logr.Logger
      lock sync.Mutex
    }
    ```

    - The `SyncWatchers()` does the following:
    
        a. For each new GVR, it starts a dynamic watcher. To be efficient, it uses a reference counter (refCount). If two policies both generate ConfigMaps, only one watcher for ConfigMaps is created.

        b. It compares the new set of GVRs with the old set for that policy. If a policy is updated to no longer generate a certain resource kind, the reference counter for that GVR is decremented. If the count reaches zero, the corresponding watcher is stopped.

        c. It populates an internal `metadataCache` with details (name, namespace, hash) of the downstream resources.

    - Once a watcher is running for a resource kind (e.g., ConfigMap), it receives events for all resources of that kind cluster-wide.

    - On UPDATE event:

        a. The `handleUpdate` function checks if the updated resource is one of the downstream resources it manages (by looking it up in its `metadataCache`).
        
        b. If it is a downstream resource, it calculates the hash of the incoming resource and compares it to the hash stored in the cache.
        
        c. If the hashes differ, it means a user has modified the resource. The WatchManager then reverts the change by re-applying the original, desired state from its cache. This is done by getting the original resource from the cache.

        d. If it is a cloned source resource, it will trigger the regeneration of the downstream resources. This is done by listing the downstream resources that have the source UID label set to the UID of the cloned source resource. Then, we update the downstream resources with the new state of the cloned source resource and update it in the metadata cache too.

    - On DELETE event:

        a. The `handleDelete` function checks if the deleted resource is one of the downstream resources it manages. (by checking if it has the label `app.kubernetes.io/managed-by: kyverno`)
        
        b. If it is a downstream resource, then we get the data from the cache, and create it.

        c. If it is a cloned source resource, we will delete the downstream resources that have the source UID label set to the UID of the cloned source resource.

Synchronization on Policy Events:

- When a GeneratingPolicy is created, the WatchManager will register watchers for the GVRs of the generated resources when a trigger resource is created.

- When a GeneratingPolicy is updated:
  - If the policy is updated to disable synchronization, the WatchManager will stop the watchers for the GVRs of the generated resources.
  - If the policy is modified to update the generated resources, we will create an UpdateRequest (UR) to trigger the evaluation of the policy and update the downstream resources.

- When a GeneratingPolicy is deleted, the WatchManager will stop the watchers for the GVRs of the generated resources. This is done by calling `RemoveWatchersForPolicy(name)` which will decrement the reference count for each GVR. If the count reaches zero, the watcher is stopped. It will also delete the downstream resources that have the policy name label set to the name of this policy.

The GeneratingPolicy controller will also be responsible for the generateExisting feature. When a GeneratingPolicy is created with `generateExisting: true`, the controller will list all existing trigger resources and create URs to generate the downstream resources for each trigger.

