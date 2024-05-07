# Generate ValidatingAdmissionPolicies

- **Authors**: [Mariam Fahmy](https://github.com/MariamFahmy98)
- **Created**: July 18th, 2023
- **Updated**: May 13th, 2024
- **Abstract**: Generate ValidatingAdmissionPolicies from Kyverno `validate.cel` subrule.

## Overview

- [Introduction](#introduction)
- [Motivation](#motivation)
- [Proposal](#proposal)
   + [Kyverno engine VS API server](#kyverno-engine-vs-api-server)
   + [Limitations](#limitations)
- [Implementation](#implementation)
   + [ClusterPolicy vs Policy](#clusterpolicy-vs-policy)
   + [Kyverno Policy Settings](#kyverno-policy-settings)
   + [Match/Exclude Resources](#matchexclude-resources)
   + [Validate Rules](#validate-rules)
   + [Audit Annotations](#audit-annotations)
   + [Parameter Resources](#parameter-resources)
   + [Limitations](#limitations-1)
- [Next Steps](#next-steps)

## Introduction

Kyverno policies support CEL expressions for resources validation. The goal is to generate Kubernetes ValidatingAdmissionPolicies and validating admission binding from the information provided by Kyverno policies.

- In case a new Kyverno policy is created, a ValidatingAdmissionPolicy and its binding are generated. 
- In case Kyverno policy is updated, the corresponding VAP and its binding needs to be updated accordingly. 
- In case Kyverno policy is deleted, the corresponding VAP and its binding are deleted.

Once ValidatingAdmissionPolicy and its binding are generated, the API server will handle the resource validation. 

An example of Kyverno policy from which ValidatingAdmissionPolicy and its binding will be generated:

```yaml=
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: check-deployment-replicas
spec:
  validationFailureAction: Enforce
  background: false
  rules:
    - name: deployment-replicas
      match:
        any:
        - resources:
            kinds:
              - Deployment
      validate:
        cel:
          expressions:
            - expression: "object.spec.replicas < 5"
              message:  "Deployment spec.replicas must be less than 5”
```

## Motivation

Currently, Kyverno engine supports CEL expressions for resource validation. However, ValidatingAdmissionPolicy is an alternative solution to validating admission webhooks. Therefore, it's better to make Kyverno generate ValidatingAdmissionPolicies so that resource validation will be handled by the API server instead. Moreover, the resource validation aganist policies won't be affected in case Kyverno isn't available for some reason.

Users can either let Kyverno engine to validate resources using CEL expressions or let the API server to validate resources and this is done by allowing Kyverno to generate ValidatingAdmissionPolicies and their bindings from a Kyverno policy containing a CEL subrule.

## Proposal

- A new controller `validatingadmissionpolicy-controller` will be created. It will watch the following:
  1. Kyverno cluster policies
  2. ValidatingAdmissionPolicies
  3. ValidatingAdmissionPolicy bindings

- When a policy event happens, the policy key is enqueued. The controller will create/update/delete ValidatingAdmissionPolicies and their bindings accordingly.

- When a ValidatingAdmissionPolicy or ValidatingAdmissionPolicy binding event happens, the policy key is deduced from owner references and it's enqueued. The controller will reset the changes done in either VAP or VAP binding.

- A flag/toggle `--generateValidatingAdmissionPolicy` will be introduced in the controller so that user can either let Kyverno engine handles CEL expressions or let api server handles it by generating VAPs and their bindings. By default, it is set to false.

### Kyverno engine VS API server

Since CEL expressions can be handled by either Kyverno engine or the API server, Here are the cases where Kyverno engine will handle the resource validations itself:
1. The flag is disabled in the `validatingadmissionpolicy-controller`. This is done by setting `--generateValidatingAdmissionPolicy` to false.
2. The flag is enabled and ValidatingAdmissionPolicies were generated but issues are encountered in processing these policies. This is done by getting the metrics exposed by the API server.

To avoid Kyverno engine from processing CEL expressions after the generation of ValidatingAdmissionPolicies, a new field `validatingAdmissionPolicy` will be introduced in the policy status to indicate whether a ValidatingAdmissionPolicy is generated or not.

### Limitations

ValidatingAdmissionPolicy and its bindings will be generated for a Kyverno cluster policy that has one `validate.CEL` subrule.

## Implementation

In this section, we will discuss how ValidatingAdmissionPolicy and its binding can be created from Kyverno policies.

### ClusterPolicy vs Policy

All Kyverno policies of kind `ClusterPolicy` can be converted into ValidatingAdmissionPolicies but not all policies of kind `Policy` can be converted into ValidatingAdmissionPolicies since they are cluster-scoped resources.

Kyverno policies of kind `Policy` can be converted into ValidatingAdmissionPolicy in case it contains namespace selectors as follows:

```
apiVersion: kyverno.io/v1
kind: Policy
metadata:
  name: pol-1
  namespaceSelector: 
    matchExpressions:
      - key: environment
        operator: In
        values:
          - prod
```

It will be converted into validating admission policy binding as follows:

```
spec:
  matchResources: 
    namespaceSelector: 
      matchExpressions:
        - key: environment
          operator: In
          values:
            - prod
```

However, namespaced kyverno policies should define a specific namespace name and since validating admission policy binding doesn't match namespaces by name we still can't generate validating admission policy and its binding from it. 
There's a future plan in the [KEP](3. Provide a per-policy control whether Kyverno engine will handle CEL itself or let API server handles it.) to support namespaced policy binding, therefore namespaced kyverno policies might be used to generate VAPs as well.

### Kyverno Policy Settings
In Kyverno policies, some common settings can be applied to all rules. We will discuss the effect of setting these values on generating ValidatingAdmissionPolicies assuming that Kyverno policy contains multiple rules, one of them is `validate.CEL` subrule.

1. `applyRules`: It must be unset since generating ValidatingAdmissionPolicies is limited to writing a Kyverno policy with one `validate.cel` rule.

2. `background`: Background-scan controller already watches for ValidatingAdmissionPolicies, apply them on the matching resources and generate reports accordingly. This is the default behavior.
    - If it's set to `true` (default), reports are generated for ValidatingAdmissionPolicies. Background-scan shouldn't generate reports for `validate.CEL` subrule to avoid having duplicate reports.
    - If it's set to `false`, reports won't be generated for ValidatingAdmissionPolicies. Background-scan should check the ValidatingAdmissionPolicy's owner references and if it's generated by a Kyverno policy, then reports won't be generated.

3. `failurePolicy`: No effect on ValidatingAdmissionPolicy generation.
4. `generateExisting`: No effect on ValidatingAdmissionPolicy generation.
5. `mutateExistingOnPolicyUpdate`: No effect on ValidatingAdmissionPolicy generation.
6. `schemaValidation`:  No effect on ValidatingAdmissionPolicy generation.

7. `validationFailureAction`: 
    - If it's set to `Enforce`, then `validationActions` will be `[Deny]` in the ValidatingAdmissionPolicyBinding.
    - If it's set to `Audit`, then validationActions can be either `[Audit]` or `[Audit, Warn]` in the ValidatingAdmissionPolicyBinding.

8. `validationFailureActionOverrides`: Both `action` and `namespaceSelector` can be converted into ValidatingAdmissionPolicyBinding.

    Kyverno policy:

    ```yaml=
    spec:
      validationFailureActionOverrides:
        - action: "Enforce"
          namespaceSelector: 
            matchExpressions:
              - key: "environment"
                operator: In
                values:
                  - prod
                  - staging
    ```

    ValidatingAdmissionPolicyBinding:

    ```yaml=
    spec:
      validationActions: [Deny]
      matchResources: 
        namespaceSelector: 
            matchExpressions:
              - key: environment
                operator: In
                values:
                  - prod
                  - staging
    ```

9. `webhookTimeoutSeconds`: No effect on ValidatingAdmissionPolicy generation.

### Match/Exclude Resources

- In ValidatingAdmissionPolicies, a request must match _all_ Constraints in `spec.matchConstraints` to be validated aganist the policy whereas `spec.matchConstraints.resourceRules` describes what operations on what resources/subresources the ValidatingAdmissionPolicy matches. The policy cares about an operation if it matches _any_ Rule. 
  In other words, `spec.matchConstraints` is AND-ed operation whereas `spec.matchConstraints.resourceRules` is OR-ed operation.

  Example:

  ```yaml=
        spec:
          matchConstraints:
            resourceRules:
              - apiGroups:     [""]
                apiVersions:   ["v1"]
                operations:    ["CREATE", "UPDATE"]
                resources:     ["pods"]
              - apiGroups:     ["apps"]
                apiVersions:   ["v1"]
                operations:    ["CREATE", "UPDATE"]
                resources:     ["deployments"]
            objectSelector:
              matchLabels:
                app: critical
  ```

  The matched resources are either pods with label `app: critical` OR deployments with label `app: critical`.


- `(match/exclude).(all/any).resources[0]` can be converted into `spec.matchConstraints.resourceRules` and `spec.matchConstraints.excludeRules` respectively as follows:
    - Kyverno policy:

      ```yaml=
      match:
        all:
          - resources:
              kinds:
                - Deployment
              names: 
                - "staging"
              operations:
                - CREATE
      ```

      ValidatingAdmissionPolicy:

      ```yaml=
      spec:
        matchConstraints:
          resourceRules:
          - apiGroups:     ["apps"]
            apiVersions:   ["v1"]
            operations:    ["CREATE"]
            resources:     ["deployments"]
            resourceNames: ["staging"]
      ```

      In case `match.all.resources.operations` isn't set, `matchConstraints.resourceRules[*].operations` will be set to CREATE, UPDATE, DELETE and CONNECT since it's a required field.

      NOTE: ValidatingAdmissionPolicies don't support wildcards in the resource names. Therefore, VAPs will only be generated in case the resource name doesn't contain any wildcards.
    
    - Kyverno policy:
      ```yaml=
      match:
        all:
        - resources:
            kinds:
            - Pod
            operations:
            - CREATE
            - UPDATE
            selector:
              matchLabels:
                app: critical
      ```

      ValidatingAdmissionPolicy:

      ```yaml=
      spec:
        matchConstraints:
          resourceRules:
            - apiGroups:     [""]
              apiVersions:   ["v1"]
              operations:    ["CREATE", "UPDATE"]
              resources:     ["pods"]
          objectSelector:
            matchLabels:
              app: critical
      ```

      `match.all.resources[0].selector` and `match.all.resources[0].namespaceSelector` can be converted into `matchConstraints.objectSelector` and `matchConstraints.namespaceSelector` respectively

    - Kyverno policy:

      ```yaml=
      match:
        any:
        - resources:
            kinds:
            - Deployment
            - Statefulset
            operations:
            - CREATE
            - UPDATE
            selector:
              matchLabels:
                app: critical
      ```

      ValidatingAdmissionPolicy:

      ```yaml=
      spec:
        matchConstraints:
          resourceRules:
            - apiGroups:     ["apps"]
              apiVersions:   ["v1"]
              operations:    ["CREATE", "UPDATE"]
              resources:     ["deployments", "statefulsets"]
          objectSelector:
            matchLabels:
              app: critical
      ```

- `(match/exclude).(all/any).resources[*].namespaces` can be converted into `spec.matchConstraints.namespaceSelector` as follows:
    - Kyverno policy:

      ```yaml=
      match:
        any:
        - resources:
            kinds:
            - Deployment
            operations:
            - CREATE
            - UPDATE
          namespaces:
            - ns-1
            - ns-2
      ```

      ValidatingAdmissionPolicy:

      ```yaml=
      spec:
        matchConstraints:
          resourceRules:
            - apiGroups:     ["apps"]
              apiVersions:   ["v1"]
              operations:    ["CREATE", "UPDATE"]
              resources:     ["deployments"]
          namespaceSelector:
            matchExpressions:
              - key: kubernetes.io/metadata.name
                operator: In
                values:
                - ns-1
                - ns-2
      ```
      
      Note: specifying namespaces by their names is not supported in ValidatingAdmissionPolicies. Therefore, in the generated VAPs, namespaceSelectors would be used to match a namespace name.

### Limitations

-  Kyverno policies of kind `Policy` that include a specific namespace can't be converted into ValidatingAdmissionPolicies as follows:

    ```
    apiVersion: kyverno.io/v1
    kind: Policy
    metadata:
      name: pol-1
      namespace: prod
    ```

- Kyverno policies can't have more than one `validationFailureActionOverrides` as follows:

  ```yaml=
      spec:
        validationFailureActionOverrides:
          - action: "Enforce"
            namespaceSelector: 
              matchExpressions:
                - key: "environment"
                  operator: In
                  values:
                    - prod
                    - staging
          - action: "Audit"
            namespaceSelector: 
              matchExpressions:
                - key: environment
                  operator: In
                  values:
                    - testing
  ```

- Kyverno policies can't have `validationFailureActionOverrides.namespaces` as follows:

  ```yaml=
      spec:
        validationFailureActionOverrides:
          - action: "Enforce"
            namespaces:
              - ns-1
              - ns-2
  ```

- `(match/exclude).(all/any).resources[*].annotations` can't be converted to ValidatingAdmissionPolicies.

   Examples: 
    ```yaml=
    spec:
      rules:
        - name: match-pod-annotations
          match:
            any:
            - resources:
                annotations:
                  imageregistry: "https://hub.docker.com/"
                kinds:
                  - Pod
                operations:
                - CREATE
                - UPDATE
    ```

- `(match/exclude).(any/all).subjects`, `(match/exclude).(any/all).roles` and `(match/exclude).(any/all).clusterRoles` can't be converted to ValidatingAdmissionPolicies.
   
   Example:
   ```yaml=
   match:
      any:
      - resources:
          kinds
          - Deployment
          operations:
          - CREATE
          - UPDATE
        subjects:
        - kind: User
          name: mary@somecorp.com
        clusterRoles: 
        - cluster-admin
   ```

- Multiple resources with different namespace/object selectors.
  
  Examples:
  ```yaml=
   match:
    any:
    - resources:
        kinds:
        - Deployment
        operations:
        - CREATE
        - UPDATE
        selector:
          matchLabels:
            app: critical-1
    - resources:
        kinds:
         - Statefulset
        operations:
        - CREATE
        - UPDATE
        selector:
          matchLabels:
            app: critical-2
  ```

- Multiple resources in which one of them specifies either an `objectSelector` or a `namespaceSelector`.

  Example:

  ```yaml=
   match:
    any:
    - resources:
        kinds:
        - Deployment
        operations:
        - CREATE
        - UPDATE
        selector:
          matchLabels:
            app: critical
    - resources:
        kinds:
         - Statefulset
        operations:
        - CREATE
        - UPDATE
  ```


### Validate Rules

`validate.cel.expressions` can be converted into `spec.validations` in ValidatingAdmissionPolicy as follows:

Kyverno policy:

```yaml=
validate:
  cel:
    expressions:
      - expression: "object.spec.replicas < 5"
        message:  "Deployment spec.replicas must be less than 5”
        messageExpression: "'Deployment spec.replicas set to ' + string(object.spec.replicas)
```

ValidatingAdmissionPolicy:

```yaml=
spec:
  validations:
    - expression: "object.spec.replicas <= 5"
      message: "replicas must be less than 5"
      messageExpression: "'Deployment spec.replicas set to ' + string(object.spec.replicas)"
```

### Audit Annotations

`validate.cel.auditAnnotations` can be converted into `spec.auditAnnotations` in ValidatingAdmissionPolicy as follows:

Kyverno policy:

```yaml=
validate:
  cel:
    auditAnnotations:
      - key: "high-replica-count"
        valueExpression: "'Deployment spec.replicas set to ' + string(object.spec.replicas)"
```

ValidatingAdmissionPolicy:

```yaml=
spec:
  auditAnnotations:
    - key: "high-replica-count"
      valueExpression: "'Deployment spec.replicas set to ' + string(object.spec.replicas)"
```

### Parameter Resources

Kyverno policy:
```yaml=
validate:
  cel:
    paramKind: 
      apiVersion: rules.example.com/v1
      kind: ReplicaLimit
    paramRef:
      name: "replica-limit-test.example.com"
    expressions:
      - expression: "object.spec.replicas <= params.maxReplicas"
        messageExpression:  "'Deployment spec.replicas must be less than ' + string(params.maxReplicas)"
```

ValidatingAdmissionPolicy:

```yaml=
spec:
  paramKind:
    apiVersion: rules.example.com/v1
    kind: ReplicaLimit
  validations:
    - expression: "object.spec.replicas <= params.maxReplicas"
```

ValidatingAdmissionPolicyBinding:

```yaml=
spec:
  paramRef:
    name: "replica-limit-test.example.com"
```

## Next Steps

1. Generate ValidatingAdmissionPolicies for multiple CEL subrules. 
2. Generate ValidatingAdmissionPolicies from namespaced kyverno policies if the namespaced policy binding is introduced. For more info, check the future plans in the [KEP](https://github.com/kubernetes/enhancements/tree/master/keps/sig-api-machinery/3488-cel-admission-control#namespace-scoped-policy-binding).
3. Support auto-gen CEL rules.
4. Provide a per-policy control whether Kyverno engine will handle CEL itself or let API server handles it.
5. Support writting policy exceptions to kyverno policies that are used to generate ValidatingAdmissionPolicies. 
