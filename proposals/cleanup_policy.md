# New Policy Type - DeletingPolicy
[meta]: #meta
- Name: New Policy Type - Cleanup Policy
- Start Date: 2025-05-07
- Update data 2025-05-07
- Author(s): fjogeleit, eddycharly
- Supersedes: N/A

# Table of Contents
[table-of-contents]: #table-of-contents
- [Meta](#meta)
- [Table of Contents](#table-of-contents)
- [Overview](#overview)
- [Motivation](#motivation)
- [Proposal](#proposal)

# Overview
[overview]: #overview

Reimplementation of CleanupPolicies as part of the new CEL driven and vap/map inspired policy types. See [Kyverno Design Proposal - New Policy Types](https://github.com/kyverno/KDP/pull/66)

# Motivation
[motivation]: #motivation

As the Kubernetes ecosystem and feature set has evolved, CEL has become the preferred method for defining build-in policies. As part of our initiative to align our policy APIs with the new Kubernetes build-in types, we want to provide new policy types for our existing feature set to align with these standards, improve usability and performance, and lower the barrier to entry for new users.

# Proposal

- Supporting the same featureset as the existing DeletingPolicy type
- Using generic and similar APIs as arealdy implemented new policy types like ValidatingPolicy
    - MatchConstraints for resource selection
    - Conditions which using the MatchCondition API to define CEL based expressions for resource targeting

## DeletingPolicy API

This example showcases the conversion of the current CleanupPolicy API to the new CEL based approach.

```yaml
apiVersion: policies.kyverno.io/v1alpha1
kind: DeletingPolicy
metadata:
  name: deleting-policy
spec:
  matchConstraints:
    resourceRules:
    - apiGroups:   [""]
      apiVersions: ["v1"]
      resources:   ["pods"]
  variables:
    - name: name
      expressions: >
        target.metadata.name
  conditions:
    - expression: "variables.name == 'example'"
  deletionPropagationPolicy: Foreground
  schedule: "*/2 * * * *"
```

The new API uses matchConstraints similar to the native VAP / MAP and new kyverno policy types VPOL and IVPOL. The conditions now use the MatchCondition API and are based on CEL expressions to targeting resources.

* DeletingPolicies will support variables. They can be used in conditions
* Schedule defines a valid cron expression to define when the deleting policy will be executed
* DeletionPropagationPolicy defines how resources will be deleted (Foreground, Background, Orphan)

The built-in variable `target` is supported to reference the target object. For example, the variable `target` is used in this policy to represent the pod resource and test its name for a given value.