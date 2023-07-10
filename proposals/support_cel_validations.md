# CEL Validation Support

- **Authors**: [Mariam Fahmy](https://github.com/MariamFahmy98)
- **Created**: April 29th, 2023
- **Abstract**: Support CEL expressions for resource validation in Kyverno Engine

## Overview
- [Introduction](#introduction)
- [Proposal](#proposal)

## Introduction
[Validating admission policies](https://kubernetes.io/blog/2022/12/20/validating-admission-policies-alpha/) use the [Common Expression Language](https://github.com/google/cel-spec) (CEL) to declare the validation rules of a policy. It's an alternative solution to [validating admission webhooks](https://kubernetes.io/docs/reference/access-authn-authz/extensible-admission-controllers/#what-are-admission-webhooks). 

To create a validating admission policy, we need:

1. `ValidatingAdmissionPolicy` defines the policy logic. (required)
2. `ValidatingAdmissionPolicyBinding` links resources together and provides scoping. (required)
3. A Parameter resource (optional)

For more information on the proposal and how to use validating admission policies, see:

- [KEP-3488: CEL for Admission Control](https://github.com/kubernetes/enhancements/blob/master/keps/sig-api-machinery/3488-cel-admission-control/README.md)
- [Validating admission policies](https://kubernetes.io/docs/reference/access-authn-authz/validating-admission-policy/)

This document is to introduce CEL support for the validate rules in kyverno policies.

## Proposal
In this section, we will dive deep into validating admission policy and how to support its features in kyverno engine.

### Match/Exclude Resources
#### Validating Admission Policies
Initially, we will discuss on what resources/subresources the ValidatingAdmissionPolicy matches/excludes:

1. Validating admission policies use: 

   - `spec.matchConstraints.resourceRules`: it identifies on which resources the CEL validation expressions must be applied. (required)
      
      Example:
      ```
      spec:
        matchConstraints:
            resourceRules:
                - apiGroups:   ["apps"]
                  apiVersions: ["v1"]
                  operations:  ["CREATE", "UPDATE"]
                  resources:   ["deployments"]
      ```
   - `spec.matchConstraints.excludeResourceRules`: it identifies on which resources the policy should exclude.
     
     Example:
     ```
      spec:
        matchConstraints:
            excludeResourceRules:
                - apiGroups:   ["apps"]
                  apiVersions: ["v1"]
                  operations:  ["CREATE", "UPDATE"]
                  resources:   ["deployments"]
      ```

    - `spec.matchConstraints.namespaceSelector`: it selects a namespace, the policy must be applied aganist resources created in that namespace.

      Example:
        ```
        spec:
        matchConstraints:
            namespaceSelector: 
                matchExpressions: 
                    - key: "environment"
                      operator: "In"
                      values: 
                         - prod
                         - staging
        ```

    - `spec.matchConditions`: It is a list of conditions that must be met for a request to be validated. it filter srequests that have already been matched by the rules, namespaceSelector, and objectSelector.
    The exact matching logic is (in order):
	  1. If ANY matchCondition evaluates to FALSE, the policy is skipped.
	  2. If ALL matchConditions evaluate to TRUE, the policy is evaluated.
	  3. If any matchCondition evaluates to an error (but none are FALSE):
	     - If failurePolicy=Fail, reject the request
	     - If failurePolicy=Ignore, the policy is skipped

        Example:
        In that case, the policy will be applied on a deployment whose name is `nginx-deployment`
        ```
        spec:
            matchConstraints:
                resourceRules:
                     - apiGroups:   ["apps"]
                       apiVersions: ["v1"]
                       operations:  ["CREATE", "UPDATE"]
                       resources:   ["deployments"]
            matchConditions:
                - name: "deployment name"
                  expression: "object.metadata.name == 'nginx-deployment'"
        ```

    - `spec.matchConstraints.objectSelector`: it decides whether to run the CEL validation expression based on if the object has matching labels.

2. Validating admission policy binding uses the same fields in `spec.matchConstraints`. It's intersected with the policy's matchConstraints, so only requests that are matched by the policy can be selected by this.

#### Kyverno Policies
We can use kyverno policies match/exclude checks to have the same functionality provided by the validating admission policies.

Here are the same examples but written as a Kyverno policy:

1. To match resources: 
    ```
    match:
        any:
        - resources:
            kinds:
            - apps/v1/Deployment
    ```

2. To exclude resources:
    ```
    exclude:
        any:
        - resources:
            kinds:
            - apps/v1/Deployment
    ```

3. To match resources in a namespace with labels:
    ```
    match:
        any:
        - resources:
            kinds:
            - Deployment
            namespaceSelector:
                matchExpressions:
                    - key: enivronment
                    operator: In
                    values: 
                    - prod
                    - staging
    ```

4. To match resources with conditions:
In case `spec.matchConditions` have conditions that check the resource name, we can do it in Kyverno policies as follows:
    ```
    match:
        any:
        - resources:
            kinds:
            - apps/v1/Deployment
            names: 
            - "nginx-deployment"
    ```

    Otherwise, kyverno policies have limitations in providing the `spec.matchConditions` functionality.

    Therefore, we can support the same exact logic in kyverno policies by introducing a new field for cel in the `match` block as follows:
    ```
    celPreconditions: 
      - name: "condition name"
        expression: "cel expression that must be evaluated to bool"
    ```

    Example:
    In case we need to apply the policy on all deployments that have `latest` tag, we can do it as follows:
    ```
    celPreconditions:
      - name: "latest tag check"
        expression: "object.spec.template.spec.containers.all(container, container.image.contains('latest'))"
    ```

#### Conclusion
Kyverno policies already have its logic for the following:
1. `policy.spec.matchConstraints`
2. `policybinding.spec.matchConstraints`

However, it needs to extend its functionality to support `policy.spec.matchConditions`.

### Validate Rules
In validating admission policies, we have the following CEL validations:
```
spec:
  validations:
    - expression: "object.spec.replicas <= 5"
      message: "replicas must be less than 5"
      messageExpression: "'Deployment spec.replicas set to ' + string(object.spec.replicas)"
```

Alternatively, kyverno policies can have a subrule in `validate` to support CEL validations:
```
rules:
    - name: check-deployment-replicas
      match:
        any:
        - resources:
            kinds:
              - Deployment
      validate:
        cel:
          expressions:
            - expression: "object.spec.replicas <= 5"
              message: "replicas must be less than 5"
              messageExpression: "'Deployment spec.replicas set to ' + string(object.spec.replicas)"
```

### Audit Annotations
In validating admission policies, we can write audit annotations using CEL expressions as follows:
```
spec:
  auditAnnotations:
    - key: "high-replica-count"
      valueExpression: "'Deployment spec.replicas set to ' + string(object.spec.replicas)"
```

Alternatively, kyverno policies can have a new attribute in `validate.cel` subrule to support audit annotations expressions:
```
rules:
    - name: audit-annotation
      match:
        any:
        - resources:
            kinds:
              - Deployment
      validate:
        cel:
          auditAnnotations:
            - key: "high-replica-count"
              valueExpression: "'Deployment spec.replicas set to ' + string(object.spec.replicas)"
```

### Parameter Resources
A policy can define paramKind, which outlines GVK of the parameter resource, and then a policy binding ties a policy by name (via policyName) to a particular parameter resource via paramRef.

A native type such as ConfigMap or a CRD defines the schema of a parameter resource.

For more information, check [Parameter Resources in VAPs](https://kubernetes.io/docs/reference/access-authn-authz/validating-admission-policy/#parameter-resources).

To support parameter resources in kyverno policies, a new attribute will be introducted in `validate.cel` that contains apiVersion, kind, name and namespace for the parameter resource:
```
rules:
    - name: check-deployment-replicas
      match:
        any:
        - resources:
            kinds:
              - Deployment
      validate:
        cel:
          paramKind:
            apiVersion: v1
            kind: ConfigMap
          paramRef:
            name: replica-limit-set
            namespace: default
          expressions: 
            - expression: "object.spec.replicas <= params.maxReplicas"
```

A config map (replica-limit-set) must be created as follows:
```
apiVersion: v1
kind: ConfigMap
metadata:
  name: game-demo
data:
  maxReplicas: 5
```

We will use the information provided by `validate.cel.paramKind` to get the resource from the cluster and use it in `validatingadmissionpolicy.validate()` that's responsible for validating CEL expressions.

## Validating Admission Policy, Phase 2
We need to generate `ValidatingAdmissionPolicy` and `ValidatingAdmissionPolicyBinding` from the kyverno CEL subrule. To do so, we will create two controllers as follows:
1. The 1st controller is to listen to kyverno policies and generate both validating admission policy and its binding via informers. We can implement its functionality either in a separate controller or in the background controller.
2. The 2nd controller is to listen to the existing VAPs and generate report requests intermediate objects for reports. We can implement its functionality either in the reports controller or in the background controller.

