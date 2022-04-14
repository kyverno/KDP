# Kyverno Design Proposal - Image Verification Restructuring

Created: Jan 30, 2022
Authors: [Sambhav Kothari](https://github.com/samj1912), [Jim Bugwadia](https://github.com/JimBugwadia/)

## Contents

- [Kyverno Design Proposal - Image Verification Restructuring](#kyverno-design-proposal---image-verification-restructuring)
  * [Current State](#current-state)
    + [Image Signature Verification](#image-signature-verification)
    + [Image Attestation Checks](#image-attestation-checks)
    + [Image Configuration Checks](#image-configuration-checks)
  * [Problems](#problems)
    + [1. Image verification for custom resources](#1-image-verification-for-custom-resources)
    + [2. Tag to digest conversion](#2-tag-to-digest-conversion)
    + [3. Combining attestation checks with validation](#3-combining-attestation-checks-with-validation)
    + [4. Combining attestation checks with mutation](#4-combining-attestation-checks-with-mutation)
  * [Requirements](#requirements)
  * [Proposal](#proposal)
      - [Example: convert tags to digests for all pods](#example-convert-tags-to-digests-for-all-pods)
      - [Example: require all pod images are from a trusted registry and are signed using a public key](#example-require-all-pod-images-are-from-a-trusted-registry-and-are-signed-using-a-public-key)
      - [Example: require all pod images are from a trusted registry and are signed by 1 of 3 keys](#example-require-all-pod-images-are-from-a-trusted-registry-and-are-signed-by-1-of-3-keys)
      - [Example: require all Tekton task images are from a trusted registry and are signed using keyless](#example-require-all-tekton-task-images-are-from-a-trusted-registry-and-are-signed-using-keyless)
      - [Example: require all pod images are have a Trivy image scan attestation with keyless signing](#example-require-all-pod-images-are-have-a-trivy-image-scan-attestation-with-keyless-signing)
  * [Alternatives](#alternatives)
    + [Alternate Proposal 1](#alternate-proposal-1)
      - [Mutate tags to digests](#mutate-tags-to-digests)
      - [Verify image has valid signatures](#verify-image-has-valid-signatures)
    + [Alternate Proposal 2](#alternate-proposal-2)
      - [Verify image signatures](#verify-image-signatures)

<small><i><a href='http://ecotrust-canada.github.io/markdown-toc/'>Table of contents generated with markdown-toc</a></i></small>


## Current State

Here are some features Kyverno currently provides:

### Image Signature Verification

The Kyverno “verifyImages” rule can verify image signatures and replaces image tags with a digest. Details at: https://kyverno.io/docs/writing-policies/verify-images/.

### Image Attestation Checks

Besides verifying signatures, the verifyImages rule can also check in-toto attestation predicates attached in cosign format. See: https://kyverno.io/docs/writing-policies/verify-images/#verifying-image-attestations.

### Image Configuration Checks

In v1.6.0, Kyverno can verify image manifest and image config data fetched from an OCI registry. The documentation is pending, but several examples are in this PR: https://github.com/kyverno/kyverno/pull/2946 and this issue https://github.com/kyverno/policies/issues/226

## Problems

While the above features are nice, they present a few limitations:

### 1. Image verification for custom resources

Currently image verification only occurs for resources like Pods, Deployments etc. It is currently difficult to extend it to custom resources.

### 2. Tag to digest conversion

It is desirable to separate the “tag-to-digest” mutation from the image signing i.e. users should not have to sign images to mutate the tags. This is already possible with the `imageRegistry` external data source introduced in Kyverno 1.6. but requires configuring a [separate policy](https://kyverno.io/policies/other/resolve_image_to_digest/resolve-image-to-digest/). This logic can be consolidated and simplified for users.

### 3. Combining attestation checks with validation

Attestations are available in the `imageVerify` rule, while image manifest and configuration data is available in the validate rule type.

The validate rule is also more expressible and flexible as it allows for features like `deny`, `foreach` and `pattern`.

### 4. Combining attestation checks with mutation

Attestations are available in the `imageVerify` rule, while image manifest and configuration data is available in the mutation rule type.

There might be cases when you would like to mutate the resource based on the attestation. For eg. being able to add env. vars (for log4j remediation) or stricter network policies based on vulnerability remediation or vulnerability's CVSS vector respectively.

## Requirements

The following functional requirements shall be supported:

1. Support tag-to-digest for all images. Make the conversion auditable (events, annotation).
2. Support custom resources with images (Tekton tasks, Argo workflows).
3. Allow validate and mutate logic to be performed after checking signatures and attestations, and using image configuration data.
4. Allow using attestation data, and image configuration data, in validation and mutation rules.
5. Allows flexibility in key/keyless matches (e.g. 2 of 5 keys must match). Currently Kyverno supports only an "AND" list of keys, and a single keyless subject and issuer. NOTE: this may require Cosign changes to move towards an [SDK](https://docs.google.com/document/d/1DoOOF29FEgjzqHwQ0Depar9Vm1lkCk_lwt8CGCyyAR0/edit#) model.
6. Allow setting media types (or other Cosign configuration related env flags) in the policy rule or in the process. 
7. Allow use cases where a policy can match a CVE and apply mutation logic.
8. Combine verification of signature and attestations in a single policy rule.
9. Allow access to attestations and signatures for mutation and validation logic.

## Proposal

A single `imageVerify` rule will contain:
- zero or more attestors (authorities)
- zero or more in-toto attestations
- an optional`mutate` clause
- an `validate` clause 

Attestations and signatures would be available in the policy rule context, to allow lookups in the mutate and validate clauses. 

New fields for enforcing and image digest and globally requiring signatures are added. 

The high-level structure would look something like this:


```yaml
apiVersion : kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: validate-images
spec:
  rules:
  - name: check-image
    match:
      resources:
        kinds:
        - Pod
    context:
    
    # Specify a custom location for images in the resource
    # The results are merged into the images context variable
    # Optional.
    - imagePaths:
        # JMESPath to locate the objects that contain the image
      - path: "{{ spec.tasks[*] }}"
        # key field used to uniquely reference the image 
        # optional. defaults to "name"
        keyField: name
        # value field that holds the image
        # optional. defaults to "image"
        valueField: image
        
    imageVerify:
    
      # Specify the behavior for images that do not match
      # any configured rule:
      # - allow: allow image to run an report a violation
      # - deny: prevent the image from running
      default: deny
      
      # Check for a digest in the image reference. The policy
      # rule fails if a digest is not specified.
      # Optional.
      digestVerify: true
      
      # Update the image reference with a digest. 
      # Optional.   
      digestMutate: true
      
      # Match one or more image patterns
      imageReferences: 
      - registry.io/org/*
        
      # Specify one or more attestors (keys, keyless, other) that 
      # must sign the image or attestions.
      # At least 1 attestor is required
      attestors:
      - entries: 
        # a static key entry
        - staticKey: {}
        # a keyless entry
        - keyless: {}
        # a nested attestor set
        - attestors: {}
        # Optional count of attestors. If specified the
        # entry must be between 0 and the size of the list. If 0 or nil is 
        # specified, all entries must match. Validated signatures are
        # made available in the Context for custom checks.
        count: 1
        
      # Specify one or more attestation checks.
      # optional
      attestations:
      - predicateType: "..."
        conditions:
        - all:
          - key: "{{ ... }}"
            operator: Equals
            value: "..."
            
      # Specify mutate behaviors applied if and after attestors 
      # are validated. Signatures and Attestation data can be 
      # referenced from the Context.
      # Optional.
      mutate:
          ...
          
      # Specify additional validate logic applied if and after 
      # attestors are validated. Signatures and Attestation data 
      # can be referenced from the Context.
      # Optional.
      validate:
          ....

```

#### Example: require digests for all pods, and convert tags to digest if needed

```yaml
apiVersion : kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-digest
spec:
  rules:
  - name: replace-tags
    match:
      resources:
        kinds:
        - Pod
    imageVerify:
    - images:
      - "*"
    - digestMutate: true
      digestVerify: true
```

#### Example: require all pod images are from a trusted registry and are signed using a public key

```yaml
apiVersion : kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: verify-images
spec:
  rules:
  - name: check-key
    match:
      resources:
        kinds:
        - Pod
    imageVerify:
      default: deny
      digestMutate: true
      digestVerify: true
      images:
        - "registry.company.com/*"
      attestors:
      - entries:
        - staticKey:
            key: |-
            -----BEGIN PUBLIC KEY-----
            -----END PUBLIC KEY-----  

```

#### Example: require a keyless GitHub attestor

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: check-image-keyless
  annotations:
    pod-policies.kyverno.io/autogen-controllers: none
spec:
  validationFailureAction: enforce
  background: false
  webhookTimeoutSeconds: 30
  failurePolicy: Fail  
  rules:
    - name: check-image-keyless
      match:
        resources:
          kinds:
            - Pod
      verifyImages:
      - imageReferences:
        - "ghcr.io/demo/*"
        attestors:
        - entries:
          - keyless:
              subject: "https://github.com/demo/*"
              issuer: "https://token.actions.githubusercontent.com"
              additionalExtensions:
                githubWorkflowTrigger: push
                githubWorkflowSha: 73f9df28b6c67e3d4a4ffc4b75aaeed89be88b58
                githubWorkflowName: cosign
                githubWorkflowRepository: demo/workflows
```

#### Example: require all pod images are from a trusted registry and are signed by 1 of 3 keys

```yaml
apiVersion : kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: verify-images
spec:
  rules:
  - name: check-key
    match:
      resources:
        kinds:
        - Pod
    imageVerify:
      default: deny
      digestMutate: true
      digestVerify: true
      images:
        - "registry.company.com/*"
      attestors:
      - count: 1
        entries:
        - staticKey:
            key: |-
            -----BEGIN PUBLIC KEY-----
            -----END PUBLIC KEY-----  
        - staticKey:
            key: |-
            -----BEGIN PUBLIC KEY-----
            -----END PUBLIC KEY-----  
        - staticKey:
            key: |-
            -----BEGIN PUBLIC KEY-----
            -----END PUBLIC KEY-----  
```

#### Example: require all Tekton task images are from a trusted registry and are signed using keyless

```yaml
apiVersion : kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: verify-task-images
spec:
  rules:
  - name: check-key
    match:
      resources:
        kinds:
        - tekton.io/v1/Task
    imageVerify:
      imagePaths:
      - path: "{{ spec.tasks[*] }}"
      default: deny
      digestVerify: true
      images:
      - "registry.company.com/*"
      attestors:
      - match:
        entries:
          - keyless:
            subject: "https://github.com/JimBugwadia/demo-java-tomcat/.github/workflows/publish.yaml@refs/tags/*"
            issuer: "https://token.actions.githubusercontent.com"
            additionalExtensions:
              githubWorkflowTrigger: push
              githubWorkflowSha: 73f9df28b6c67e3d4a4ffc4b75aaeed89be88b58
              githubWorkflowName: build-sign-attest
              githubWorkflowRepository: JimBugwadia/demo-java-tomcat
```

#### Example: require all pod images are have a Trivy image scan attestation with keyless signing

```yaml
apiVersion : kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: verify-images
spec:
  rules:
  - name: check-vuln-scan
    match:
      resources:
        kinds:
        - Pod
    imageVerify:
      digestMutate: true
      digestVerify: true
      images:
      - "*"
      attestors:
      - match:
        entries:
        - staticKey:
            key: |-
            -----BEGIN PUBLIC KEY-----
            -----END PUBLIC KEY-----      
      attestions:
      - predicateType: https://trivy.aquasec.com/scan/v2
        conditions:
        - all:
          - key: "{{ scanner }}"
            operator: Equals
            value: trivy
          - key: "{{ time_since('', timestamp, '') }}"
            operator: LessThan
            value: 24h
          - key: "{{ Results[].Vulnerabilities[?CVSS.redhat.V3Score > `8.0`][] | length(@) }}"
            operator: Equals
            value: 0
```

### Limitations

#### Attestions with different attestors

The proposal allows declaring a list of attestors and attestions. This assumes that all attestations are provided by the same set of attestors. Although not very common, its possible that different attestations on the same image are verified by different attestors. For example, a static key is used to sign the SBOM but a keyless attestion is used to sign the vulnerability scan report.

With the current design, this can be handled using separate image verification declarations in the same policy rule, using separate rules, or using separate policies. Here are some additional thoughts for the future

* To reduce the number of OCI lookups, the signed paylods returned for a digest can be cached during webhook execution.
* The current design can be extended to allow `attestors` to be specified under an `attestion`. This would allow declaring common attestors, if any, and then attestors specific to attestions.
* An alternate scheme would be to allow an `attestor` to be named, and then referenced by name within an `attestion`.

We can revisit these topics based on usage patterns.

## Alternatives

Here are some of the prior proposals which were used to evolve the currently proposed solutions:

### Alternate Proposal 1

Enhance the `imageRegistry` context call with the following options -

- `verify`: An array of struct which contains the following - 
  - `image`: Image glob to verify against
  - `key` (optional): public key to use for verification. If not specified, keyless verification is assumed
  - `id`: A unique id to identify this key/image configuration.
  - `repository` (optional): same as COSIGN_REPOSITORY flag
- `fetchAttestations` (optional): boolean value to fetch attestations or not.
- `fetchConfig` (optional): boolean value to fetch the image config or not.
- `tagsAllowed` (optional): boolean value indicating if tags are allowed as input `references` or not. Defaults to `false`. In case the `reference` is a tag and `tagsAllowed` is set to false, the external data call should fail and the webhook should error out with an appropriate message. The policy owner may choose to create a mutate rule before hand that mutates all image reference tags to digest to ensure that this is always followed.

The image manifest is always fetched as that is needed to compute the image digest.

Based on the above options, the output context variable will be enhanced with the following fields - 

- `signatures`: A list of verified signatures along with their annotations in the `optional` field. In case of keyless verification, the `optional` field contains the cosign keyless bundle including fields like `subject` and `issuer`. This is to allow for dynamic verification of `subject` and `issuer` fields. They will additionally contain an `id` field that matches the `id` from the `verify` call.
- `attestations`: A list of decoded [attestations statements](https://github.com/in-toto/attestation/tree/main/spec#statement) attached to the image.


#### Mutate tags to digests


```yaml=
apiVersion : kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: resolve-images
spec:
  background: false
  rules:
  - name: resolve-tag-to-digest
    match:
      resources:
        kinds:
        - Pod
    preconditions:
      all:
      - key: "{{request.operation}}"
        operator: NotEquals
        value: DELETE
    mutate:
      foreach:
      - list: "request.object.spec.containers"
        context:
          - name: resolvedRef
            imageRegistry:
              reference: "{{ element.image }}"
              jmesPath: "resolvedImage"
        patchStrategicMerge:
          spec:
            containers:
            - name: "{{ element.name }}"           
              image: "{{ resolvedRef }}"
```

#### Verify image has valid signatures


```yaml=
apiVersion : kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: validate-images
spec:
  background: false
  rules:
  - name: validate-signatures
    match:
      resources:
        kinds:
        - Pod
    preconditions:
      all:
      - key: "{{request.operation}}"
        operator: NotEquals
        value: DELETE
    validate:
      message: "Images must be signed by a valid authority"
      foreach:
      - list: "request.object.spec.containers"
        context: 
        - name: imageData
          imageRegistry: 
            reference: "{{ element.image }}"
            allowTags: false
            verify:
              - image: "registry.io/org/*"
                id: registry-io-org
                key: |-
                  -----BEGIN PUBLIC KEY-----
                  MFkwEwYHKoZIzj0CAQYIKoZIzj0DAQcDQgAEHMmDjK65krAyDaGaeyWNzgvIu155
                  JI50B2vezCw8+3CVeE0lJTL5dbL3OP98Za0oAEBJcOxky8Riy/XcmfKZbw==
                  -----END PUBLIC KEY-----
              - image: "registry.io/*"
                id: registry-io
                key: |-
                  -----BEGIN PUBLIC KEY-----
                  MFkwEwYHKoZIzj0CAQYIKoZIzj0DAQcDQgAEHMmDjK65krAyDaGaeyWNzgvIu155
                  JI50B2vezCw8+3CVeE0lJTL5dbL3OP98Za0oAEBJcOxky8Riy/XcmfKZbw==
                  -----END PUBLIC KEY-----
              - image: "registry.io/*"
                id: registry-io-keyless
        deny:
          conditions:
            all:
              # Image must contain at least 1 verified signature
              # Since this array is populated with all valid signatures, 
              # we can add custom logic to implement `and`/ `or` conditions as well as noted below.
              - key: "{{ imageData.signatures }}"
                operator: GreaterThan
                value: 0
              # If the image is from registry.io/org, it must contain a valid signature from key registry-io-org
              - key: "{{ length(imageData.signatures[?id == `registry-io-org`]) > `0` }}"
                operator: Equals
                value: "{{ starts_with(element.image, `registry.io/org/`)}}"
              # All verified keyless signatures must have a subject that has an email from kyverno.io
              - key: "{{ length(imageData.signatures[?id == `registry-io-keyless`]) }}"
                operator: Equals
                value: "{{ length(imageData.signatures[?id == `registry-io-keyless`][?contains(subject, `@kyverno.io`)]) }}"
```
              
TODO: Add more examples.


### Alternate Proposal 2

Deprecate the existing `verifyImages` rule in favor of 2 new sub-rule types:
- `mutate.images` fetches image configuration data and replaces the image tag with a digest.
- `validate.images` validates image signatures, configuration data, and attestations.


```yaml=
apiVersion : kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: resolve-images
spec:
  background: false
  rules:
  - name: resolve-tag-to-digest
    match:
      resources:
        kinds:
        - Pod
    mutate:
      images:
      - image: *
        addDigest: {}
        foreach:
          ...
```


#### Verify image signatures


```yaml=
apiVersion : kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: validate-images
spec:
  rules:
  - name: validate-signatures
    match:
      resources:
        kinds:
        - Pod
    validate:
      message: "Images must be signed by a valid authority"
      images:
        - image: registry.io/org/*
          keys:
            any: [...]
            all: [...]
            min:
              count: 2
              entries: [...]
          keyless:
            url: https://fulcio.dev
            identities: 
              any:
              - subject: "..."
                issuer: "..."
          attestations:
            - predicateType: "..."
              conditions:
                any:
                all:
                min:
          deny:
             ...
             
          pattern:
            ...
            
          foreach:
            ...
 
```