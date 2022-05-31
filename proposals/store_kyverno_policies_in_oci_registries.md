# Kyverno Design Proposal - Store Kyverno Policies in OCI Registries

- Start Date: May 31, 2022
- Author(s): [Batuhan Apaydın](https://github.com/developer-guy), [Furkan Türkal](https://github.com/dentrax)

# Table of Contents

[table-of-contents]: #table-of-contents

- [Meta](#meta)
- [Table of Contents](#table-of-contents)
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

The concept behind OCI Artifacts was first published on an [OpenContainers official blog](https://opencontainers.org/posts/blog) with a post written by Steve Lasker ([@stevelasker](https://twitter.com/stevelasker).

    “The OCI Technical Oversight Board (TOB) has approved a new Artifacts project, utilizing the OCI manifest and OCI index definitions, new artifact types can be stored and served using the OCI distribution-spec without changing the actual distribution spec. This repository will provide a reference for artifact authors and registry implementers for supporting these new artifact types themselves with the existing implementations of distribution.”

The most helpful advantage of having an OCI Artifacts definition is that the community can continue to innovate, focusing on new artifact types without building another storage solution (YASS). More technically, OCI Artifacts are not a new specification, format, or API, but rather a set of conventions for storing assets other than images inside an OCI registry. For example, thanks to OCI Artifacts spec, additional artifacts, such as Helm Charts, can be easily stored and accessible from an OCI registry.

# Definitions

[definitions]: #definitions

Make a list of the definitions that may be useful for those reviewing. Include phrases and words that Kyverno users or other interested parties may not be familiar with.

# Motivation

[motivation]: #motivation

OCI Artifacts provides a [reference guide](https://github.com/opencontainers/artifacts/blob/main/artifact-authors.md) for authors looking to introduce a new artifact type. A unique artifact type is similar to defining a file extension. Defining a unique artifact allows various tools to work with the type uniquely. For example, it will enable a registry to display the type and tooling, such as vulnerability scanners, to know if and how they should interact with the contents. To learn more about defining your artifact type, please [see](https://github.com/opencontainers/artifacts/blob/main/artifact-authors.md#defining-a-unique-artifact-type).

     “application/vnd.[org/company].[objectType].[optional sub type].config.[version]+json”

# Proposal

> Issue [#3154](https://github.com/kyverno/kyverno/issues/3154)

Two implementation points exist in the Kyverno space that can be used to support pulling/pushing policies based on OCI Artifacts specification: Kyverno itself and its Command Line Interface (CL)I. First, we will start by introducing new subcommands for pulling/pushing policies using Kyverno CLI and then to Kyverno itself.

One of the initial steps that OCI artifact authors need to decide upon is the mediaTypes that we will use to store Kyverno policies. According to the OCI Artifact Authors specification format defined for new artifact types, for the Kyverno policy layer, which we use to keep the policy itself, it should be represented as `application/vnd.cncf.kyverno.policy.layer.v1+yaml` or `application/vnd.cncf.kyverno.policy.layer.v1+json`, or if we want to store more than one policy within the same policy layer we should bundle them up in `.tar.gz` file and upload them with a representation `application/vnd.cncf.kyverno.policy.layer.v1.tar+gzip`, and then, `application/vnd.cncf.kyverno.config.v1+json` which will be used to store the policy config.

The next step would be determining the library that we will use to interact with OCI Artifacts. The two well-known libraries that can help us achieve this are [go-containerregistry](https://github.com/google/go-containerregistry/) and [ORAS](https://oras.land). We can leverage either of them to pull and push Kyverno policies as OCI Artifacts.

To push a Kyverno policy to a registry, the command would be:

```shell
kyverno oci push img <yaml(s)>
```

To pull a policy from a registry, the command would be:

```shell
kyverno oci pull <img> –o <dir>
```


# Implementation

A demonstration has been prepared using the go-containerregistry illustrate this use case which can be found in the associated GitHub project. At the time of writing this, DockerHub does not support OCI Artifacts  (related issue). Alternative registry options are available which do support OCI Artifacts including quay.io, Harbor, gcr.io, AWS ECR, etc. Still, for the sake of simplicity, in this example, a registry leveraging the registry:2 image will be used.

So, let’s start by cloning the project.

```shell
$ git clone https://github.com/developer-guy/oci-kyverno
$ cd oci-kyverno
```

Next, run the registry.

$ docker run -d -p 5000:5000 --restart=always --name registry registry:2

Now, we are ready to store the policy on OCI Registry: disallow-capabilities.yaml

```shell
$ go run main.go disallow-capabilities.yaml localhost:5000/disallow-capabilities:latest
```

We have successfully uploaded our Kyverno policy to the registry we host. Next, inspect the manifest of that image. To do that, we’ll be using the command-line utility [crane](https://github.com/google/go-containerregistry/blob/master/cmd/crane).

```shell
$ crane manifest localhost:5000/disallow-capabilities:latest | jq 
```

As you can see from the output above, the mediaTypes are precisely the same as what we’ve defined for the Kyverno policy.
In the event that you want to inspect the contents of the layer, the blob command of the crane utility can be used.

```shell
$ crane blob localhost:5000/disallow-capabilities@sha256:5b6075facc39bd992695f2c44285ae78165cf1497539b49168da4698a16cbfe7 | yh
```

# Drawbacks

We should definitely do this :)

# Alternatives

- Loading Rego policies over OCI images to validate attestations: [sigstore/cosign#1478](https://github.com/sigstore/cosign/pull/1478)

- DockerHub is not supporting OCI artifacts right : [docker/roadmap#135](https://github.com/docker/roadmap/issues/135#issuecomment-921038044)

- Tekton is also working on a project called Bundles to store tekton tasks on OCI registries, [see](https://vinamra-jain.medium.com/referencing-tekton-tasks-from-oci-registry-5ff983d536e0).

- Confetst Rego push code: open-policy-agent/conftest@7d0099b/internal/commands/push.go#L48-L50

- https://kustomizer.dev

- https://helm.sh/blog/storing-charts-in-oci/


# Prior Art

Discuss prior art, both the good and bad.

# Unresolved Questions

- What parts of the design do you expect to be resolved before this gets merged?
- What parts of the design do you expect to be resolved through implementation of the feature?
- What related issues do you consider out of scope for this KDP that could be addressed in the future independently of the solution that comes out of this KDP?
