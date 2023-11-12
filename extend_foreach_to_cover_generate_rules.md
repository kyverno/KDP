# Support `forEach` in Generate Rules

- **Authors**: [Mariam Fahmy](https://github.com/MariamFahmy98)
- **Created**: Nov 11th, 2023
- **Abstract**: Supporting `forEach` in Generate Rules

# Table of Contents
- [Overview](#overview)
- [Definitions](#definitions)
- [Motivation](#motivation)
- [Proposal](#proposal)
- [Implementation](#implementation)
- [Drawbacks](#drawbacks)
- [Alternatives](#alternatives)
- [Prior Art](#prior-art)
- [Unresolved Questions](#unresolved-questions)

# Overview
[overview]: #overview
Generate rules are in charge of creating new resources. The rule can either define the new resource using the `data` object or use an existing resource in the cluster with the `clone` object. Currently, only one resource can be generated. 

This proposal is to extend the `forEach` to cover generate rules.

# Definitions
[definitions]: #definitions

1. Trigger: It refers to the resource responsible for triggering the generate rule as defined in a combination of `match` and `exclude` blocks.

2. Downstream: It refers to the generated resource(s).

3. Source: It refers to the clone source when generating new resources from existing ones in the cluster.

# Motivation
[motivation]: #motivation

There are some use cases where a single "trigger" resource should result in the creation of multiple downstream resources.

Examples:
1. Generating a list of new network policies based on a comma separated string in an annotation using a `forEach` and `Split()`.
2. Migrating from ingress to Kubernetes Gateway API requires generating at least two resources: `Gateway` and `HTTPRoute`.

# Proposal
In this proposal, Kyverno generate rule can be extended to allowing the generation of multiple resources using `forEach`.
This could be done by using the `list` attribute which is a JMESPath expression that defines the sub-elements it processes.

Examples:
1. Generate multiple network policies when a new namespace is created:
```yaml
spec:
  rules:
  - name: generate-network-policies
    match:
      any:
      - resources:
          kinds:
          - Namespace
    generate:
      synchronize: true
      foreach:
        - list: request.object.metadata.labels.networkpolicies | split(@, ',')
          resources:
            - apiVersion: v1
              kind: NetworkPolicy
              name: '{{ element }}'
              namespace: "{{request.object.metadata.name}}"
              data:
                spec:
                  podSelector: {}
                  policyTypes:
                  - Ingress
                  - Egress
```

2. Generate ConfigMap and Secret for each container:
```yaml
spec:
  rules:
  - name: generate-configmaps-and-secrets
    match:
      any:
      - resources:
          kinds:
          - Pod
    generate:
      synchronize: true
      foreach:
        - list: request.object.spec.containers
          resources:
            - apiVersion: v1
              kind: ConfigMap
              name: cm
              namespace: "{{request.object.metadata.namespace}}"
              data:
                foo: bar
            - apiVersion: v1
              kind: Secret
              name: secret
              namespace: "{{request.object.metadata.namespace}}"
              data:
                foo: bar
```

3. Generate Gateway and HTTPRoute from Ingress:
```yaml
spec:
  rules:
  - name: generate-gatewayapi
    match:
      any:
      - resources:
          kinds:
          - Ingress
    generate:
      synchronize: false
      foreach:
        - list: request.object.spec.rules
          resources:
            - apiVersion: gateway.networking.k8s.io/v1beta1
              kind: HTTPRoute
              # the name must be unique
              name: httproute
              namespace: "{{request.object.metadata.namespace}}"
              data:
                spec:
                  hostnames: 
                    - '{{ element.host }}'
        - list: request.object.spec.ingressClassName
          resources:
            - apiVersion: gateway.networking.k8s.io/v1beta1
              kind: Gateway
              name: gateway
              data:
                spec:
                  gatewayClassName: '{{ element }}'
```

# Implementation

We will extend the generate rule as follows:
```go
type Generation struct {
  ForEachGeneration []ForEachGeneration `json:"foreach,omitempty" yaml:"foreach,omitempty"`
}
```

```go
type ForEachGeneration struct {
  List string `json:"list,omitempty" yaml:"list,omitempty"`

  Resource []Resource `json:"resources,omitempty" yaml:"resources,omitempty"`
}
```

```go
type Resource struct {
  ResourceSpec `json:",omitempty" yaml:",omitempty"`

  RawData *apiextv1.JSON `json:"data,omitempty" yaml:"data,omitempty"`
}
```

- `foreach` will only support generating Kubernetes resources that are defined as a part of the rule.

- Nested `foreach` can be supported but there is no real use case for it.

- At least one of the following: `data`, `clone`, or `foreach` must be declared.

# Drawbacks

N/A

# Alternatives

N/A

# Prior Art

N/A

# Unresolved Questions

N/A
