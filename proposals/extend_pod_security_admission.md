# Extending Pod Security Admission

**Author**: Shuting Zhao (shuting@nirmata.com)
**Date**: Dec 15th, 2021
**Update**: June 15th, 2022
**Abstract**: Using Kyverno to extend Pod Security Admission for Kubernetes.

## Contents

- [Contents](#contents)
- [Introduction](#introduction)
- [The Problem](#the-problem)
- [Proposed Solution](#proposed-solution)
  - [Extend PSA via a new rule](#extend-psa-via-a-new-rule)
  - [Exceptions for container security context](#exceptions-for-container-security-context)
- [Alternate Solutions Considered](#alternate-solutions-considered)
  - [Extend Via Annotations](#extend-via-annotations)
    - [Step 1. Enable additional policies for a namespace](#step-1-enable-additional-policies-for-a-namespace)
    - [Step 2. Fine-grained Policy Control](#step-2-fine-grained-policy-control)
- [Next Steps](#next-steps)

<small><i><a href='http://ecotrust-canada.github.io/markdown-toc/'>Table of contents generated with markdown-toc</a></i></small>

## Introduction

[Pod Security admission](https://kubernetes.io/docs/concepts/security/pod-security-admission/) (PSA) is a built-in solution that applies different isolation levels of [Pod Security Standards](https://kubernetes.io/docs/concepts/security/pod-security-standards/) for Pods. With the release of Kubernetes v1.23, PSA has entered beta and is enabled by default for 1.23+ clusters.

For more information on the proposal and how to use PSA, see:

- [KEP-2579: Pod Security Admission Control](https://github.com/kubernetes/enhancements/tree/master/keps/sig-auth/2579-psp-replacement)
- [how PSA is used in a cluster](https://hackmd.io/C0prhSfdTbSv9d1ag7Ft9g).

This document proposes a solution to extend PSA for finer-grained and flexible policy control.

## The Problem

Once PSA is enabled for namespaces, a configured level of Privileged, Baseline, or Restricted applies to all pods and workloads within the namespace. The level is configured as a label on the namespace. There is no option to select specific pods or control, for granular policies.

Users can choose to configure controls for select pods, but there is no validation or enforcement beyond meeting the requirements for the specified level. This means that if a namespace is baseline, any pod in that namespace can run with root user privileges.

Here is an example of this request: [Grant specific permissions for specific services](https://github.com/kubernetes/kubernetes/issues/108802).

A solution would be to use microsegmentation, and isolate privileged pods to their own namespace. However, this introduces other complexities such as managing networking across pods across namespaces, managing permissions, etc.

There are a few other considerations with PSA:

- For users, violations are visible only for `warn` and `enforce` mode. With `audit` there's no trace that users can track. Only someone that has access to the audit log (a control plane, protected feature) can.
- The violations are only returned along with admission responses. And the warning messages are not easy to parse. It exposes a metrics endpoint to record [evaluations](https://github.com/kubernetes/pod-security-admission/blob/5219c1944103298680f1298e30155ce541af8a02/metrics/metrics.go#L43-L47).
- No mutation ability.
- No support for any other resources than Pods.
- Workload controllers (ex., Deployments) [are not blocked](https://kubernetes.io/docs/concepts/security/pod-security-admission/#workload-resources-and-pod-templates) in `enforce` mode, only the Pods created indirectly by them.

## Proposed Solution

PSA is an in-tree solution for pod security compliance, and it seems best to enable users to leverage the built-in solution for pod security enforcement. Kyverno can be integrated with PSA to extend its ability and provide fine grained checks.

With this solution, Kyverno would reuse the in-tree pod security implementation and not require users to manage multiple Kyverno policies, one for each control.

### Extend PSA via a new rule

Kyverno can extend PSS profiles with a new validation rule, `validate.podSecurity`.

The rule will contain a "profile" that is set to "restricted" by default, and a list of exclusions that the user wants to allow. When set to "restricted", Kyverno will apply both Baseline and Restricted pod security profiles and block on violations.

If the PSA is enabled with "enforce=restricted", there's not much left for this Kyverno rule to do. Only if the PSA is in "enforce=baseline" or "enforce=privileged" mode, users can use the Kyverno rule for granular control.

The `podSecurity` validate rule applies configured [pod security profile level](https://kubernetes.io/docs/concepts/security/pod-security-admission/#pod-security-levels) to the selected namespaces. You can define exemptions from enforcement when creating pods. Exemptions can be configured via the `podSecurity.exclude` attribute.

For example, if a user wants to apply Restricted PSS to the selected namespaces "test" and "staging" but to skip checking the Control "Capabilities" for pods running "ghcr.io/example/nginx:1.2.3", the following policy does the job.

```yaml
validationFailureAction: enforce
rules:
  - name: enforce-restricted-exclude-all-capabilities-nginx:1.2.3
    match:
      any:
        - resources:
            kinds:
              - Pod
          namespaces:
            - test
            - staging
    exclude:
      any:
        - userInfo:
            username: dummyuser
    validate:
      # this new type of rule only deals with PSS profiles
      # as we need to check if the value in the resource
      # is actually allowed by PSS
      podSecurity:
        profile: restricted
        # version must be a valid Kubernetes minor version,
        # or `latest`.
        version: v1.24
        exclude:
          # controlName is the Control defined in PSS
          # +required
          - controlName: Capabilities
            images:
              - ghcr.io/example/nginx:1.2.3
```

If the user wants to specifically skip checking the custom "add" capabilities, `restrictedField` can be used to define the allowed list. For example, the following rule applies Restricted PSS to namespace "test" and "staging" while excluding pods running image "ghcr.io/example/nginx:1.2.3" to be able to add `kill`, `SETGID` and `SETUID` capabilities.

```yaml
validationFailureAction: enforce
rules:
  - name: enforce-restricted-exclude-capabilities-nginx:1.2.3
    match:
      any:
        - resources:
            kinds:
              - Pod
          namespaces:
            - test
            - staging
    exclude:
      any:
        - userInfo:
            username: dummyuser
    validate:
      # this new type of rule only deals with PSS profiles
      # as we need to check if the value in the resource
      # is actually allowed by PSS
      podSecurity:
        profile: restricted
        # version must be a valid Kubernetes minor version,
        # or `latest`.
        version: latest
        exclude:
          # controlName is the Control defined in PSS
          # +required
          - controlName: Capabilities
            images:
              - ghcr.io/example/nginx:1.2.3
            # +optional
            restrictedField: spec.containers[*].securityContext.capabilities.add
            # Any type
            # +optional
            values: # how to define an object?
              - "KILL"
              - "SETGID"
              - "SETUID"
```

Note that securityContext is configured at container level, not pod, as the matching image selects a specific container. The pod creation will be rejected if there's any violation found in its container. For [pod level checks](#fields-outside-container-object), other match criteria such as labels could be used.

In the following example we apply restricted PSS to namespaces "test" and "staging" and check the Control "Volumes" while excluding pods with the label "env: prod".

```yaml
validationFailureAction: enforce
rules:
  - name: enforce-restricted-exclude-all-volumes-prod
    match:
      any:
        - resources:
            kinds:
              - Pod
          namespaces:
            - test
            - staging
    validate:
      # this new type of rule only deals with PSS profiles
      # as we need to check if the value in the resource
      # is actually allowed by PSS
      podSecurity:
        profile: restricted
        # version must be a valid Kubernetes minor version,
        # or `latest`.
        version: v1.24
        exclude:
          # controlName is the Control defined in PSS
          # +required
          - controlName: Volumes
            labels:
              env: prod
```

### Policy Reports

For a PolicyReport, `cpol.rules.validate.podSecurity.exclude.restrictedField` and `cpol.rules.validate.podSecurity.exclude.controlName` will be added to `polr.results.properties` to audit the result.

```yaml
PolicyReport:
results:
  - message: |-
      'validation error: The label `app.kubernetes.io/name` is required. 
      Rule check-for-labels failed at path
      /metadata/labels/app.kubernetes.io/name/'
    policy: require-labels
    resources:
      - apiVersion: v1
        kind: Pod
        name: nginx
        namespace: default
        uid: 13fcb726-5597-4fcd-a708-ce8b558bd484
    rule: check-for-labels
    properties:
      controlName: Capabilities
      podSecurity: spec.containers[*].securityContext.capabilities.add
    scored: true
    status: fail
```

### Fields outside container object

- Host Namespaces: `spec.hostNetwork,spec.hostPID,spec.hostIPC`
- Sysctls: `spec.securityContext.sysctls[*].name`
- volumes: `spec.volumes`
- Non-root groups: `spec.securityContext.supplementalGroups[*], spec.securityContext.fsGroup`

For these pod level checks, we can use labels to select them.

## Alternate Solutions Considered

### Extend Via Annotations

This approach extends the PSA via annotations for granular control.

#### Step 1. Enable additional policies for a namespace

One should be able to configure additional pod-security policies by adding the following annotation to the Namespace:

```yaml
# Support modes are: warn, audit, enforce.
# The value of the annotation is comma-separated policy names.
# Multiple annotations with different modes can be configured
# for the same namespace.
# If a policy is enabled for more than one mode,
# the most restricted mode will take precedence.
pod-security.kyverno.io/<mode>=<policy1,policy2>
```

Once the annotation is detected, the corresponding policies (from
“kyverno/policies”) will be installed to the cluster as Namespace policies, and will be applied to all pods and pod controllers. The installed Namespace policy shares the same lifecycle of the configured annotation, if the policy is removed from the annotation, or if this annotation is removed completely from the Namespace, the policy will be removed from the cluster.

Note: We'll need to define what are the policies we would like to support via annotation. Once set, the policy name should be barely changed.

#### Step 2. Fine-grained Policy Control

We can start with label selectors to select matching resources, and extend to use annotation selectors if needed. Kyverno will look up given labels for the incoming pods and pod controllers, and apply the selected policies (by the annotation above) to the matching resources. This annotation must be configured along with `pod-security.kyverno.io/<mode>` for granular control.

```yaml
# selectors can be defined either in JSON string or Yaml format

# use label selector for exact match
pod-security.kyverno.io/label-selector: >-
    matchLabels:
        env: prod

# use label selector to match expressions
pod-security.kyverno.io/label-selector: >-
    matchExpressions:
    - key: env
      operator: In
      values:
      - "prod"
      - "staging"
```

## Next Steps

The first implementation will enable the new type of rule to be working together with PSA. The following requirements also need to be considered while developing.

- Add "warn" mode
- Support "auto-gen"
- Metrics support
- CLI support
- Instructions of how to set up the two to work together
