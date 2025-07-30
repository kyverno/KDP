# Meta
[meta]: #meta
- Name: The CEL Fine-Grained Exceptions
- Start Date: 2025-07-29
- Author(s): [Mariam Fahmy](https://github.com/MariamFahmy98)

# Table of Contents
[table-of-contents]: #table-of-contents
- [Overview](#overview)
- [Motivation](#motivation)
- [Proposal](#proposal)
- [Implementation](#implementation)

# Overview
[overview]: #overview

This proposal outlines the design for a fine-grained exception for Kyverno's CEL-based ValidatingPolicy.

The goals of this proposal are to:

1. **Excluding Specific Images:** A straightforward method to exempt resources from policy enforcement based on their container image reference.

2. **Allowing Specific Values:** A powerful and generic mechanism to allow a resource to contain a specific configuration value (e.g., a hostPath volume, a privileged security context) that a policy would normally deny.

# Motivation

Currently, Kyverno's ValidatingPolicy relies on `matchConstraints.resourceRules` and `matchConstraints.excluderesourceRules` blocks to control policy scope. While effective for excluding entire resources based on metadata (labels, namespaces, names), this approach will not work for many real-world scenarios. For example:

1. A policy might disallow a specific capability for all containers, but a single, trusted container within a Pod needs it. Excluding the entire Pod is not a secure option.

2. A policy might enforce a set of approved volume types, but a specific application requires a `hostPath` volume.

Writing complex CEL logic within the policy itself to handle these one-off cases is hard to manage, and violates the principle of separating policy definition from exceptions. This proposal introduces a dedicated, structured way to manage these fine-grained exceptions.

# Proposal

The main idea is to use the existing `PolicyException` resource to gather exception data and inject it into the CEL evaluation context. This makes the exception data available to the ValidatingPolicy expression through a new kyverno CEL variable.

If one or more PolicyException resources match an incoming resource for a given policy, Kyverno will aggregate their specifications and populate two new context variables:

1. `kyverno.excludedImages`: A list of image strings to be exempted.
2. `kyverno.allowedValues`: A structured map containing permissible values for specific fields in the resource.

If no exception applies, these variables will be empty (empty list and empty map, respectively), ensuring the policy functions normally without any modification.

## Excluding Specific Images

This feature allows users to specify a list of container image names that should be exempt from policy enforcement. This is particularly useful for scenarios where certain images are known to be safe or necessary, and the policy should not apply to them.

Let's use a consistent example: a policy that disallows privilege escalation in all Pods.

```yaml
apiVersion: policies.kyverno.io/v1alpha1
kind: ValidatingPolicy
metadata:
  name: "disallow-privilege-escalation"
spec:
  matchConstraints:
    resourceRules:
    - apiGroups:   [""]
      apiVersions: ["v1"]
      operations:  ["CREATE", "UPDATE"]
      resources:   ["pods"]
  validations:
    - expression: >- 
        object.spec.containers.all(container, 
           has(container.securityContext) &&
           has(container.securityContext.allowPrivilegeEscalation) &&
           container.securityContext.allowPrivilegeEscalation == false)
```

Now, let's say we need to create an exception for the `nginx:latest` and `busybox:latest` images. There are two primary ways to achieve what we need.

1. **Using a Logical OR in the Expression**:

    This method modifies the expression to include a bypass condition. The expression effectively becomes: "A container is valid if it is on the exclusion list OR it meets the security criteria."

```yaml
apiVersion: policies.kyverno.io/v1alpha1
kind: ValidatingPolicy
metadata:
  name: "disallow-privilege-escalation"
spec:
  matchConstraints:
    resourceRules:
    - apiGroups:   [""]
      apiVersions: ["v1"]
      operations:  ["CREATE", "UPDATE"]
      resources:   ["pods"]
  variables:
    # list of excluded images
    - name: excludedImages
      expression: "['busybox:latest', 'nginx:latest']"
  validations:
    - expression: >
        object.spec.containers.all(container,
          container.image in variables.excludedImages ||
          (
            has(container.securityContext) &&
            has(container.securityContext.allowPrivilegeEscalation) &&
            container.securityContext.allowPrivilegeEscalation == false
          )
        )
```


2. **Filtering the Containers**:

    This method uses a variable to create a pre-filtered list of items that the validation rule should apply to. The logic is: "Create a list of containers that are **NOT** on the exclusion list, and then validate that every container in this new list is compliant."

```yaml
apiVersion: policies.kyverno.io/v1alpha1
kind: ValidatingPolicy
metadata:
  name: "disallow-privilege-escalation"
spec:
  matchConstraints:
    resourceRules:
    - apiGroups:   [""]
      apiVersions: ["v1"]
      operations:  ["CREATE", "UPDATE"]
      resources:   ["pods"]
  variables:
    # create a new list of containers, excluding those with specified images
    - name: containersToValidate
      expression: >
        object.spec.containers.filter(container, !(container.image in ["nginx:latest", "busybox:latest"]))
  validations:
    - expression: >
        variables.containersToValidate.all(container,
          has(container.securityContext) &&
          has(container.securityContext.allowPrivilegeEscalation) &&
          container.securityContext.allowPrivilegeEscalation == false
        )
```

For fine-grained exceptions, we need to implement it in a similar way to the above examples. We will introduce a new field `images` in the PolicyException resource, which will allow users to specify a list of image names that should be exempt from policy enforcement.

```yaml
apiVersion: policies.kyverno.io/v1alpha1
kind: PolicyException
metadata:
  name: image-based-exceptions
spec:
  policyRefs:
    - name: disallow-privilege-escalation
      kind: ValidatingPolicy
  # list of image names to exclude
  images:
  - busybox:latest
  - nginx:latest
```

In order to use the excluded images from the exception in the policy, Kyverno will introduce a new CEL object `kyverno.excludedImages` that will be available in the CEL expressions. This object will contain the list of images specified in the PolicyException resource. If no exception exists, `kyverno.excludedImages` is an empty list, and the policy functions normally.

The policy can then be modified to use this new object in the validation expressions as follows

```yaml
apiVersion: policies.kyverno.io/v1alpha1
kind: ValidatingPolicy
metadata:
  name: "disallow-privilege-escalation"
spec:
  matchConstraints:
    resourceRules:
    - apiGroups:   [""]
      apiVersions: ["v1"]
      operations:  ["CREATE", "UPDATE"]
      resources:   ["pods"]
  validations:
    - expression: >
        object.spec.containers.all(container,
          container.image in kyverno.excludedImages ||
          (
            has(container.securityContext) &&
            has(container.securityContext.allowPrivilegeEscalation) &&
            container.securityContext.allowPrivilegeEscalation == false
          )
        )
```

## Allowing Specific Values

This feature allows users to specify a specific configuration value that should be allowed in a resource, even if the policy would normally deny it. This is useful for scenarios where a specific configuration is necessary for certain resources, and the policy should not apply to them.

Consider a policy that denies all but a few known-safe volume types:

```yaml
apiVersion: policies.kyverno.io/v1alpha1
kind: ValidatingPolicy
metadata:
  name: enforce-approved-volume-types
spec:
  matchConstraints:
    resourceRules:
      - apiGroups:   [""]
        apiVersions: ["v1"]
        operations:  ["CREATE", "UPDATE"]
        resources:   ["pods"]
  validations:
    - expression: >-
        !has(object.spec.volumes) ||
        object.spec.volumes.all(vol, 
            has(vol.configMap) ||
            has(vol.csi) ||
            has(vol.downwardAPI) ||
            has(vol.emptyDir) ||
            has(vol.ephemeral) ||
            has(vol.persistentVolumeClaim) ||
            has(vol.projected) ||
            has(vol.secret)
        )
```

Now, let's say we need to allow a specific Pod to use a `hostPath` volume. To achieve this, we will introduce a new field `allowedValues` to the PolicyException resource. This field will be a map where keys correspond to logical names (chosen by the exception author) and values are lists of permitted values.

To allow the `hostPath` volume type for a pod named `nginx-pod`, the exception would be:

```yaml
apiVersion: policies.kyverno.io/v1alpha1
kind: PolicyException
metadata:
  name: allow-hostpath-for-nginx
spec:
  policyRefs:
    - name: enforce-approved-volume-types
      kind: ValidatingPolicy
  matchConditions:
    - expression: "object.metadata.name == 'nginx-pod'"
  allowedValues:
    volumeTypes:
      - hostPath
```

We can then modify the validation expression to check for the allowed values specified in the PolicyException resource.

```yaml
validations:
  - expression: >-
      !has(object.spec.volumes) ||
      object.spec.volumes.all(vol,
        kyverno.allowedValues.volumeTypes.exists(type, has(vol[type])) ||
        has(vol.configMap) ||
        has(vol.csi) ||
        has(vol.downwardAPI) ||
        has(vol.emptyDir) ||
        has(vol.ephemeral) ||
        has(vol.persistentVolumeClaim) ||
        has(vol.projected) ||
        has(vol.secret)
      )
```

The new CEL object `kyverno.allowedValues` will contain the allowed configuration values specified in the PolicyException resource. If no exception exists, `kyverno.allowedValues` is an empty map, and the policy functions normally. It is a map of allowed values for each field, allowing for flexible and specific exceptions.

## Combining Image Exclusions with Allowed Values

A powerful feature is the ability to tie allowed values to specific images. For instance, allowing `seccompProfile.type: 'Unconfined'` only for the `nginx` image.

The original policy that enforces the seccompprofile type can be defined as follows:

```yaml
apiVersion: policies.kyverno.io/v1alpha1
kind: ValidatingPolicy
metadata:
  name: enforce-seccomp-profile
spec:
  matchConstraints:
    resourceRules:
      - apiGroups:   [""]
        apiVersions: ["v1"]
        operations:  ["CREATE", "UPDATE"]
        resources:   ["pods"]
  validations:
    - expression: >
        object.spec.containers.all(container,
            !has(container.securityContext) ||
            !has(container.securityContext.seccompProfile) ||
            !has(container.securityContext.seccompProfile.type) ||
            container.securityContext.seccompProfile.type == 'RuntimeDefault' ||
            container.securityContext.seccompProfile.type == 'Localhost'
        )
```

But we need to allow 'Unconfined' but only for containers using the `nginx` image, so we can create the following PolicyException:

```yaml
apiVersion: policies.kyverno.io/v1alpha1
kind: PolicyException
metadata:
  name: allow-unconfined-for-nginx
spec:
  policyRefs:
    - name: enforce-seccomp-profile
      kind: ValidatingPolicy
  images:
    - nginx
  allowedValues:
    seccompProfileType:
      - Unconfined
```

The modified policy will then use both `kyverno.allowedValues` and `kyverno.excludedImages` in order to allow the `Unconfined` seccomp profile type only for the `nginx` image:

```yaml
validations:
  - expression: >
      object.spec.containers.all(container,
        !has(container.securityContext) ||
        !has(container.securityContext.seccompProfile) ||
        !has(container.securityContext.seccompProfile.type) ||
        (container.image in kyverno.excludedImages && 
        container.securityContext.seccompProfile.type in kyverno.allowedValues.seccompProfileType) ||
        container.securityContext.seccompProfile.type == 'RuntimeDefault' ||
        container.securityContext.seccompProfile.type == 'Localhost'
      )
```

# Implementation

