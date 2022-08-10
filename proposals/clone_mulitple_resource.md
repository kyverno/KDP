# Meta
[meta]: #meta
- Name: Allow cloning multiple resources from a Namespace
- Start Date: 2022-08-04
- Update data (optional): 2022-08-04
- Author: [Prateek Pandey](https://github.com/prateekpandey14)
- Supersedes:  N/A

# Table of Contents
[table-of-contents]: #table-of-contents
- [Meta](#meta)
- [Table of Contents](#table-of-contents)
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

Enhance kyverno generate rule to clone mulitple resources from a given namespace.


# Definitions
[definitions]: #definitions

N/A

# Motivation
[motivation]: #motivation

The initial idea for this feature came from issue [282](https://github.com/kyverno/kyverno/issues/282) and received feedback from the kyverno community in slack.

It would be good to have a way to copy/clone multiple resources from a namespace in a single rule, instead of
writing a new rule or policy.

There are primarily main use cases associated with this.

1. To clone set of resources from a common namespace, using a match selector whenever we create a new namespace
   to bootstrap the deployment process.

2. Implicit, User has to less number of policy/rule to copy the set of necessary resources.

# Proposal

In this proposal, Kyverno generate rule can be extended to allowing to clone multiple resources from a namespace.
This could be done using  matching selector to select multiple resources of the given apiVersion/group and allowing
multiple kinds/name pairs, similar to the match/exclude blocks we use to filter resources in the admission control
webhooks.

In below extended ClusterPolicy example, multiple resources can be selected using a match label selector `rollout: true`
to copy/clone from `bootstrap` namespace to newly created namespace.

```yaml
    generate:
      namespace: "{{request.object.metadata.name}}"
      synchronize : true
      clone:
        namespace: bootstrap    // common namespace
        kinds:
          - v1/Secret
        selector:
          matchLabels:
            rollout: "true"    // clone resource with matching selector i.e. configmap, secrets etc
```

### Examples:

1. Clone multiple resource using matching kinds list

```yaml

apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: clone-resource-with-selector
spec:
  generateExistingOnPolicyUpdate: true
  rules:
  - name: clone-resource
    match:
      any:
      - resources:
          kinds:
          - Namespace
    generate:
      namespace: "{{request.object.metadata.name}}"
      synchronize : true
      cloneList:
        - namespace:
            - stagging
          kinds:
            - v1/Secret   // clone resource with matching selector i.e. configmap, secrets etc
```


# Implementation

- As part of implementation changes,firstly we have to extend the existing APIs, the `generate.clone` rule
will have a selector option to provide the matching labels as key:value pair.

```go
// Generation defines how new resources should be created and managed.
type Generation struct {
.
.
.
// Clone specifies the source resource used to populate each generated resource.
// +optional
Clone CloneFrom `json:"clone,omitempty" yaml:"clone,omitempty"`
}

type CloneFrom struct {
// Namespace specifies source resource namespace.
// +optional
Namespace string `json:"namespace,omitempty" yaml:"namespace,omitempty"`

// Name specifies name of the resource.
Name string `json:"name,omitempty" yaml:"name,omitempty"`

// Selector is a label selector. Label keys and values in `matchLabels` support the wildcard
// characters `*` (matches zero or many characters) and `?` (matches one character).
// Wildcards allows writing label selectors like ["storage.k8s.io/*": "*"]. Note that
// using ["*" : "*"] matches any key and value but does not match an empty label set.
// +optional
Selector *metav1.LabelSelector `json:"selector,omitempty" yaml:"selector,omitempty"`
}
```

- Generate controller will filter out the resources based on labels selectors and wildcards, similar to
  match and exclude block.

- Once the resources have been cloned, we need to make sure the reference resource and newly generated 
  resources are always synced.

## Link to the Implementation PR

N/A

# Migration (OPTIONAL)

N/A

# Drawbacks

* Makes Kyverno code base more complex
* Places additional strain on the Kubernetes API server and Kyverno itself as some rule processing may need to list and filter so many resources in a namespace.

# Alternatives

- Clone multiple resource using list of kind/namespace pairs in `spec.Clone`
- which is not easy to extend/manage new or existing policies due to backward compatibility.
- If we support cloning specific resources from a namespace, we end up writing
  more then one policy/rule to clone multiple resources.

# Prior Art

N/A

# Unresolved Questions

N/A

# CRD Changes (OPTIONAL)

This KDP will require introduction of a new field type under `generate.clone` rule named as `selector`.
