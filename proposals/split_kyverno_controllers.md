# Meta
[meta]: #meta
- Name: Split / Decompose Kyverno Controllers
- Start Date: November 9th, 2022
- Update data (optional): November 9th, 2022
- Author(s): @JimBugwadia, @eddycharlie

# Table of Contents
[table-of-contents]: #table-of-contents
- [Meta](#meta)
- [Table of Contents](#table-of-contents)
- [Overview](#overview)
- [Definitions](#definitions)
- [Motivation](#motivation)
- [Proposal](#proposal)
- [Implementation](#implementation)
- [Migration (OPTIONAL)](#migration-optional)
- [Drawbacks](#drawbacks)
- [Alternatives](#alternatives)
- [Prior Art](#prior-art)
- [Unresolved Questions](#unresolved-questions)
- [CRD Changes (OPTIONAL)](#crd-changes-optional)

# Overview
[overview]: #overview

Kyverno 1.8 runs as a single Deployment. This includes the webhook, reporting, and background controllers for generate and update requests. This design causes heavy backhround processing to impact the webhook performance. This proposal separates the webhook into its own Deployment.

# Definitions
[definitions]: #definitions

Make a list of the definitions that may be useful for those reviewing. Include phrases and words that Kyverno users or other interested parties may not be familiar with.

# Motivation
[motivation]: #motivation

- Eliminate, or greatly reduce, client-side throttling of the webhook.
- Improve webhook performance
- Allow independent sizing and scalability of webhook and background processing controllers

# Proposal

## Kyverno Controllers

### Webhook Controller

The webhook controller is responsible for:
1. Processing AdmissionReview requests from the API Server
2. Listening to policy and policyexception changes to maintain the policy cache

### Background Controller

The background controller is responsible for:
1. Processing report changes
2. Processing mutate and generate requests

**NOTE 1**: its possible to further decompose the background controller, but seems like that may be of incremental value. Even with a single background controller, we can enable / disable modules or features.

**NOTE 2**: we need to decide if the new cleanup controller is part of the background controller or should be separated.

## Code Repository Structure

All Kyverno code can stay in the same repository, and the top-level packages can reflect the logical design:

* **controllers/webhook**
* **controllers/background**

## User Experience

This section describes the user experience across installation and management of Kyverno:

### Installation

Kyverno can still be deployed as a single Helm chart or via a single YAML file. 

### Runtime 

When Kyverno is installed there would be two Deployment resources created. The Helm chart can allow each Deployment to be scaled independently. 

### Configuration

The Kyverno ConfigMap can allow enabling or disabling all major features or modules.

# Implementation

TBD.

## Link to the Implementation PR

# Migration (OPTIONAL)

When a new version of Kyverno, with this feature, is installed it will deploy two controllers. The `kyverno` deployment can continue to be the webhook, and a new `kyverno-background-controller` can execute to process the reports and update requests. 

# Drawbacks

This scheme adds some more complexity to the overall Kyverno installation.

# Alternatives

### Granular controllers

As mentioned above, it is possible to further decompose the background controller.

### Multiple Helm Charts

A more granular approach would also allow separate Helm charts to be installed. 

# Prior Art

## ArgoCD

When ArgoCD is installed it runs three different controllers.

# Unresolved Questions

TBD

# CRD Changes (OPTIONAL)

There are no CRD changes envisioned.