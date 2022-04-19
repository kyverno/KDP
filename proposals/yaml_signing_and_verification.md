# YAML Signing & Verification 

Name: YAML signing and verification \
Start Date: 2021-12-25 \
Authors: [Namanl2001](https://github.com/Namanl2001), [JimBugwadia](https://github.com/JimBugwadia)\
Supersedes: N/A

## Overview

This proposal adds support for verifying the integrity and source of a Kubernetes resource using cosign signatures and a Kyverno policy.

## Motivation

For certain Kubernetes resources, like central policies applied across multiple clusters, it is important to ensure that the resources can be trusted, and that any attempts to tamper with the resource  definition is blocked.  

Signatures are commonly used to verify origin and content of critical resources. For example, container image signing is used to verify the source and integrity of images.

This feature uses signatures to sign resource declarations via Cosign, and verify the integrity of the resource using Kyverno.

## Proposal

The proposal is to support the sigstore [k8s-manifest CLI](https://github.com/sigstore/k8s-manifest-sigstore)  for signing YAMLs using the option to embed the signature into the YAML declaration:

Here is an example of signing an nginx pod using a static key:

```
 kubectl sigstore sign -f nginx-pod.yaml  --key cosign.key
```

This produces a `nginx-pod.yaml.signed` declaration that looks like this:

```
 apiVersion: v1
kind: Pod
metadata:
  annotations:
    cosign.sigstore.dev/message: H4sIAAAAAAAA/wAzAcz+H4sIAAAAAAAA/+ySPU/DMBCGM+dXnLqH2v2IW6/sKANiP5IjMjg+y3YLBfHfUdwKmEB0qZDyLCf7PT/+kOdp8POWBx8oRuP6KmGo+tdlLWW9larezl1v3Evlubs64GCLMxBCCLVej1WqtfheM0u1KuRK1lJJpepFIRZCrZYFiHM2+yu7mDAUQjya4ce+3/LTXT7rPwG9uaMQDTsNe1k+GddpaLgrB0rYYUJdAqBznDAZdlHD23sJYPGebBwzALSWn6uOLCWKGmYPaCPNSgCHA2nw3FX5D5XRUzsuadklNI5CFlRgBuxJQ27SFhPFlMV5vtlZ27A17UHDDe0p5OioPmrHseeQTsepvvwNh6RhIzYiJwA+cOKWrYbb66a89NtPTExMXJKPAAAA//+lixZxAAgAAAEAAP//QYqwEDMBAAA=
    cosign.sigstore.dev/signature: MEQCIGJ9cjguB6TXcMGCDyWyCRyVOb6YOrHDo6S4VtZGuvVjAiA1qm5N/koj94I4JtL5BywhuBVxS7VpEkGLtcwIkVDogg==
  labels:
    allow-deletes: "false"
  name: pod-nginx
spec:
  containers:
  - image: nginx:latest
    imagePullPolicy: Never
    name: nginx
    ports:
    - containerPort: 8080
      protocol: TCP

```

This declaration can then be verified by a Kyverno policy rule. This policy checks that all Kyverno policies are signed using a static key:

```
---
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: check-image
spec:
  rules:
    - name: check-manifest
      match:
        resources:
          kinds:
            - ClusterPolicy
            - Policy
      validate:
        manifests:
          - key: |-
            -----BEGIN PUBLIC KEY-----
                      ...
            -----END PUBLIC KEY-----
```

This option of signing and verifying YAMLs is compatible with GitOps workflows and does not require an OCI registry to store signed resource definitions.

## Implementation

Although referred to as “YAML signing” this feature may be more accurately described as verifying the integrity of a Kubernetes manifest or resource declaration. YAML is used as a canonical format during signing and verification.

Since Kubernetes resources are typically mutated during admission controls by native and 3rd party controllers, this feature needs to be able to allow certain mutations, and ignore them during serialization and verification.

## Requirements

1. Kyverno should support a new `validate.manifests` declaration to allow verifying a YAML has been signed by one or more signatures.
2. The declaration must support a list of keys and allow logical AND / OR operations to allow signing by at least one of the keys for key rotation, a minimum count of keys, and multiple keys e.g. team and group level.  
3. A Kyverno policy rule for YAML verification should automatically allow (i.e. ignore) mutations performed by “trusted” controllers such as Kubernetes controllers and  Kyverno mutate rules.  


## Drawbacks

* Each resource needs to be signed vs. creating a resource bundle for a set of resources.
* Calculating the canonical form of the resource requires a “dry-run” to eliminate changes made based on defaults and other admission controllers.
* Performing a dry-run create requires Kyverno to have permissions to create the resource.

## Known Implementations

The Stolstron project Contains OpenShift/OKD compatibility projects for Open Cluster Management. Stolostron Integrity Shield provides:

* Signing and verification with GPG, X.509 certificates, and Sigstore cosign
* An admission controller with custom policy resources [ManifestIntegrityConstraint ](https://github.com/stolostron/integrity-shield/blob/master/docs/README_GETTING-STARTED-TUTORIAL.md#create-a-sample-manifestintegrityconstraint)and [ManifestIntegrityState](https://github.com/stolostron/integrity-shield/blob/master/docs/README_GETTING-STARTED-TUTORIAL.md#check-the-resource-integrity-status-from-the-manifestintegritystate-generated-by-observer).
* An Gatekeeper/OPA extension
* An operator

More information at: [https://github.com/stolostron/integrity-shield#integrity-shield](https://github.com/stolostron/integrity-shield#integrity-shield) 


# Alternatives

## Signed OCI Bundle

Kyverno policies can be stored in an OCI registry and signed as an OCI bundle. This is discussed in:  

## Integrate Integrity Shield library with Kyverno

An alternative to directly using the Sigstore [k8s-manifest-sigstore](https://github.com/sigstore/k8s-manifest-sigstore) library is to integrate with the IntegrityShield library.

This gist provides an implementation:

[https://gist.github.com/rurikudo/60a537ee25ca9134decbce248491c669](https://gist.github.com/rurikudo/60a537ee25ca9134decbce248491c669)

### Benefits

1. Support for GPG and X.509, in addition to Cosign signatures.
2. out-of-the-box support for [skipping users](https://github.com/stolostron/integrity-shield/blob/c8b478ab69e00080df5d9ec528e5fd529859e1c2/shield/resource/manifest-verify-config.yaml#L64-L223)  (e.g. replicaset-controller service account for Pods)
3. manifest verification with "dry-run & compare" which provides robust matching against mutation of manifests.

### Drawbacks

1. An additional layer of dependency will be introduced for Kyverno, which will introduce coupling across the projects. 
2. Support for skipping users and objects may be better controlled in the policy declarations rather than internals. If it’s best to implicitly handle, or some combination of implicit vs explicit control is required, Kyverno’s match / exclude function can be leveraged.
3. The “dry-run & compare” can be reused from the Sigstore implementation. It needs to be determined, if this is required for all resources or if users should be able to control its usage via policy.
4. There are a number of IBM and OpenShift references in the Stolstrom code which may not be applicable to Kyverno. For example, the default skip users.
5. We want to avoid any disk operations in kyverno and keep it read only. But sigstore requires file-system access to process the encrypted message and signature added to the metadata, after signing a manifest. To deal with this in kyverno we are trying to perform all the required tasks in-memory.

## Unresolved Questions

1. Determine if dry-run is really needed, or if Kyverno should provide support to exclude attributes that can be changed both via an implicit list for Kubernetes resource or an explicit list in the policy declaration.
2. Check if [Keyless](https://github.com/sigstore/cosign/blob/main/KEYLESS.md) signing and verification can be supported.

## CRD Changes
TODO

# Design

## Use a YAML subset as canonical format with k8s-manifest-sigstore

This would be similar to what is currently done via the sigstore Kubectl plugin ([https://github.com/sigstore/k8s-manifest-sigstore](https://github.com/sigstore/k8s-manifest-sigstore)). The plugin supports two options:

1. Store YAML signatures and digest in an OCI registry
2. Store YAML signature and digest in the YAML itself

The priority would be to investigate the 2nd option.Resources to test with 

* Unit tests 
* Testing with various resourceTypes done
* Use sigstore’s default-ignoreFields
* Added kyverno specific default ignoreFields
* Test with various resourceTypes again
* documentation

# References

* [GitHub - sigstore/k8s-manifest-sigstore: kubectl plugin for signing Kubernetes manifest YAML files with sigstore](https://github.com/sigstore/k8s-manifest-sigstore)
* [https://sigstore.slack.com/archives/C01DGF0G8U9/p1625691042453400?thread_ts=1625651994.446200&cid=C01DGF0G8U9](https://sigstore.slack.com/archives/C01DGF0G8U9/p1625691042453400?thread_ts=1625651994.446200&cid=C01DGF0G8U9)
* [GitHub - IBM/integrity-shield: Integrity Shield is a tool for built-in preventive integrity control for regulated cloud workloads. It includes signature based configuration drift prevention based on Admission Webhook on Kubernetes cluster.](https://github.com/IBM/integrity-shield)