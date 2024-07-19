# Support `forEach` in Generate Rules

- **Authors**: [Mariam Fahmy](https://github.com/MariamFahmy98), [Shuting Zhao](https://github.com/realshuting)
- **Created**: Nov 11th, 2023
- **Updated**: Jul 18th, 2024
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
Generate rules are in charge of creating new resources. The rule can either generate the new resource using the `data` object or clone from an existing resource with the `clone` object. Currently, both generate patterns create single target resource. 

The `forEach` declaration is supported in validate and mutate rules to simplify rule applications of sub-elements in resource declarations. This proposal is to extend `forEach` support to offer flexible options for resource generation.

# Definitions
[definitions]: #definitions

1. Trigger: It refers to the resource responsible for triggering the generate rule as defined in a combination of `match` and `exclude` blocks.

2. Downstream/target: It refers to the generated resource(s).

3. Source: It refers to the clone source when cloning new resources from existing ones.

# Motivation
[motivation]: #motivation

There are some use cases where a single "trigger" resource should result in the creation of multiple downstream resources.

Examples:
1. Generate a list of new `NetWorkPolicy` based on a comma separated string in an annotation using a `forEach` and `Split()`.
2. Migrate from ingress to Kubernetes Gateway API requires generating at least two resources: `Gateway` and `HTTPRoute`.
3. Create several `ProjectIamMember` Crossplane GCP CRs based on a comma-separated label of a namespace.
4. Clone the `Secret` to a list of `Namespace` looked up by the source `Secret`'s label.

# Proposal
In this proposal, the generate rule can be expanded to support the creation of multiple resources or the cloning of a source to multiple targets using the `forEach` declaration.

This could be done by using the `list` attribute which is a JMESPath expression that defines the sub-elements it processes.

Examples:
1. Generate multiple `NetWorkPolicy` when a new `Namespace` is created:
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
          dataList:
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

2. Generate `ConfigMap` and `Secret` for each container:
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
          dataList:
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

3. Generate `Gateway` and `HTTPRoute` from Ingress:
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
          dataList:
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
          dataList:
            - apiVersion: gateway.networking.k8s.io/v1beta1
              kind: Gateway
              name: gateway
              data:
                spec:
                  gatewayClassName: '{{ element }}'
```

4. Clone the `Secret` to a list of `Namespace` looked up by the source `Secret`'s label:
```yaml
spec:
  rules:
  - name: clone-seret-to-multiple-namespace
    match:
      any:
      - resources:
          kinds:
          - Secret
          namespaces:
          - default
    context:
    - variable:
        name: selector
        jmesPath: request.object.metadata.labels.sync
        default: a=b
    - name: namespaces
      apiCall:
        urlPath: /api/v1/namespaces?labelSelector={{selector}}?limit=5
    generate:
      synchronize: false
      foreach:
        - list: "{{namespaces}}"
          cloneList:
          - source:
              namespace: default
              name: regcred
              kind: v1
              apiVersion: Secret
            target:
              namespace: "{{element}}"
              name: regcred
```

Note a generate rule allows only one of the following `data`, `clone`, `cloneList` and `foreach` to be declared.

# Implementation
tbd

# Drawbacks

N/A

# Alternatives

N/A

# Prior Art

N/A

# Unresolved Questions

- Support nested `foreach`?
