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

- Initialise new filtered informer at [here](https://github.com/kyverno/kyverno/blob/main/cmd/kyverno/main.go#L456)
// this label need to be hard-coded as at this point we don't have any resource to get this value from
```bash
labelsMap := map[string]string{"cache.kyverno.io/enabled": "true"}
labelSelector := labels.Set(labelsMap).AsSelector().String()
kubeResourceInformer := informers.NewFilteredSharedInformerFactory(
    kubernetesClientSet, 10*time.Minute, "", func(l0 *metav1.ListOptions) {
        // setting kind here is optional, if we don't provide it, all resources which have labels will be cached
        l0.Kind = "ConfigMap"
        l0.LabelSelector = labelSelector
    })
```
- pass this `kubeResourceInformer` to webhooks handlers on [line](https://github.com/kyverno/kyverno/blob/main/cmd/kyverno/main.go#L617)
- add `kubeResourceInformer` variable in [handlers struct](https://github.com/kyverno/kyverno/blob/main/pkg/webhooks/resource/handlers.go#L53)
- initialise `kubeResourceInformer` [here](https://github.com/kyverno/kyverno/blob/main/pkg/webhooks/resource/handlers.go#L88)
- add kubeResourceInformer in policyContext at following places.
    - https://github.com/kyverno/kyverno/blob/main/pkg/webhooks/resource/handlers.go#L127
    - https://github.com/kyverno/kyverno/blob/main/pkg/webhooks/resource/handlers.go#L169 (need to verify if we require to set here as we are calling build again on line 181)
    - https://github.com/kyverno/kyverno/blob/main/pkg/webhooks/resource/handlers.go#L185
    - https://github.com/kyverno/kyverno/blob/main/pkg/webhooks/resource/validation/validation.go#L140 (need to verify)
    NOTE: we are adding `kubeResourceInformer` in policyContext so that in future we won't need to create lister for each resources again.
- add `KubeResourceInformer` [here](https://github.com/kyverno/kyverno/blob/main/pkg/engine/policyContext.go#L44)
- get ConfigMap from informer cache [here](https://github.com/kyverno/kyverno/blob/main/pkg/engine/jsonContext.go#L353)
```bash
    informerCache := &InformerCache{}
    obj, err := informerCache.GetK8sResourceFromCache(ctx, "ConfigMap", name.(string), namespace.(string))
    if err == nil {
        // use object from informer
    } else if strings.Contains(err.Error(), "not found") {
        obj, err := ctx.Client.GetResource("v1", "ConfigMap", namespace.(string), name.(string))
        if err != nil {
            return nil, fmt.Errorf("failed to get configmap %s/%s : %v", namespace, name, err)
        }
    } else {
        return nil, fmt.Errorf("failed to get configmap %s/%s : %v", namespace, name, err)
    }
```
create following interface in new file https://github.com/kyverno/kyverno/blob/main/pkg/engine/k8sResourceCache.go
```bash
type InformerCache interface {
    GetK8sResourceFromCache(policyContext *engine.PolicyContext, resourceKind, resourceName, resourceNamespace string) (*unstructured.Unstructured, error)
}
```

// Create a function
func GetK8sResourceFromCache(ctx, kind, name, namespace) (*unstructured.Unstructured, error) {
    // get the lister for kind using switch case ?
    // get the object from cache
    // if object not found, return (nil, err)
    // if object found, convert it to *unstructure.Unstructured
    // obj, err := runtime.DefaultUnstructuredConverter.ToUnstructured(resourceObj)
    // return obj and err
}

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
