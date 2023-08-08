# Meta
[meta]: #meta
- Name: Resourc Cache
- Start Date: Aug 7, 2023
- Update data (optional): Aug 7, 2023
- Author(s): @JimBugwadia

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

Optional caching of any Kubernetes resource.


# Motivation
[motivation]: #motivation

From: https://github.com/kyverno/kyverno/issues/4459

> In some cases to properly validate a resource we need to examine other resources. Particularly for Ingress and Istio Gateways/VirtualServices we need to look at all the other Ingress/virtualservices or services in the cluster. At large scale we are finding that Kyverno struggles to handle repeatedly pulling thousands of resources using the context apiCall variables. On a cluster with around 4,000 Service objects and 3,000 Ingresses we found that a policy validating Istio VirtualService destinations against Services (ensuring the target exists) was taking > 10s for all measurements (p50, p95 and p99), and the webhook timeout was exceeded. Another policy that validates an Ingress doesn't duplicate a hostname from another Ingress had p95 execution times over 5 seconds. During this time the controllers were at/below requested values for cpu/memory and otherwise had no other indicator of performance problems.

# Proposal

There are two aspects to this feature:
1. Allow users to manage which resources should be cached
2. Allow policy rules to reference cached resources

Users can manage which resources to cache using the same mechanism that is currently used for ConfigMap resources i.e. adding a label `cache.kyverno.io/enabled: "true"` to the resource.

To reference cached resources, the `apiCall` context entry can be used:

```yaml
      context:
        - name: hosts
          apiCall:
            urlPath: "/apis/networking.k8s.io/v1/ingresses"
            jmesPath: "items[].spec.rules[].host"
            cache: true
```

# Implementation

When policies are created or modified, Kyverno will attempt to initialize informers for any resource type when `cache: true` is specified in the `apiCall`. In case an informer cannot be initialized, or the `urlPath` cannot be converted to an cache lookup (see [Converting API Calls](#converting-api-calls), an error will be returned.

During rule execution, Kyverno will again convert the API call to a cache lookup and add the matching resources to the rule context.

## Converting API Calls

Kyverno will attempt to convert API calls to the following resource information:
* Group
* Version
* Kind
* Namespace (optional)
* Name (optional)

If the API call has parameters, or other complexities that prevent conversion, the conversion will fail and return an error.

Kyverno will then load one or more instances of the resources into the policy rule context.

Here are some other API calls from sample policies:

https://kyverno.io/policies/other/e-l/ensure-production-matches-staging/ensure-production-matches-staging/

```
    context:
    - name: deployment_count
      apiCall:
        urlPath: "/apis/apps/v1/namespaces/staging/deployments"
        jmesPath: "items[?metadata.name=='{{ request.object.metadata.name }}'] || `[]` | length(@)"
```

https://kyverno.io/policies/linkerd/require-linkerd-server/require-linkerd-server/

```
    context:
    - name: server_count
      apiCall:
        urlPath: "/apis/policy.linkerd.io/v1beta1/namespaces/{{request.namespace}}/servers"
```

https://kyverno.io/policies/linkerd/check-linkerd-authorizationpolicy/check-linkerd-authorizationpolicy/


```
    context:
    - name: servers
      apiCall:
        urlPath: "/apis/policy.linkerd.io/v1beta1/namespaces/{{request.namespace}}/servers"
        jmesPath: "items[].metadata.name || `[]`"
    - name: httproutes
      apiCall:
        urlPath: "/apis/policy.linkerd.io/v1beta1/namespaces/{{request.namespace}}/httproutes"
        jmesPath: "items[].metadata.name || `[]`"
```

https://kyverno.io/policies/istio/require-authorizationpolicy/require-authorizationpolicy/

```
    - name: allauthorizationpolicies
      apiCall:
        urlPath: "/apis/security.istio.io/v1beta1/authorizationpolicies"
        jmesPath: "items[].metadata.namespace"

```

https://kyverno.io/policies/other/rec-req/require-netpol/require-netpol/


```
    - name: policies_count
      apiCall:
        urlPath: "/apis/networking.k8s.io/v1/namespaces/{{request.namespace}}/networkpolicies"
        jmesPath: "items[?label_match(spec.podSelector.matchLabels, `{{request.object.spec.template.metadata.labels}}`)] | length(@)"
```

## Link to the Implementation PR

TBD

# Migration (OPTIONAL)

N/A

# Drawbacks

It may be confusing that we are using APICalls to specify resource caching.

# Alternatives

## Introduce a new context entry for cached resources

An alternative scheme would be to define a new context entry type:

```yaml
      context:
        - name: hosts
          resourceCache:
            group: "apis/networking.k8s.io"
            version: "v1"
            kind: "ingresses"
            jmesPath: "items[].spec.rules[].host"
```

# Prior Art

N/A

# Limitations

1. The inital version will only address Kubernetes API server calls. Other service API calls can be considered in future versions.
2. 

# CRD Changes (OPTIONAL)

Yes. A new field `cache` will be added to the `apiCall`

