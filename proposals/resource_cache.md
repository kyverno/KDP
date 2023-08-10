# Meta
[meta]: #meta
- Name: Resource Cache
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

Optional caching for any Kubernetes resource.

# Motivation
[motivation]: #motivation

From: https://github.com/kyverno/kyverno/issues/4459

> In some cases to properly validate a resource we need to examine other resources. Particularly for Ingress and Istio Gateways/VirtualServices we need to look at all the other Ingress/virtualservices or services in the cluster. At large scale we are finding that Kyverno struggles to handle repeatedly pulling thousands of resources using the context apiCall variables. On a cluster with around 4,000 Service objects and 3,000 Ingresses we found that a policy validating Istio VirtualService destinations against Services (ensuring the target exists) was taking > 10s for all measurements (p50, p95 and p99), and the webhook timeout was exceeded. Another policy that validates an Ingress doesn't duplicate a hostname from another Ingress had p95 execution times over 5 seconds. During this time the controllers were at/below requested values for cpu/memory and otherwise had no other indicator of performance problems.

# Proposal

There are two parts to this feature:
1. Allow users to manage which resources should be cached
2. Allow policy rules to reference cached resource data

Users can manage which resources to cache using the same mechanism that is currently used for ConfigMap resources i.e. adding a label `cache.kyverno.io/enabled: "true"` to the resource.

In the Kyverno policy a new `context` entry `resourceCache` will be added. Here is an example:

```yaml
context:
  - name: ingresses
    resourceCache:
      group: "apis/networking.k8s.io"
      version: "v1"
      kind: "ingresses"
      jmesPath: "ingresses | items[].spec.rules[].host"
```

This allows policy authors to declare what resource should be cached. 

The `group` and `version` are optional. If not specified, the preferred versions should be used.

An optional `namespace` can be used to only cache resources in the namespace, rather than across all namespaces which is the behavior is a namespace is not specified.

The JMESPath is optional and is applied to add a resulting subset of the resource data to the rule context.

Note that Kyverno will only cache matching resources that have the label: `cache.kyverno.io/enabled: "true"`.

# Implementation

When policies are created or modified, Kyverno will attempt to initialize informers for any `resourceCache` declaration. In case an informer cannot be initialized, an error will be returned.

During rule execution, Kyverno will add the resource data to the rule context.

## Link to the Implementation PR

TBD

# Migration (OPTIONAL)

There is no automated migration. 

However, to leverage resource caching, users can convert API calls to the new `resourceCache` declaration.

Here are some API calls from sample policies along with the correspinding `resourceCache` declarations:

https://kyverno.io/policies/other/e-l/ensure-production-matches-staging/ensure-production-matches-staging/

```
    context:
    - name: deployment_count
      apiCall:
        urlPath: "/apis/apps/v1/namespaces/staging/deployments"
        jmesPath: "items[?metadata.name=='{{ request.object.metadata.name }}'] || `[]` | length(@)"
```

can be converted to:

```
    context:
    - name: deployment_count
      resourceCache:
        group: "apps"
        version: "v1"
        kind: "deployments"
        namespace: "staging"
        jmesPath: "items[?metadata.name=='{{ request.object.metadata.name }}'] || `[]` | length(@)"
```

https://kyverno.io/policies/linkerd/require-linkerd-server/require-linkerd-server/

```
    context:
    - name: server_count
      apiCall:
        urlPath: "/apis/policy.linkerd.io/v1beta1/namespaces/{{request.namespace}}/servers"
```

can be converted to:

```
    context:
    - name: server_count
      resourceCache:
        group: "policy.linkerd.io"
        version: "v1beta1"
        kind: "servers"
        namespace: "{{request.namespace}}"
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

can be converted to:

```
    context:
    - name: servers
      resourceCache:
        group: "policy.linkerd.io"
        version: "v1beta1"
        kind: "servers"
        namespace: "{{request.namespace}}"
        jmesPath: "items[].metadata.name || `[]"
```


https://kyverno.io/policies/istio/require-authorizationpolicy/require-authorizationpolicy/

```
    - name: allauthorizationpolicies
      apiCall:
        urlPath: "/apis/security.istio.io/v1beta1/authorizationpolicies"
        jmesPath: "items[].metadata.namespace"

```

can be converted to:

```
    context:
    - name: allauthorizationpolicies
      resourceCache:
        group: "security.istio.io"
        version: "v1beta1"
        kind: "authorizationpolicies"
        jmesPath: "items[].metadata.namespace"
```


https://kyverno.io/policies/other/rec-req/require-netpol/require-netpol/


```
    - name: policies_count
      apiCall:
        urlPath: "/apis/networking.k8s.io/v1/namespaces/{{request.namespace}}/networkpolicies"
        jmesPath: "items[?label_match(spec.podSelector.matchLabels, `{{request.object.spec.template.metadata.labels}}`)] | length(@)"
```

can be converted to:

```
    context:
    - name: policies_count
      resourceCache:
        group: "networking.k8s.io"
        version: "v1"
        kind: "networkpolicies"
        namespace: "{{request.namespace}}"
        jmesPath: "items[?label_match(spec.podSelector.matchLabels, `{{request.object.spec.template.metadata.labels}}`)] | length(@)"
```

# Drawbacks

API calls do not leverage caching by default.

If needed, we can add a separate caching mechanism for API calls in the future.


# Alternatives

## Add caching to API calls 

To cache resources, the `apiCall` context entry can be extended to add a `cache` field:

```yaml
      context:
        - name: hosts
          apiCall:
            urlPath: "/apis/networking.k8s.io/v1/ingresses"
            jmesPath: "items[].spec.rules[].host"
            cache: true
```

However, coupling resource caching to API calls could be confusing for users.

# Prior Art

N/A

# Limitations

1. This only addresses Kubernetes resource caching. Resources from other API calls are not cached.

# Open Items

1. We may not be able to use a static Kubernetes client for all types, as the client set can include custom types, and dynamic clients may be resource intensive. More research is needed to determine the best way to manage informers.
2. Typically informers are initialized on startup. This feature may require adding / deleting informers after startup.
3. All admission controller replicas will need to cache data. For background controller and reports controller the leader will need to cache data.

# CRD Changes (OPTIONAL)

Yes. A new context entry `resourceCache` will be added. The CRD will be defined in the implementation stage.

