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

In below extended ClusterPolicy example, multiple resources can be selected using a kinds list to copy/clone from 
`stagging` namespace to newly created namespace.

```yaml
    generate:
      namespace: "{{request.object.metadata.name}}"
      synchronize : true
      cloneList:
      - namespace: stagging
        kinds:
          - v1/Secret   // clone resource with matching selector i.e. configmap, secrets etc
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
      - namespace: stagging
        kinds:
          - v1/Secret   // clone resource with matching selector i.e. configmap, secrets etc
```


# Implementation

- As part of implementation changes,firstly we have to extend the existing APIs, the `generate.cloneList` rule
will have a `kinds` and `namespace` field to provide the list of resources to be cloned.

```go
// Generation defines how new resources should be created and managed.
type Generation struct {
.
.
.
// CloneList specifies the list of source resource used to populate each generated resource.
// +optional
CloneList CloneList `json:"cloneList,omitempty" yaml:"cloneList,omitempty"`
}

type CloneList struct {
// Namespace specifies source resource namespace.
// +optional
Namespace string `json:"namespace,omitempty" yaml:"namespace,omitempty"`

// Kinds is a list of resource kinds.
// +optional
Kinds []string `json:"kinds,omitempty" yaml:"kinds,omitempty"`
}
```

- Generate controller will filter out the resources based on kinds list, similar to
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

- Clone multiple resource using list of kind/namespace pairs in `spec.clone`
- which is not easy to extend/manage new or existing policies due to backward compatibility.
- If we support cloning specific resources from a namespace, we end up writing
  more then one policy/rule to clone multiple resources.

# Prior Art

N/A

# Unresolved Questions

N/A

# CRD Changes (OPTIONAL)

This KDP will require introduction of two new fields under `generate.cloneList` rule.
