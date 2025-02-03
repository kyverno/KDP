# Meta
[meta]: #meta
- Name: Image Verification Policy
- Start Date: 2025-01-27
- Update data (optional): ~
- Author(s): vishal-chdhry
- Supersedes: N/A

# Table of Contents
[table-of-contents]: #table-of-contents
- [Meta](#meta)
- [Table of Contents](#table-of-contents)
- [Overview](#overview)
- [Definitions](#definitions)
- [Motivation](#motivation)
  - [Goals:](#goals)
- [Proposal](#proposal)
  - [Match Images](#match-images)
  - [Images](#images)
  - [Attestors](#attestors)
  - [Attestations](#attestations)
  - [Verifications](#verifications)
    - [Custom CEL functions](#custom-cel-functions)
      - [VerifyImageSignatures](#verifyimagesignatures)
      - [VerifyAttestationSignatures](#verifyattestationsignatures)
    - [Payload](#payload)
    - [ParseImageReference](#parseimagereference)
  - [Other details](#other-details)
    - [Caching](#caching)
    - [Digest Mutation](#digest-mutation)
- [Drawbacks](#drawbacks)
- [Alternatives](#alternatives)
- [Prior Art](#prior-art)
- [Unresolved Questions](#unresolved-questions)

# Overview
[overview]: #overview

This document proposes the new design for image verification policies. 

# Definitions
[definitions]: #definitions

1. **Image:** An image is an archive containing application and all of its dependencies.
2. **Metadata:** Metadata is used to describe software and the build environment. Provenance (origin) data, SBOMs, and vulnerability scan reports are the essential set of metadata required to assess security risks for software.
3. **Attestation:**  Authenticated metadata is used to attest to the integrity of a software system. Both custom and standardized metadata can be converted into attestations.
4. **Attestor:** An identity, such as a key or certificate that confirms or verifies the authenticity of an image or an attestors

# Motivation
[motivation]: #motivation

Image verification policies were first added to Kyverno in v1.4 to support Sigstore Cosign, and have since added support for Notary as well as verification of image attestations. The incremental growth of fatures has led to a complex structure. 
Kubernetes has changed a lot in the last few years and has added direct support for CEL but Kyverno's verifyImages policies still use JSON patch and JMESPath.

This re-design will help in simplifying the image verification policy structure and also integrate CEL into image verification.

## Goals:
- Simplify the syntax of image verification policy
- Integrate CEL into image verification
- Allow usage in non-Kubernetes environments

# Proposal

Here is a high level overview of the proposed image verification policy syntax:
```yaml
apiVersion: kyverno.io/v2alpha1
kind: ImageVerificationPolicy
metadata:
  name: sample
spec:
  failurePolicy: Fail
  failureAction: Enforce
  mutateDigest: true
  verifyDigest: true
  required: true
  matchConstraints:
    resourceRules:
    - apiGroups:   ["apps"]
      apiVersions: ["v1"]
      operations:  ["CREATE", "UPDATE"]
      resources:   ["pods"]
  credentials:
    secrets:
    - regcred
    helpers:
    - amazon
    - google
  matchImages:
    references:
    - "ghcr.io/*"
    expressions:
    - $image == "nginx"
  images:
  - name: containers
    expression: request.object.spec.containers.image
  - name: initContainers
    expression: request.object.spec.initContainers.image
  - name: ephemeralContainers
    expression: request.object.spec.containers.image
  - name: all
    expression: request.object.spec.{containers,initContainers,ephemeralContainers}.image
  attestors:
  - name: keyless
    cosign:
      keyless:
       identities:
       - issuerRegExp: .*kubernetes.default.*
         subjectRegExp: .*kubernetes.io/namespaces/default/serviceaccounts/default
      ctlog:
        url: http://rekor.rekor-system.svc
        ignoreTlog: false
  - name: kms
    cosign:
      key:
        kms: awskms:///arn:aws:kms:us-east-1:123456789012:alias/cosign-key
  attestations:
  - name: slsa
    intoto:
      type: https://slsa.dev/provenance/v0.2
  - name: sbom
    referrer:
      type: sbom/cyclone-dx
  verifications:
  - expression: >-
      verifyImageSignatures(images.initContainers[0], attestors.kms)
  - expression: >-
      payload(images.initContainers[0], attestations.sbom).builderId == "foo"
      && 
      payload(images.initContainers[0], attestations.slsa).version == "0.2"
```

It is split into following components:
1. Match Constraints
2. Images
3. Attesters
4. Attestations
5. Verifications

## Match Images
Match images adds a global image filtering criteria. Conditions defined in the `spec.matchImages` block are used in all the image verification CEL functions. If the image does not match a criteria, it is skipped. `matchImages` is a way of defining the scope of the policy. Match images is a required field.
`matchImages` has two fields:
`references`, which is an array of globs which are matched against the image, if one of them match, then the policy is applied to the image. This uses shell globbing syntax.
`expresions` is a array of CEL expressions that are applied to the image, if one of the expressions return true. then the policy is applied to the image. The image is accessible in the CEL expression as `$image`. Custom functions such as `parseReference` is also present here.

```yaml
 # ...
 matchImages:
   references:
   - "ghcr.io/*"
   expressions:
   - $image == "nginx"
```

## Images
Images is the list of images that can be referred in validation block for verification. It is not a requirement to define images here, but it makes it easier to refer to the images in the verification block. It is recommended to define the lists of images in images variable for custom resource and non kubernetes objects. The images field is automatically populated with containers, initContainers, ephemeralContainer and all for all pod controllers. Any additional definitions will be merged with this default list for pod controllers. The expression of images must return a string or an array of strings.

```yaml
images:
- name: containers
 expression: request.object.spec.containers.image
- name: initContainers
 expression: request.object.spec.initContainers.image
- name: ephemeralContainers
 expression: request.object.spec.ephemeralContainers.image
- name: all
 expression: request.object.spec.{containers,initContainers,ephemeralContainers}.image
```

## Attestors
Attestors are the trusted identities, keys and certificates, i.e. trusted authorities. They can be of several types, like Cosign Key, Keyless, KMS or Notary's certs. It can also be extended in the future for newer attestors. Attestors can be accessed in verification block using `attestors.<name>`. Attestors and Attestations serve different purposes. Attestors are trust information, Attestations are additional image metadata that should be verified, separating them simplifies the structure. The relationship between attestors and attestations is described in `verification` block.

Attestors and Attestations are separated from each other, unlike current schema where users can specify `verifyImages.attestors` to verify only images and `verifyImages.attestations.attestors` to verify images and attestations which is complex and too nested. That can lead to confusion and makes policies hard to read because of nested YAML syntax.

```yaml
attestors:
- name: keyless
  cosign:
    keyless:
     identities:
     - issuerRegExp: .*kubernetes.default.*
       subjectRegExp: .*kubernetes.io/namespaces/default/serviceaccounts/default
    ctlog:
      url: http://rekor.rekor-system.svc
      ignoreTlog: false
- name: kms
  cosign:
    key:
      kms: awskms:///arn:aws:kms:us-east-1:123456789012:alias/cosign-key
```

## Attestations
Attestations are additional image metadata attached to an image by different methods such as In-toto attestations or artifacts associated to the image using the OCI 1.1 Referrers API. These attestations are fetched from the registry and then can be verified against the attestors. Their payload can also be accessed in verification block as json objects using `attestations.<name>` to run CEL expressions against them.
```yaml
attestations:
- name: slsa
  intoto:
    type: https://slsa.dev/provenance/v0.2
- name: sbom
  referrer:
    type: sbom/cyclone-dx
```

## Verifications
Verifications are similar to validation policies' `spec.validation`. They can be used to specify CEL expressions to perform image verification, or attestation payload validation. The entire admission request is also present in the context as `request` object and CEL expressions can be written to verify that.

```yaml
verifications:
- expression: >-
    verifyImageSignatures(images.initContainers[0], attestors.kms)
- expression: >-
    payload(images.initContainers[0], attestations.sbom).builderId == "foo"
    && 
    payload(images.initContainers[0], attestations.slsa).version == "0.2"
```

### Custom CEL functions

Some custom CEL functions are added to faciliate image and attestation verification.
#### VerifyImageSignatures
```go
verifyImageSignatures(
  image string, # must be a valid image,
  attestors: []string # must be defined in attestors block
) count int
```
`verifyImageSignatures` takes an image and a list of attestors by name. It returns the count of images verified.
NOTE: Only the images that match `spec.matchConstraint.imageRules` are verified, others are skipped

Examples:
`verifyImageSignatures(images.initContainers[0], [attestors.kms])` will verify the first initContainer image against the kms attestor.

#### VerifyAttestationSignatures
`verifyAttestationSignatures` takes an image, an attestation and a list of attestors. It returns the count of attestations verified in the image.
```go
verifyAttestationSignatures(
  image string, # must be a valid image
  attestation string, # must be defined in the attestation block
  attestors []string # must be defined in attestors block
) -> count: int
```

Examples:
`verifyAttestationSignatures(images.initContainers[0], attestation.sbom, [attestors.kms])` will verify the SBOM attestation in all the init container images against the kms attestor

### Payload
`payload` takes an image and an attestor and returns the attestation payload for that image.
```go
payload(
  image string, # must be a valid image
  attestation string, # must be defined in the attestation block
) object map[string]interface{}
```

### ParseImageReference
`parseImageReference` takes a valid image string and returns an object containing components of the image.
```go
parseImageReference(
  image string, # must be a valid image
) struct {
      registry string,
      repository string,
      tag string,
      digest string,
      referenceWithTagandDigest string
}
```

## Other details

### Caching
The image and attestation verification outputs will be cached in the pod with keys `policyid/attester_name/image` and `att/policyid/attester_name/image/attestation_name` When CEL functions are invoked, these entries will be checked. The entries expires based on a TTL value.

### Digest Mutation
Kyverno will mutate digest by default for all matching images, it will also preserve the tag information, images will be referenced as `<image>:<tag>@<digest>`. This can be disabled using `spec.mutateDigest`
# Drawbacks

Why should we **not** do this?

# Alternatives

- What other designs have been considered?
- Why is this proposal the best?
- What is the impact of not doing this?

# Prior Art

Discuss prior art, both the good and bad.

# Unresolved Questions

- What parts of the design do you expect to be resolved before this gets merged?
- What parts of the design do you expect to be resolved through implementation of the feature?
- What related issues do you consider out of scope for this KDP that could be addressed in the future independently of the solution that comes out of this KDP?

