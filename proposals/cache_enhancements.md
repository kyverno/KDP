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

- Initialise new filtered informer at [here](https://github.com/kyverno/kyverno/blob/925f0cf182c74fbf23f5d974cafeb0f05f7292bf/cmd/kyverno/main.go#L473)
```bash
// this label need to be hard-coded as at this point we do not have any resource to get this value from
labelsMap := map[string]string{"cache.kyverno.io/enabled": "true"}
labelSelector := labels.Set(labelsMap).AsSelector().String()
kubeResourceInformer := informers.NewFilteredSharedInformerFactory(
    kubernetesClientSet, 10*time.Minute, "", func(l0 *metav1.ListOptions) {
        // setting kind here is optional, if we do not provide it, all resources which have labels will be cached
        l0.Kind = "ConfigMap"
        l0.LabelSelector = labelSelector
    })
```

- Start `kubeResourceInformer` [here](https://github.com/kyverno/kyverno/blob/925f0cf182c74fbf23f5d974cafeb0f05f7292bf/cmd/kyverno/main.go#L527)

- Create new directory `informerCache` inside [this](https://github.com/kyverno/kyverno/tree/925f0cf182c74fbf23f5d974cafeb0f05f7292bf/pkg) directory

- Create new file `cache.go` inside `informerCache` directory with following contents
```bash
package informerCache

import (
	"errors"

	"k8s.io/client-go/informers"
)

type ConfigmapResolver interface {
	Get(string, string, logr.Logger) (map[string]interface{}, error)
}

type InformerBasedResolver struct {
	CMLister corev1listers.ConfigMapLister
}

type ClientBasedResolver struct {
	DynamicClient dclient.Interface
}

type ResolverChain []ConfigmapResolver

func NewResolver(kubeResourceInformer informers.SharedInformerFactory, dynamicClient dclient.Interface) ResolverChain {
	resolver := []ConfigmapResolver{
		&InformerBasedResolver{CMLister: kubeResourceInformer.Core().V1().ConfigMaps().Lister()},
		&ClientBasedResolver{DynamicClient: dynamicClient}}
	return resolver
}

func (i *InformerBasedResolver) Get(namespace, name string, logger logr.Logger) (map[string]interface{}, error) {
	// try to lookup from the cache
	return nil, nil
}

func (c *ClientBasedResolver) Get(namespace, name string, logger logr.Logger) (map[string]interface{}, error) {
	// try to lookup from the client
	return nil, nil
}

func (r ResolverChain) Get(namespace, name string, logger logr.Logger) (map[string]interface{}, error) {
	for _, resolver := range r {
		if cm, err := resolver.Get(namespace, name, logger); err == nil {
			return cm, nil
		}
	}
	return nil, errors.New("not found")
}
```

- Initialise interface and pass object to webhooks handlers on [line](https://github.com/kyverno/kyverno/blob/925f0cf182c74fbf23f5d974cafeb0f05f7292bf/cmd/kyverno/main.go#L630)
```bash
informerCacheResolver := informerCache.NewResolver(kubeResourceInformer, dClient)
resourceHandlers := webhooksresource.NewHandlers(
    dClient,
    kyvernoClient,
    configuration,
    metricsConfig,
    policyCache,
    informerCacheResolver,  <-- added variable
    kubeInformer.Core().V1().Namespaces().Lister(),
    kubeInformer.Rbac().V1().RoleBindings().Lister(),
    kubeInformer.Rbac().V1().ClusterRoleBindings().Lister(),
    kyvernoInformer.Kyverno().V1beta1().UpdateRequests().Lister().UpdateRequests(config.KyvernoNamespace()),
    urgen,
    eventGenerator,
    openApiManager,
    admissionReports,
)
```

- add `informerCacheResolver` variable in [handlers struct](https://github.com/kyverno/kyverno/blob/925f0cf182c74fbf23f5d974cafeb0f05f7292bf/pkg/webhooks/resource/handlers.go#L47)

- add `informerCacheResolver` in policyContext at following places.
    - [validate](https://github.com/kyverno/kyverno/blob/925f0cf182c74fbf23f5d974cafeb0f05f7292bf/pkg/webhooks/resource/handlers.go#L127)
    - [mutate](https://github.com/kyverno/kyverno/blob/925f0cf182c74fbf23f5d974cafeb0f05f7292bf/pkg/webhooks/resource/handlers.go#L169) (need to verify if we require to set here as we are calling build again on line 181)
    - [mutate](https://github.com/kyverno/kyverno/blob/925f0cf182c74fbf23f5d974cafeb0f05f7292bf/pkg/webhooks/resource/handlers.go#L186)
    - [audit](https://github.com/kyverno/kyverno/blob/925f0cf182c74fbf23f5d974cafeb0f05f7292bf/pkg/webhooks/resource/validation/validation.go#L141) (need to verify)
    
    NOTE: we are adding `informerCacheResolver` in policyContext as only policyContext is passed to some functions in image verify process. If we add this as a separate parameter, we might have to make lot of changes.

- add `informerCacheResolver` [here](https://github.com/kyverno/kyverno/blob/925f0cf182c74fbf23f5d974cafeb0f05f7292bf/pkg/engine/policyContext.go#L44)

- get ConfigMap from informerCacheResolver [here](https://github.com/kyverno/kyverno/blob/925f0cf182c74fbf23f5d974cafeb0f05f7292bf/pkg/engine/jsonContext.go#L353)
```bash
    obj, err := ctx.informerCacheResolver.Get()
    if err != nil {
        return nil, fmt.Errorf("failed to get configmap %s/%s : %v", namespace, name, err)
    }
```

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
