# Meta
[meta]: #meta
- Name: Enable cache with informers for kubernetes resources
- Start Date: 2022-11-23
- Author(s): shahpratikr, eddycharly
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

- Initialise new informer and create resolver chain for config map lister and kubeclient at [here](https://github.com/kyverno/kyverno/blob/5244730f7aef36961518feb57fb837050562a88d/cmd/kyverno/main.go#L410)
```golang
	cacheInformer, err := resolvers.GetCacheInformerFactory(kubeClient, resyncPeriod)
	if err != nil {
		logger.Error(err, "failed to create cache informer factory")
		os.Exit(1)
	}
	informerBasedResolver, err := resolvers.NewInformerBasedResolver(cacheInformer.Core().V1().ConfigMaps().Lister())
	if err != nil {
		logger.Error(err, "failed to create informer based resolver")
		os.Exit(1)
	}
	clientBasedResolver, err := resolvers.NewClientBasedResolver(kubeClient)
	if err != nil {
		logger.Error(err, "failed to create client based resolver")
		os.Exit(1)
	}
	configMapResolver, err := resolvers.NewResolverChain(informerBasedResolver, clientBasedResolver)
	if err != nil {
		logger.Error(err, "failed to create config map resolver")
		os.Exit(1)
	}
```

- Start `cacheInformer` [here](https://github.com/kyverno/kyverno/blob/5244730f7aef36961518feb57fb837050562a88d/cmd/kyverno/main.go#L462) and [here](https://github.com/kyverno/kyverno/blob/5244730f7aef36961518feb57fb837050562a88d/cmd/kyverno/main.go#L610)

- Pass `configMapResolver` object to webhooks handlers on [line](https://github.com/kyverno/kyverno/blob/5244730f7aef36961518feb57fb837050562a88d/cmd/kyverno/main.go#L572)
```golang
	resourceHandlers := webhooksresource.NewHandlers(
		dClient,
		kyvernoClient,
		configuration,
		metricsConfig,
		policyCache,
		configMapResolver,  <-- new parameter
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

- Create new package `resolvers` inside [context](https://github.com/kyverno/kyverno/tree/5244730f7aef36961518feb57fb837050562a88d/pkg/engine/context) directory which will create an interface `NamespacedResourceResolver` needed to implement resolver chain. This resolver chain will have informer based resolver and Kubernetes client based resolver.

- Pass `configMapResolver` variable to policy builder [here](https://github.com/kyverno/kyverno/blob/5244730f7aef36961518feb57fb837050562a88d/pkg/webhooks/resource/handlers.go#L91)


- Add and initialize `InformerCacheResolvers` at following places
	- [PolicyContext struct](https://github.com/kyverno/kyverno/blob/5244730f7aef36961518feb57fb837050562a88d/pkg/engine/policyContext.go#L44)
	- [PolicyContext Copy](https://github.com/kyverno/kyverno/blob/5244730f7aef36961518feb57fb837050562a88d/pkg/engine/policyContext.go#L57)
	- [PolicyContext builder](https://github.com/kyverno/kyverno/blob/5244730f7aef36961518feb57fb837050562a88d/pkg/webhooks/utils/policy_context_builder.go#L40)
	- [PolicyContext build](https://github.com/kyverno/kyverno/blob/5244730f7aef36961518feb57fb837050562a88d/pkg/webhooks/utils/policy_context_builder.go#L86)

- Get ConfigMap using InformerCacheResolvers [here](https://github.com/kyverno/kyverno/blob/5244730f7aef36961518feb57fb837050562a88d/pkg/engine/jsonContext.go#L354)
```golang
	obj, err := ctx.InformerCacheResolvers.Get(context.TODO(), namespace.(string), name.(string))
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

[Kyverno PR](https://github.com/kyverno/kyverno/pull/5484/)

# Migration (OPTIONAL)

N/A

# Drawbacks

N/A

# Alternatives

N/A

# Prior Art

N/A

# Unresolved Questions

N/A

# CRD Changes (OPTIONAL)

N/A
