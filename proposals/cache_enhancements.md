# Meta
[meta]: #meta
- Name: Enable cache with informers for kubernetes resources
- Start Date: 2022-11-23
- Author(s): shahpratikr
- Supersedes: N/A

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

This proposal is to add cache with informers for Kubernetes resources specifically ConfigMap at this point. Later, similar approaches can be used for KMS, Image verify results, etc.

# Definitions
[definitions]: #definitions
CM: ConfigMap

# Motivation
[motivation]: #motivation

When CM is used as a context variable or while mutating variables, Kyverno needs to call API server each time policy executes. At large scale where we are processing 5-6 admission requests per second this puts additional load on the cluster API and will increase the latency of admission processing. To reduce this load, we can use separate informer or in-memory cache.

# Proposal

- We create separate filtered informer which listens for CM kind and label selector.
```bash
labelsMap := map[string]string{"cache-enabled": "true"}
labelSelector := labels.Set(labelsMap).AsSelector().String()
informerFactory := informers.NewFilteredSharedInformerFactory(
    kubernetesClientSet, 10*time.Minute, "", func(l0 *metav1.ListOptions) {
        l0.Kind = "ConfigMap"
        l0.LabelSelector = labelSelector
    })
```
- If we get CMs with expected labels, we store the names of those CMs in in-memory cache list.
- While mutating values of CM, we can refer this in-memory cache list. If the CM name is present in in-memory list, we fetch details from InformerFactory. Else, we fetch details from KubeClient.

# Implementation

N/A

## Link to the Implementation PR

N/A

# Migration (OPTIONAL)

N/A

# Drawbacks

N/A

# Alternatives

N/A

# Prior Art

N/A

# Unresolved Questions

- Do we want to always use cache when CMs are declared in context variable?
- Do we need a separate way to use cached data in a policy rule directly?

# CRD Changes (OPTIONAL)

N/A
