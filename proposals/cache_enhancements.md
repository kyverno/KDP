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
```golang
// this label need to be hard-coded as at this point we don't have any resource to get this value from
labelsMap := map[string]string{"cache.kyverno.io/enabled": "true"}
labelSelector := labels.Set(labelsMap).AsSelector().String()
kubeResourceInformer := informers.NewFilteredSharedInformerFactory(
    kubernetesClientSet, 10*time.Minute, "", func(l0 *metav1.ListOptions) {
        // setting kind here is optional, if we don't provide it, all resources which have labels will be cached
        l0.Kind = "ConfigMap"
        l0.LabelSelector = labelSelector
    })
```

- Start `kubeResourceInformer` [here](https://github.com/kyverno/kyverno/blob/925f0cf182c74fbf23f5d974cafeb0f05f7292bf/cmd/kyverno/main.go#L527)

- Create new directory `informerCache` inside [this](https://github.com/kyverno/kyverno/tree/925f0cf182c74fbf23f5d974cafeb0f05f7292bf/pkg) directory

- Create new file `cache.go` inside `informerCache` directory with following contents
```golang
package informerCache

import (
	"context"

	corev1 "k8s.io/api/core/v1"
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
	"k8s.io/client-go/kubernetes"
	corev1listers "k8s.io/client-go/listers/core/v1"
)

type ConfigmapResolver interface {
	Get(context.Context, string, string) (*corev1.ConfigMap, error)
}

type informerBasedResolver struct {
	lister corev1listers.ConfigMapLister
}

func NewInformerBasedResolver(lister corev1listers.ConfigMapLister) ConfigmapResolver {
	return &informerBasedResolver{
		lister: lister,
	}
}

func (i *informerBasedResolver) Get(bgContext context.Context, namespace, name string) (*corev1.ConfigMap, error) {
	return i.lister.ConfigMaps(namespace).Get(name)
}

type clientBasedResolver struct {
	kubeClient kubernetes.Interface
}

func NewClientBasedResolver(kubeClient kubernetes.Interface) ConfigmapResolver {
	return &clientBasedResolver{
		kubeClient: kubeClient,
	}
}

func (c *clientBasedResolver) Get(bgContext context.Context, namespace, name string) (*corev1.ConfigMap, error) {
	return c.kubeClient.CoreV1().ConfigMaps(namespace).Get(bgContext, name, metav1.GetOptions{})
}

type resolverChain []ConfigmapResolver

func NewResolverChain(resolver ...ConfigmapResolver) ConfigmapResolver {
	return resolverChain(resolver)
}

func (r resolverChain) Get(bgContext context.Context, namespace, name string) (*corev1.ConfigMap, error) {
	var (
		cm  *corev1.ConfigMap
		err error
	)
	for _, resolver := range r {
		cm, err = resolver.Get(bgContext, namespace, name)
		if err == nil {
			return cm, nil
		}
	}
	return nil, err
}
```

- Initialise interface and pass object to webhooks handlers on [line](https://github.com/kyverno/kyverno/blob/925f0cf182c74fbf23f5d974cafeb0f05f7292bf/cmd/kyverno/main.go#L630)
```golang
cacheResolvers := informerCache.NewResolver(
    informerCache.NewInformerBasedResolver(kubeResourceInformer.Core().V1().ConfigMaps().Lister(), logger),
    informerCache.NewClientBasedResolver(kubeClient, logger),
)
resourceHandlers := webhooksresource.NewHandlers(
    dClient,
    kyvernoClient,
    configuration,
    metricsConfig,
    policyCache,
    cacheResolvers,  <-- added variable
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
    - [mutate](https://github.com/kyverno/kyverno/blob/925f0cf182c74fbf23f5d974cafeb0f05f7292bf/pkg/webhooks/resource/handlers.go#L169)
    - [mutate](https://github.com/kyverno/kyverno/blob/925f0cf182c74fbf23f5d974cafeb0f05f7292bf/pkg/webhooks/resource/handlers.go#L186)
    
    NOTE: we are adding `informerCacheResolver` in policyContext as only policyContext is passed to some functions in image verify process. If we add this as a separate parameter, we might have to make lot of changes.

- add `InformerCacheResolver` [here](https://github.com/kyverno/kyverno/blob/925f0cf182c74fbf23f5d974cafeb0f05f7292bf/pkg/engine/policyContext.go#L44)

- get ConfigMap from InformerCacheResolver [here](https://github.com/kyverno/kyverno/blob/925f0cf182c74fbf23f5d974cafeb0f05f7292bf/pkg/engine/jsonContext.go#L353)
```golang
	obj, err := ctx.InformerCacheResolver.Get(context.Background(), namespace.(string), name.(string))
	if err != nil {
		return nil, fmt.Errorf("failed to get configmap %s/%s : %v", namespace, name, err)
	}

	// extract configmap data
	contextData["data"] = obj.Data
	contextData["metadata"] = obj.ObjectMeta
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
